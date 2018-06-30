# Istio
## Intro into service meshes

Vadim Rutkovsky <vrutkovs@redhat.com>

---
#### So you've installed Kubernetes

All the cool kids do this these days,

but what about my app?

* How does my service map look like?
* Why is it taking so long to process a request?
* How many RPS does my service get?
* How many 404/5xx errors are my users seeing?
* How do I rollout the new version just for a fraction of users?

---
### Istio

An open platform to connect, manage, and secure microservices

* Developed by Google, IBM, and Lyft
* Sidecar pattern - every pod gets Envoy proxy, which additionally filters traffic
---
### Istio - advanced features

* Additional routing rules - e.g. cookie-based
* Ingress/Egress traffic rules
* Circuit breaking - drop slow requests sooner
* Traffic shifting - balance between services
* Mirroring - copy existing traffic to not-yet-available version of the app
* TLS inside the cluster for various levels of access between microservices

---
### Demo

CatCatGo - search engine for Reddit cat pictures.

* Database - MongoDB
* Backend - python aiohttp, responds to 'api/v1.0/search/kitten'
* Frontend - ReactJS app

---
### Gateway
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
### VirtualService
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

* Grafana + Prometheus stack to collect metrics
* Prepared dashboard to show amount of requests, success rate, rps and response size
* Jaeger UI - a tool to trace requests
---
### Demo - v2 canary
```diff
@ route-rules.yml:22 @ spec:
  - name: v1
    labels:
      version: v1
+  - name: v2
+    labels:
+      version: v2
```
---
### Demo - v2 canary
```diff
@ gateway.yml:37 @ spec:
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
### Poor man's DOS protection
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
### Siege results
```yaml
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
```
- destination:
    host: ui
    subset: v1
  mirror:
    host: ui
    subset: v2
```
---
# Questions?

Code: https://github.com/vrutkovs/catcatgo
