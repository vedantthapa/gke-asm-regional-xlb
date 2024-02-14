# gke-asm-regional-xlb

Demo for configuring IAP with GKE, Regional External Loadbalancer and Anthos Service Mesh.

## Background

This project is about abstracting the prior work on application level SSO by moving it in to the infrastructure layer. This is known to be achievable with an identity aware proxy solution, of which multiple will be assessed in various configurations and architectures.

At a minimum, all solutions being considered will provide a language and framework agnostic method for providing SSO to arbitrary applications deployed in our GCP tennant, regardless of language and framework. This will remove the development and maintenance burden of owning per-stack SSO libraries. Additionally, moving the authentication boundary outside of the application layer provides a strong security boundary, preventing unauthenticated traffic from ever reaching the deployed application, mitigating a significant portion of the risk individual applications otherwise hold. Further benefits may be achieved depending on the solution and architecture selected to move forward.

## Implementation

IAP requires an HTTPS LB whereas the default implementation of ingressgateway, i.e, K8s service type of `LoadBalancer` provisions a Network Passthrough LB which doesn't have that capability.

The alternative then is to change the service to `ClusterIP` and provision a HTTPS XLB in front of the service mesh gateway as described [in this architecture](https://cloud.google.com/architecture/exposing-service-mesh-apps-through-gke-ingress#cloud_ingress_and_mesh_ingress). However, note that the implementation uses a Global XLB managed by an `Ingress` object in K8s, which, is an issue for us due to compliance reasons. Moreover, it looks like the `Ingress` object only works with Global XLB.

We, therefore, resort to using the [Gateway API in GKE](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api) because it's `GatewayClass` resource has [support for a Regional XLB](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api#gatewayclass). See `xlb/` for full spec. Based on this, the current traffic flow is `Client --HTTPS --> GCP Gateway(Regional XLB) --HTTPS ---> Istio GW --HTTP + Istio mTLS ---> Sidecar --HTTP -->App`.

IAP is then configured via a `GCPBackendPolicy` attached to the ASM's ingressgateway service as documented [here](https://cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-resources#configure_iap).

### Setup

Provision Anthos Service Mesh as directed [here](https://cloud.google.com/service-mesh/docs/managed/provision-managed-anthos-service-mesh).

Provision a regional static external IP address with:
```sh
gcloud compute addresses create gateway-ip --project=${PROJECT_ID} --network-tier=STANDARD --region=northamerica-northeast1
```

Create certificates with:
```
# Create a root certificate and private key to sign the certificates for your services:
mkdir example_certs1
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -subj '/O=example Inc./CN=example.com' -keyout example_certs1/example.com.key -out example_certs1/example.com.crt

# Generate a certificate and a private key for httpbin.example.com:
openssl req -out example_certs1/httpbin.example.com.csr -newkey rsa:2048 -nodes -keyout example_certs1/httpbin.example.com.key -subj "/CN=httpbin.example.com/O=httpbin organization"
openssl x509 -req -sha256 -days 365 -CA example_certs1/example.com.crt -CAkey example_certs1/example.com.key -set_serial 0 -in example_certs1/httpbin.example.com.csr -out example_certs1/httpbin.example.com.crt

# Create secret for xlb gateway
kubectl create -n istio-system secret tls httpbin-credential \
  --key=example_certs1/httpbin.example.com.key \
  --cert=example_certs1/httpbin.example.com.crt

# Create secret for mesh gateway
kubectl create -n asm-ingress secret tls httpbin-credential \
  --key=example_certs1/httpbin.example.com.key \
  --cert=example_certs1/httpbin.example.com.crt
```

Apply the manifests with:
```sh
kustomize build app/ | kubectl apply -f -
kustomize build asm-ingressgateway/ | kubectl apply -f -
kustomize build xlb/ | kubectl apply -f -
```

Get the gateway address and port from the Gateway resource:
```
kubectl wait --for=condition=programmed gtw xlb-gateway -n istio-system
export INGRESS_HOST=$(kubectl get gtw xlb-gateway -n istio-system -o jsonpath='{.status.addresses[0].value}')
export SECURE_INGRESS_PORT=$(kubectl get gtw xlb-gateway -n istio-system -o jsonpath='{.spec.listeners[?(@.name=="https")].port}')
```

Wait for resources to be ready. Once done, run the following command to see the output:
```
curl -v -HHost:httpbin.example.com --resolve "httpbin.example.com:$SECURE_INGRESS_PORT:$INGRESS_HOST" \
  --cacert example_certs1/example.com.crt "https://httpbin.example.com:$SECURE_INGRESS_PORT/status/418"

```

## Alternatives

[GKE Gateway also has support for ASM](https://cloud.google.com/service-mesh/docs/managed/service-mesh-cloud-gateway#preview_limitations), however, there are limitations with that approach mainly due to the fact that it's still in experimentation, autopilot clusters are not supported and it uses a Global XLB.

Another option is to directly route traffic from client to the service, i.e, `Client --HTTPS --> GCP Gateway(Regional XLB) --HTTPS --> Sidecar --HTTP --> App`. In this case though we lose out on mTLS. Locking down the namespace to strict mTLS causes the sidecar to deny any requests coming from the GCP Gateway.

## References
https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/
https://cloud.google.com/architecture/exposing-service-mesh-apps-through-gke-ingress
https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api#gatewayclass
https://cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-resources
https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/
