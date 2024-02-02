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

> TODO: setup instructions

## Alternatives

[GKE Gateway also has support for ASM](https://cloud.google.com/service-mesh/docs/managed/service-mesh-cloud-gateway#preview_limitations), however, there are limitations with that approach mainly due to the fact that it's still in experimentation, autopilot clusters are not supported and it uses a Global XLB.

Another option is to directly route traffic from client to the service, i.e, `Client --HTTPS --> GCP Gateway(Regional XLB) --HTTPS --> Sidecar --HTTP --> App`. In this case though we lose out on mTLS. Locking down the namespace to strict mTLS causes the sidecar to deny any requests coming from the GCP Gateway.

## References
https://istio.io/latest/docs/tasks/traffic-management/ingress/secure-ingress/
https://cloud.google.com/architecture/exposing-service-mesh-apps-through-gke-ingress
https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api#gatewayclass
https://cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-resources
https://gateway-api.sigs.k8s.io/guides/http-redirect-rewrite/
