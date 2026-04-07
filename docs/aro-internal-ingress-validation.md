
## Validate internal ingress with a dedicated IngressController

This section validates the following traffic flow on ARO:

**Azure internal load balancer -> OpenShift router -> Route -> ClusterIP service -> Pod**

For this lab, the dedicated IngressController is `rhize`, configured with:

- internal LoadBalancer publishing
- domain `rhize.apps.dianasari-udr.westus.aroapp.io`
- route selector `ingress-scope=rhize`

The router service created for this ingresscontroller is `router-rhize`, with Azure internal load balancer IP `10.0.2.4`.

### Confirm the dedicated IngressController

Inspect the ingresscontroller and router service:

```bash
oc get ingresscontroller rhize -n openshift-ingress-operator -o yaml
oc get svc router-rhize -n openshift-ingress -o yaml
```

Confirm the following values:

* `spec.domain: rhize.apps.dianasari-udr.westus.aroapp.io`
* `spec.routeSelector.matchLabels.ingress-scope: rhize`
* service annotation `service.beta.kubernetes.io/azure-load-balancer-internal: "true"`
* load balancer IP `10.0.2.4`

### Create a test project

Create a test namespace:

```bash
oc new-project ingress-lab
```

### Deploy a simple HTTP application

Deploy an unprivileged nginx image and expose it as a ClusterIP service:

```bash
oc create deployment hello --image=nginxinc/nginx-unprivileged:stable-alpine -n ingress-lab
oc expose deployment hello --port=8080 --target-port=8080 -n ingress-lab
```

Verify the pod and service:

```bash
oc get pods -n ingress-lab
oc get svc -n ingress-lab
```

Expected result:

* the pod is `Running`
* the `hello` service is `ClusterIP`



### Create the route

Create a route from the service:

```bash
oc expose svc hello -n ingress-lab
```

Patch the route host so it matches the `rhize` ingress domain:

```bash
oc patch route hello -n ingress-lab --type=merge -p \
  '{"spec":{"host":"hello.rhize.apps.dianasari-udr.westus.aroapp.io"}}'
```

Label the route so it matches the `rhize` route selector:

```bash
oc label route hello -n ingress-lab ingress-scope=rhize --overwrite
```

Inspect the route:

```bash
oc describe route hello -n ingress-lab
```

Expected values:

* requested host: `hello.rhize.apps.dianasari-udr.westus.aroapp.io`
* label: `ingress-scope=rhize`
* backend endpoint on port `8080`

### Validate end-to-end HTTP connectivity through the Azure internal load balancer

Get the route host:

```bash
ROUTE_HOST=$(oc get route hello -n ingress-lab -o jsonpath='{.spec.host}')
echo $ROUTE_HOST
```

Test the route by sending traffic to the Azure internal load balancer IP while overriding DNS resolution locally:

```bash
curl -vk --resolve "${ROUTE_HOST}:80:10.0.2.4" "http://${ROUTE_HOST}"
```

Expected result:

* TCP connection to `10.0.2.4:80`
* HTTP `200 OK`
* nginx welcome page returned

This confirms the full path is working:

**10.0.2.4 -> router-rhize -> Route -> ClusterIP service -> nginx pod**



### Validate private DNS

Create a private DNS A record for:

* `hello.rhize.apps.dianasari-udr.westus.aroapp.io -> 10.0.2.4`

After the DNS zone and VNet link are in place, validate name resolution and access from an internal client:

```bash
nslookup hello.rhize.apps.dianasari-udr.westus.aroapp.io
curl -vk http://hello.rhize.apps.dianasari-udr.westus.aroapp.io
```

Expected result:

* DNS resolves the hostname to `10.0.2.4`
* HTTP access succeeds without using `curl --resolve`

### Enforce strict router isolation

By default, the `default` ingresscontroller admitted the same application host because it did not have any selector configured. To isolate the application strictly behind the dedicated `rhize` ingresscontroller, add namespace-based sharding.

Label the application namespace:

```bash
oc label namespace ingress-lab ingress-shard=rhize --overwrite
```

Patch `rhize` so it only admits routes from namespaces labeled `ingress-shard=rhize`:

```bash
oc patch ingresscontroller rhize -n openshift-ingress-operator --type=merge -p '
{
  "spec": {
    "namespaceSelector": {
      "matchLabels": {
        "ingress-shard": "rhize"
      }
    }
  }
}'
```

Patch `default` so it excludes namespaces labeled `ingress-shard=rhize`:

```bash
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge -p '
{
  "spec": {
    "namespaceSelector": {
      "matchExpressions": [
        {
          "key": "ingress-shard",
          "operator": "NotIn",
          "values": ["rhize"]
        }
      ]
    }
  }
}'
```

Wait for the ingresscontrollers and router pods to reconcile:

```bash
oc get ingresscontroller -n openshift-ingress-operator
oc get pods -n openshift-ingress -o wide
```

Re-check the route:

```bash
oc describe route hello -n ingress-lab
```

Expected result:

* the route is exposed only on `router rhize`

Validate both internal load balancer IPs directly with the same host header:

```bash
ROUTE_HOST=$(oc get route hello -n ingress-lab -o jsonpath='{.spec.host}')

curl -vk -H "Host: ${ROUTE_HOST}" http://10.0.2.4
curl -vk -H "Host: ${ROUTE_HOST}" http://10.0.3.254
```

Expected result:

* `10.0.2.4` returns `HTTP/1.1 200 OK`
* `10.0.3.254` returns `503 Application is not available`

This confirms the application host is isolated behind the dedicated `rhize` internal load balancer and is no longer served by the default ingresscontroller. 

### Validate edge TLS

After HTTP validation and strict router isolation are working, validate TLS termination at the OpenShift route.

Delete the existing unsecured route:

```bash
oc delete route hello -n ingress-lab
```

Create an edge-terminated Route manifest such as:

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: hello
  namespace: ingress-lab
  labels:
    ingress-scope: rhize
spec:
  host: hello.rhize.apps.dianasari-udr.westus.aroapp.io
  to:
    kind: Service
    name: hello
    weight: 100
  port:
    targetPort: 8080
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

Apply it:

```bash
oc apply -f hello-edge-route.yaml
```

Inspect the route:

```bash
oc describe route hello -n ingress-lab
oc get route hello -n ingress-lab -o yaml
```

Expected result:

* requested host: `hello.rhize.apps.dianasari-udr.westus.aroapp.io`
* `TLS Termination: edge`
* exposed only on `router rhize`

Validate HTTPS directly against the `rhize` internal load balancer:

```bash
ROUTE_HOST=$(oc get route hello -n ingress-lab -o jsonpath='{.spec.host}')
echo $ROUTE_HOST

curl -vk --resolve "${ROUTE_HOST}:443:10.0.2.4" "https://${ROUTE_HOST}"
```

Expected result:

* TLS handshake succeeds
* the certificate subject matches `*.rhize.apps.dianasari-udr.westus.aroapp.io`
* HTTP `200 OK` is returned

Validate both internal load balancer IPs with HTTPS:

```bash
curl -vk --resolve "${ROUTE_HOST}:443:10.0.2.4" "https://${ROUTE_HOST}"
curl -vk --resolve "${ROUTE_HOST}:443:10.0.3.254" "https://${ROUTE_HOST}"
```

Expected result:

* `10.0.2.4` returns `HTTP/1.1 200 OK`
* `10.0.3.254` returns `503 Application is not available`

This confirms the HTTPS flow is working through the dedicated `rhize` ingresscontroller only:

**10.0.2.4:443 -> router-rhize terminates TLS (edge) -> HTTP to service -> pod**

In this validation, the `rhize` router presented a self-signed wildcard certificate for `*.rhize.apps.dianasari-udr.westus.aroapp.io`, while the default router presented its own wildcard certificate for `*.apps.dianasari-udr.westus.aroapp.io` and returned `503` for the isolated application host.  

### Notes

* No manual Azure Load Balancer creation was required for this validation. The ARO/OpenShift IngressController created and managed the Azure internal load balancer automatically for `router-rhize`.
* Plain HTTP is the simplest initial validation path because it removes TLS variables and isolates ingress behavior.
* Private DNS validation is useful after the initial `curl --resolve` check so the application can be reached normally from internal clients.
* Namespace-based sharding is required if the goal is strict isolation behind a dedicated ingresscontroller. Without it, the default ingresscontroller may still admit the same application host.
* Edge TLS validation proves that the router can terminate HTTPS while the backend application remains plain HTTP.

### Troubleshooting

#### Route host is generated under the default apps domain

If the route is created with a generated host under the default apps domain, patch the route host so it matches the dedicated ingresscontroller domain:

```bash
oc patch route hello -n ingress-lab --type=merge -p \
  '{"spec":{"host":"hello.rhize.apps.dianasari-udr.westus.aroapp.io"}}'
```

#### Curl returns `Empty reply from server`

This can happen if the request is sent to the `rhize` load balancer IP but the route host does not match the `rhize` ingress domain. Confirm that:

* the route host is under `rhize.apps.dianasari-udr.westus.aroapp.io`
* the route is labeled `ingress-scope=rhize`

#### Route is exposed on both `default` and `rhize`

Inspect both ingresscontrollers:

```bash
oc get ingresscontroller default -n openshift-ingress-operator -o yaml
oc get ingresscontroller rhize -n openshift-ingress-operator -o yaml
```

If `default` has no selector, it may still admit the same application host. Apply namespace-based sharding so the application namespace is excluded from `default` and admitted only by `rhize`:

```bash
oc label namespace ingress-lab ingress-shard=rhize --overwrite

oc patch ingresscontroller rhize -n openshift-ingress-operator --type=merge -p '
{
  "spec": {
    "namespaceSelector": {
      "matchLabels": {
        "ingress-shard": "rhize"
      }
    }
  }
}'

oc patch ingresscontroller default -n openshift-ingress-operator --type=merge -p '
{
  "spec": {
    "namespaceSelector": {
      "matchExpressions": [
        {
          "key": "ingress-shard",
          "operator": "NotIn",
          "values": ["rhize"]
        }
      ]
    }
  }
}'
```

#### HTTPS certificate is self-signed

For this lab, the `rhize` router presented its default wildcard certificate for `*.rhize.apps.dianasari-udr.westus.aroapp.io`, which was self-signed. For initial validation, `curl -k` is expected and acceptable.

#### Test application is not healthy

Check the pod state and logs:

```bash
oc get pods -n ingress-lab
oc describe pod -n ingress-lab <pod-name>
oc logs -n ingress-lab <pod-name>
```
