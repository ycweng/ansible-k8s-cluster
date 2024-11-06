# install istio
```
helm install istio-base istio/base -n istio-system --set defaultRevision=default --create-namespace

$ helm ls -n istio-system
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
istio-base      istio-system    1               2024-10-24 15:23:25.50959346 +0800 CST  deployed        base-1.23.2     1.23.2 

helm get all istio-base -n istio-system


helm install istiod istio/istiod -n istio-system --wait

# 安裝gateway
helm install istio-ingressgateway istio/gateway -n istio-system
```

# istio app
```
kubectl create namespace cps-demo
```
### gateway
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: demo-gateway
  namespace: cps-demo
spec:
  selector:
    app: istio-ingressgateway
  servers:
    - hosts:
        - '*'
      port:
        name: http
        number: 80
        protocol: HTTP
```
### virtual service
- 寫path rule的地方
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: report-route
spec:
  gateways:
    - istio-system/istio-gateway
  hosts:
    - 10.110.54.201
  http:
    - match:
        - uri:
            regex: /demo/.*
      route:
        - destination:
            host: demo-app-svc
            port:
              number: 8080
```
### namespace
要開註解
```yaml
kind: Namespace
apiVersion: v1
metadata:
  name: cps-demo
  labels:
    istio-injection: enabled
```

## Auth
- request filter
- 會作用到整個namespace, nodeport直接開的服務也會受影響
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-all
  namespace: recommendation
spec:
  rules:
    - to:
        - operation:
            ports:
              - '8080'
    - from:
        - source:
            requestPrincipals:
              - '*'
      to:
        - operation:
            paths:
              - /recommendation/api/eges-search/*
            ports:
              - '5000'
```

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: cps-auth
  namespace: recommendation
spec:
  jwtRules:
    - forwardOriginalToken: true
      issuer: http://192.168.151.190:31080/keycloak/auth/realms/CPS
      jwksUri: >-
        http://192.168.151.190:31080/keycloak/auth/realms/CPS/protocol/openid-connect/certs
```