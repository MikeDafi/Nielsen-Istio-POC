# Nielsen-Istio-POC 

### Setup
With the provided istio download folder, you can skip installing isto but you will need istoctl. 
Refer to the [Getting Started](https://istio.io/latest/docs/setup/getting-started/) page to prepare your BookInfo Application.

### Notice
For each of these examples you will need to ```kubectl apply -f``` then ```kubectl delete -f``` as they will conflict with each other

## Istio Authentication

To invoke TLS in a cluster, we have multiple options in Istio. We can assign mTLS
to our mesh or each individual service in the mesh. One quick way to apply
mTLS encryption is with ```PeerAuthentication```.

### Apply mTLS on a service
In the code below, we select the frontend service with label app=frontend and
require mtls for any inbound traffic.
```
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "productpage"
  namespace: "default"
spec:
  selector:
    matchLabels:
      app: frontend
  mtls:
    mode: STRICT
```

### Apply mTLS on the mesh
The same is done on the mesh. Any inbound traffic to any service on the cluster
must require https protocol.
```
apiVersion: "security.istio.io/v1beta1"
kind: "PeerAuthentication"
metadata:
  name: "default"
  namespace: "default"
spec:
  mtls:
    mode: STRICT
```

### Apply JWT Authentication
Instead of purely having a tls protocol, we can extend this security to relying
on jwt authentication(AWS IAM Roles can be configured). With ```RequestAuthentication```, we will add jwtRules:
```
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
 name: productpage
 namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  jwtRules:
  - issuer: "testing@secure.istio.io"
    jwksUri: "https://raw.githubusercontent.com/istio/istio/release-1.5/security/tools/jwt/samples/jwks.json"
```
**You will need a token** to authenticate with the jwt public key
```
TOKEN=$(curl -k https://raw.githubusercontent.com/istio/istio/release-1.4/security/tools/jwt/samples/demo.jwt -s); echo $TOKEN
```
As an example, you can run the BookInfo application with this command and it will return 200:
```
kubectl exec $(kubectl get pod -l app=details -o jsonpath={.items..metadata.name}) -c istio-proxy \
-- curl  http://productpage:9080/ -o /dev/null --header "Authorization: Bearer $TOKEN" -s -w '%{http_code}\n'
```

But this Authorization header is optional so this would also work:
```
kubectl exec $(kubectl get pod -l app=details -o jsonpath={.items..metadata.name}) -c istio-proxy \
-- curl  http://productpage:9080/ -o /dev/null -s -w '%{http_code}\n'
```
**Notice** you will need to also apply a ```AuthorizationPolicy``` to make this header unoptional:
```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-jwt
  namespace: default
spec:
  selector:
    matchLabels:
      app: productpage
  action: ALLOW
  rules:
  - from:
    - source:
       requestPrincipals: ["testing@secure.istio.io/testing@secure.istio.io"]
```
Now running the same two commands should produce a ```200``` and ```403 - Forbidden```
response.

