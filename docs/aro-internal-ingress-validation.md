
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

### Validate end-to-end connectivity through the Azure internal load balancer

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

### Optional host header test

As an alternative test, send traffic directly to the Azure internal load balancer IP with the route hostname in the `Host` header:

```bash
curl -vk -H "Host: ${ROUTE_HOST}" http://10.0.2.4
```

### Notes

* No manual Azure Load Balancer creation was required for this validation. The ARO/OpenShift IngressController created and managed the Azure internal load balancer automatically for `router-rhize`. 
* Plain HTTP is the simplest initial validation path because it removes TLS variables and isolates ingress behavior.
* After HTTP validation succeeds, the next logical steps are:

  * create private DNS for `hello.rhize.apps.dianasari-udr.westus.aroapp.io` pointing to `10.0.2.4`
  * validate access from an internal client without `curl --resolve`
  * test TLS termination, depending on whether TLS will terminate at Azure or at the OpenShift route

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

#### Test application is not healthy

Check the pod state and logs:

```bash
oc get pods -n ingress-lab
oc describe pod -n ingress-lab <pod-name>
oc logs -n ingress-lab <pod-name>
```
