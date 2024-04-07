---
title: "Comparing Kubernetes Gateway and Ingress APIs"
date: 2022-10-14T09:12:38+05:30
draft: false
weight: 20
ShowToc: false
summary: "Exploring the new Kubernetes Gateway API and comparing it with the existing Kubernetes Ingress API for handling external traffic."
tags: ["kubernetes", "ingress", "api gateway"]
categories: ["Featured", "Kubernetes"]
cover:
    image: "/images/gateway-vs-ingress-api/banner-comparing-apples.jpeg"
    alt: "Photo of a person comparing peaches in a food market."
    caption: "Photo by [Michael Burrows](https://www.pexels.com/photo/crop-faceless-male-choosing-peaches-in-food-market-7129157/)"
    relative: false
---

A couple of months ago, the new [Kubernetes Gateway API graduated to beta](https://kubernetes.io/blog/2022/07/13/gateway-api-graduates-to-beta/).

Why do you need another API to handle external traffic when you have the stable [Kubernetes Ingress API](https://kubernetes.io/docs/concepts/services-networking/ingress/#alternatives) and [dozens of implementations](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/#additional-controllers)? What problems of the Ingress API does the new Gateway API solve? Does this mean the end of the Ingress API?

I will try to answer these questions in this article by getting hands-on with these APIs and looking at how they evolved.

## Standardizing External Access to Services: The Ingress API

The Kubernetes Ingress API was created to standardize exposing services in Kubernetes to external traffic. The Ingress API overcame the limitations of the default service types, `NodePort` and `LoadBalancer`, by introducing features like routing and SSL termination.

{{< figure src="/images/gateway-vs-ingress-api/kubernetes-ingress.png#center" title="Kubernetes Ingress" link="/images/gateway-vs-ingress-api/kubernetes-ingress.png" target="_blank" class="align-center" >}}

There are over [20 implementations of Ingress controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/#additional-controllers) available. In this article, I will use [Apache APISIX](https://apisix.apache.org/) and its [Ingress controller](https://apisix.apache.org/docs/ingress-controller/next/getting-started/) for examples.

{{< figure src="/images/gateway-vs-ingress-api/apisix-ingress-controller.png#center" title="APISIX Ingress" link="/images/gateway-vs-ingress-api/apisix-ingress-controller.png" target="_blank" class="align-center" >}}

You can create an [Ingress resource](https://kubernetes.io/docs/concepts/services-networking/ingress/#the-ingress-resource) to configure APISIX or any other Ingress implementations.

The example below shows how you can route traffic between two versions of an application with APISIX Ingress:

```yaml {title="kubernetes-ingress-manifest.yaml"}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-routes
spec:
  ingressClassName: apisix
  rules:
    - host: local.navendu.me
      http:
        paths:
          - backend:
              service:
                name: bare-minimum-api-v1
                port:
                  number: 8080
            path: /v1
            pathType: Prefix
          - backend:
              service:
                name: bare-minimum-api-v2
                port:
                  number: 8080
            path: /v2
            pathType: Prefix
```

> **Tip**: You can check out this [hands-on tutorial](/posts/hands-on-set-up-ingress-on-kubernetes-with-apache-apisix-ingress-controller/) to learn more about setting up Ingress on Kubernetes with Apache APISIX Ingress controller.

Since the Ingress API is not tied to any particular controller implementation, you can swap APISIX with any other Ingress controller, and it will work similarly.

This is okay for simple routing. But the API is limited, and if you want to use the full features provided by your Ingress controller, you are stuck with [annotations](https://apisix.apache.org/docs/ingress-controller/concepts/annotations/).

For example, the Kubernetes Ingress API does not provide a schema to configure rewrites. Rewrites are useful when your upstream/backend URL differs from the path configured in your Ingress rule.

APISIX supports this feature, and you have to use custom annotations to leverage it:

```yaml {title="kubernetes-ingress-manifest.yaml"}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-routes
  annotations:
    k8s.apisix.apache.org/rewrite-target-regex: "/app/(.*)"
    k8s.apisix.apache.org/rewrite-target-regex-template: "/$1"
spec:
  ingressClassName: apisix
  rules:
    - host: local.navendu.me
      http:
        paths:
          - backend:
              service:
                name: bare-minimum-api
                port:
                  number: 8080
            path: /app
            pathType: Prefix
```

This creates an Ingress resource that configures APISIX to route any requests with the `/app` prefix to the backend with the prefix removed. For example, a request to `/app/version` will be forwarded to `/version`.

Annotations are specific to your choice of an Ingress controller. These "proprietary" extensions limited the scope of portability intended initially with the Ingress API.

## Custom CRDs > Ingress API

Being stuck with annotations also sacrifice the usability of the Ingress controllers.

Controllers therefore solved the limitations of the Ingress API by creating their [own custom resources](https://apisix.apache.org/docs/ingress-controller/references/apisix_route_v2/). The example below shows configuring Ingress to route traffic between two versions of an application using APISIX's custom resource:

```yaml {title="apisix-ingress-manifest.yaml"}
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: api-routes
spec:
  http:
    - name: route-1
      match:
        hosts:
          - local.navendu.me
        paths:
          - /v1
      backends:
        - serviceName: bare-minimum-api-v1
          servicePort: 8080
    - name: route-2
      match:
        hosts:
          - local.navendu.me
        paths:
          - /v2
      backends:
        - serviceName: bare-minimum-api-v2
          servicePort: 8080
```

These CRDs made it much easier to configure Ingress, but you are tied to the specific Ingress control implementation. Without the Ingress API evolving, you had to choose between usability or portability.

## Extending Ingress and Evolution to Gateway API

Ingress API was not broken; it was limited. The Gateway API was designed to overcome these limitations.

{{< blockquote author="gateway-api.sigs.k8s.io" link="https://gateway-api.sigs.k8s.io/" title="What is the Gateway API?" >}}
(Gateway API) aim to evolve Kubernetes service networking through expressive, extensible, and role-oriented interfaces ...
{{< /blockquote >}}

It takes inspiration from the custom CRDs of different Ingress controllers mentioned earlier.

The Gateway API adds [many features](https://gateway-api.sigs.k8s.io/#gateway-api-concepts) "on top" of the Ingress API's capabilities. This includes HTTP header-based matching, weighted traffic splitting, and other features that require custom proprietary annotations with the Ingress API.

Traffic split with APISIX Ingress resource (see [ApisixRoute/v2 reference](https://apisix.apache.org/docs/ingress-controller/references/apisix_route_v2/)):

```yaml {title="apisix-ingress-manifest.yaml"}
apiVersion: apisix.apache.org/v2
kind: ApisixRoute
metadata:
  name: traffic-split
spec:
  http:
    - name: rule-1
      match:
        hosts:
          - local.navendu.me
        paths:
          - /get*
      backends:
        - serviceName: bare-minimum-api-v1
          servicePort: 8080
          weight: 90
        - serviceName: bare-minimum-api-v2
          servicePort: 8080
          weight: 10
```

Traffic split with Gateway API (see [Canary traffic rollout](https://gateway-api.sigs.k8s.io/guides/traffic-splitting/#canary-traffic-rollout)):

```yaml {title="kubernetes-gateway-manifest.yaml"}
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: HTTPRoute
metadata:
  name: traffic-split
spec:
  hostnames:
  - local.navendu.me
  rules:
  - backendRefs:
    - name: bare-minimum-api-v1
      port: 8080
      weight: 90
    - name: bare-minimum-api-v2
      port: 8080
      weight: 10
```

Another improvement from the Ingress API is how the Gateway API [separates concerns](https://gateway-api.sigs.k8s.io/concepts/security-model/#roles-and-personas). With Ingress, the application developer and the cluster operator work on the same Ingress object, unaware of the other's responsibilities and opening the door for misconfigurations.

The Gateway API separates the configurations into Route and Gateway objects providing autonomy for the application developer and the cluster operator. The diagram below explains this clearly:

{{< figure src="/images/gateway-vs-ingress-api/gateway-api.png#center" title="The Gateway API" caption="Adapted from [gateway-api.sigs.k8s.io](https://gateway-api.sigs.k8s.io/)" link="/images/gateway-vs-ingress-api/gateway-api.png" target="_blank" class="align-center" >}}

## Is This the End of Ingress API?

The Gateway API is relatively new, and its implementations are constantly breaking. On the contrary, the Ingress API is in stable release and has stood the test of time.

If your use case only involves simple routing and if you are okay with using custom annotations to get extra features, the Ingress API is still a solid choice.

With the Gateway API being a superset of the Ingress API, it might make sense to consolidate both. Thanks to the [SIG Network](https://github.com/kubernetes/community/tree/master/sig-network) community, Gateway API is still growing and will soon be production ready.

Most Ingress controllers and [service meshes](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/) have already implemented the Gateway API along with the Ingress API, and as the project evolves, more implementations will surface.

Personally, at least for now, I would stick with [custom CRDs](https://apisix.apache.org/docs/ingress-controller/next/getting-started/) provided by the Ingress controllers like Apache APISIX instead of the Ingress or Gateway API.
