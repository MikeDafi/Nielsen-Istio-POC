# Nielsen-Istio-POC 

### Why Istio? 
Within the microservice architecture, each service needs to include code that accounts for its own business logic along with networking logic (communication configuration, security logic, retry logic, telemetry). As an application grows and more pods are added into a Kubernetes cluster, each microservice still needs to account for both types of logic, ultimately leading to a more complicated, bulkier application. 

Istio helps with this as it takes care of the networking logic, leaving each microservice to focus on its own fucntionality. The beauty in Istio lies in the fact that our Kubernetes deployment and service files don't need to be modified. All of the configuration is done within Istio itself.

### Setup
With the provided istio download folder, you can skip installing istio but you will need istioctl. 
Refer to the [Getting Started](https://istio.io/latest/docs/setup/getting-started/) page to prepare your BookInfo Application.

Note: within that setup, there were issues encountered with the ```minikube tunnel``` command. To determine the correct port for your web page, run:
```minikube service -n istio-system istio-ingressgateway```.
The second tab that opens contains the correct port. Append ```/productpage``` to the end of this URL.

### Notice
For each of these examples you will need to ```kubectl apply -f``` then ```kubectl delete -f``` as they will conflict with each other

## Istio Authentication

To invoke TLS in a cluster, we have multiple options in Istio. We can assign mTLS
to our mesh or each individual service in the mesh. One quick way to apply
mTLS encryption is with ```PeerAuthentication```.

### Apply mTLS on a service
In the code below, we select the productpage service with label app=productpage and name="productpage" and
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
      app: productpage
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

### Telemetry
Istio integrates with several different telementry applications: cert-manager, Jaeger, Prometheus, Grafana, Kiali, Zipkin.

To install the addons into your cluster, perform the following command from the root of your Istio installation:
```
$ kubectl apply -f samples/addons
```

Note: in order to view trace data, traffic must be sent to the Bookinfo product page. You can do this with:
```
$ for i in $(seq 1 100); do curl -s -o /dev/null "http://$GATEWAY_URL/productpage"; done
```

#### Accessing the dashboards
One can access the Istio dashboard via any of the addons through the following:
```
$ istioctl dashboard <addon name>
```

