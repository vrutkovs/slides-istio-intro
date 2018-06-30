# Istio
## Intro into service mesh

Vadim Rutkovsky <vrutkovs@redhat.com>

---
### So you've installed Kubernetes

All the cool kids do this these days,

but what about my app?

* How does my service map look like?
* Why is it taking so long to process a request?
* How many RPS does my service get?
* How many 404/5xx errors are my users seeing?
* How do I rollout the new version just for a fraction of users?

---
### Istio

An open platform to connect, manage, and secure microservices.
https://istio.sh

* Developed by Google, IBM, and Lyft
* **Sidecar pattern** - every pod gets Envoy proxy, which additionally filters traffic
* Controlled by CRD

---
### Istio - advanced features

* Flexible routing rules - e.g. cookie-based
* Ingress/Egress traffic rules
* **Circuit breaking** - drop slow requests sooner
* **Traffic shifting** - split traffic between services
* **Mirroring** - copy existing traffic to not-yet-available version of the app
* **TLS between pods** - in-cluster security

---
#### Istio components
* **Pilot** - controls Envoy configs - routing, service discovery
* **Mixer** - enforces policies and collects stats from Envoy
* **Citadel** - provides service-to-service authentication

Additional services:
* **Zipkin** - tracing requests
* **Prometheus+Grafana** - metrics
* **ServiceGraph** - render service status as a graph

---
### Getting Started
Fetch `istioctl` binary and config yamls
```shell
$ curl -L https://git.io/getLatestIstio | sh -
$ cd istio-0.8.0
$ export PATH=$PWD/bin:$PATH
```

Setup Istio with service-to-service auth:
```
$ kubectl apply -f install/kubernetes/istio-demo-auth.yaml
```
OR without auth:
```
$ kubectl apply -f install/kubernetes/istio-demo.yaml
```

---
### Demo

**CatCatGo** - search engine for Reddit cat pictures.

* Database - MongoDB
* Backend - python aiohttp

  responds to 'api/v1.0/search/kitten'
* Frontend - ReactJS app
* (hidden) CronJob to scrape fresh cat pictures from several subreddits and fill DB in

https://catcatgo.cloud.vrutkovs.eu

---
### Gateway
`Gateway` - Istio custom resource which manages ingress

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: catacatgo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```
---
### DestinationRule
`DestinationRule` - Istio CR to select pods based on app/version/etc.

```yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: backend
spec:
  host: backend
  subsets:
  - name: v1
    labels:
      version: v1
```
---
### VirtualService part 1
`VirtualService` - routing rules for services
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: catcatgo
spec:
  hosts:
  - "*"
  gateways:
  - catacatgo-gateway
...
```
---
### VirtualService part 2
```
...
  http:
  - match:
    - uri:
        exact: /
    - uri:
        exact: /index.html
    - uri:
        exact: /app.jsx
    route:
    - destination:
        host: ui
        subset: v1
...
```
---
### VirtualService part 3
```
...
  - match:
    - uri:
        prefix: /api/v1.0/search/
    route:
    - destination:
        host: backend
        subset: v1
```
---
### Metrics and Tracing

Grafana + Prometheus stack to collect metrics

Dashboard to show amount of requests, success rate, rps and response size
---
### Tracing requests
Jaeger UI - a tool to trace requests
---
### Demo - v2 canary
```yaml
  - name: v1
    labels:
      version: v1
+  - name: v2
+    labels:
+      version: v2
```
---
### Demo - v2 canary
```yaml
    - destination:
        host: ui
        subset: v1
+      weigth: 50
+    - destination:
+        host: ui
+        subset: v2
+      weigth: 50
  - match:
    - uri:
        prefix: /api/v1.0/search/
```
---
### Demo - v2 canary
Based on user-agent:
```yaml
- match:
  - headers:
      user-agent:
        regex: ".*62.*"
  route:
```
---
### Testing - fault injection
Throw server errors a lot
```yaml
- fault:
    abort:
      httpStatus: 500
      percent: 75
```
Delay the requests
```yaml
- fault:
    delay:
      fixedDelay: 3s
      percent: 50
```
---
### Limit incoming requests - config
```yaml
spec:
  host: backend
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 2
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
    outlierDetection:
      http:
        consecutiveErrors: 1
        interval: 1s
        baseEjectionTime: 1m
        maxEjectionPercent: 100
```
---
### Limit incoming requests - results
```
$ sudo podman run --rm -ti ecliptik/docker-siege -b -c 5 https://catcatgo.cloud.vrutkovs.eu/api/v1.0/search/test
New configuration template added to /root/.siege
Run siege -C to view the current settings in that file
** SIEGE 4.0.2
** Preparing 5 concurrent users for battle.
The server is now under siege...
HTTP/1.1 503     0.82 secs:      57 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 503     0.82 secs:      57 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 503     0.82 secs:      57 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.82 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.82 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.55 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.55 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 503     0.57 secs:      57 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.58 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.60 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.53 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.53 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.55 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.60 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.58 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.55 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.56 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 200     0.54 secs:       3 bytes ==> GET  /api/v1.0/search/test
HTTP/1.1 503     0.51 secs:      57 bytes ==> GET  /api/v1.0/search/test
```
---
### Mirroring

Copy real traffic to v2 version
```yaml
- destination:
    host: ui
    subset: v1
  mirror:
    host: ui
    subset: v2
```

Check `istio-proxy` container logs to see the real traffic results

---
# Questions?

Slides: https://vrutkovs.github.io/slides-istio-intro
Code: https://github.com/vrutkovs/catcatgo

*<!-- -->* vrutkovs  <!-- .element: class="fab fa-github-square" -->

*<!-- -->* vrutkovs  <!-- .element: class="fab fa-twitter-square" -->

*<!-- -->* vrutkovs@redhat.com  <!-- .element: class="fas fa-envelope-square" -->
