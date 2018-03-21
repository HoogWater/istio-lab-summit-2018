# Basic Routing

There are three different super simple microservices in this lab and they are chained together in the following sequence:

`customer -> preference -> recommendation`

In this lab you'll dynamically alter routing between the services using Istio intelligent routing.

## What you will learn

* How to deploy microservice applications to OpenShift
* How to inject sidecar proxies into applications which form a service mesh
* How to use the service mesh configuration to dynamically route and shape traffic to and from services

## Step 1: Login and setup services

First, Login to OpenShift using the `oc` CLI:

```bash
oc login -u system:admin
```

Confirm that Istio is deployed and working:

```bash
oc get pods -n istio-system
```

You should see:
```console
NAME                             READY     STATUS    RESTARTS   AGE
grafana-3617079618-4qs2b         1/1       Running   0          4m
istio-ca-1363003450-tfnjp        1/1       Running   0          4m
istio-ingress-1005666339-vrjln   1/1       Running   0          4m
istio-mixer-465004155-zn78n      3/3       Running   0          5m
istio-pilot-1861292947-25hnm     2/2       Running   0          4m
jaeger-210917857-2w24f           1/1       Running   0          4m
prometheus-168775884-dr5dm       1/1       Running   0          4m
servicegraph-1100735962-tdh78    1/1       Running   0          4m
```

Let's set a few environment variables to use later on and add the istio CLI tools
to our `$PATH`:

```bash
export ISTIO_HOME=/opt/lab/istio-0.6.0
export PATH=${PATH}:${ISTIO_HOME}/bin
export LAB_HOME=/opt/lab/istio-lab-summit-2018
export LAB_PROJECT=istio-lab
```

## Step 2: Create Project

Create a new project to house the services used in this lab, and add
the necessary permissions to allow the Istio containers to do privileged things:

```bash
oc new-project $LAB_PROJECT
oc adm policy add-scc-to-user privileged -z default -n $LAB_PROJECT
```

We'll use the `istioctl` command line to create and install Istio components to OpenShift.

## Step 3: Deploy Customer service

Let's deploy the customer pod with its sidecar. We'll use the `kube-inject` CLI to manually
inject sidecar containers to our Customer deployment:

```bash
oc apply -f <(istioctl kube-inject -f ${LAB_HOME}/src/customer/src/main/kubernetes/Deployment.yml) -n ${LAB_PROJECT}
oc create -f ${LAB_HOME}/src/customer/src/main/kubernetes/Service.yml -n ${LAB_PROJECT}
```

Since customer is the forward-most microservice (`customer -> preference -> recommendation`),
let's add an OpenShift Route that exposes that endpoint:

```bash
oc expose svc/customer
```

And wait for it to completely roll out and receive a `deployment xxxxxx successfully rolled out` result:


```bash
oc rollout status -w deployment/customer
```

Now, test the service, which should fail:

```bash
curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
```

> The use of `oc get route` above simply returns the public hostname of the route that we can use
to access the customer endpoint from outside of OpenShift.

You should see `customer => I/O error on GET request for "http://preference:8080": preference; nested exception is java.net.UnknownHostException: preference`.
This is because the _preferences_ service is not yet deployed.

## Step 4: Deploy Preferences service

Let's deploy the preferences pod with its sidecar. We'll again use the `kube-inject` CLI:


```bash
oc apply -f <(istioctl kube-inject -f ${LAB_HOME}/src/preference/src/main/kubernetes/Deployment.yml) -n ${LAB_PROJECT}
oc create -f ${LAB_HOME}/src/preference/src/main/kubernetes/Service.yml -n ${LAB_PROJECT}
```

Since preference service is an intermediate service (`customer -> preference -> recommendation`),
it is not exposed to the outside world.

Wait for it to completely roll out and receive a `deployment xxxxxx successfully rolled out` result:

```bash
oc rollout status -w deployment/preference
```

Now, test the customer service again (which will call the preference service), which should fail:

```bash
curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
```

You should see `customer => 503 preference => I/O error on GET request for "http://recommendation:8080": recommendation; nested exception is java.net.UnknownHostException: recommendation`
as the recommendation service is not yet deployed.

## Step 5: Deploy Recommendations service version 1 (`v1`)

Istio introduces the concept of a service version, which is a finer-grained way to subdivide
service instances by versions (`v1`, `v2`) or environment (`staging`, `prod`). These variants are not
necessarily different API versions: they could be iterative changes to the same service, deployed
in different environments (prod, staging, dev, etc.). Common scenarios where this is used include
A/B testing or canary rollouts. Istio’s [traffic routing rules](https://istio.io/docs/concepts/traffic-management/rules-configuration.html) can refer to service versions to
provide additional control over traffic between services.

Let's deploy the recommendations `v1` pod with its sidecar.

```bash
oc apply -f <(istioctl kube-inject -f ${LAB_HOME}/src/recommendation/src/main/kubernetes/Deployment.yml) -n ${LAB_PROJECT}
oc create -f ${LAB_HOME}/src/recommendation/src/main/kubernetes/Service.yml -n ${LAB_PROJECT}
```

Since the recommendation service is at the end of our service chain (`customer -> preference -> recommendation`),
it is not exposed to the outside world.

Wait for it to completely roll out and receive a `deployment xxxxxx successfully rolled out` result:

```bash
oc rollout status -w deployment/recommendation-v1
```

> NOTE: The tag "v1" at the end of the deployment is important. We will be creating a v2 version of
recommendation later in this lab. Having both a v1 and v2 version of the recommendation
code will allow us to exercise some interesting aspects of Istio's capabilities.

Now, test the customer service again (which will call the preference service which in turn calls
the recommendation service). Now that it's all deployed, it should work:

```bash
curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
```

You should see: `customer => preference => recommendation v1 from '99634814-sf4cl': 1`.`

This shows that the call was successful. It also shows the hostname of the recommendation service's pod (`99634814-sf4cl`, yours will be different) and the
number of times (`1`) that the service has been called since it started. Run the above commands multiple times
to see this number go up.

## Step 6: Deploy Recommendations service version 2 (`v2`)

We can experiment with Istio routing rules by deploying a second version of the recommendations
service:

```bash
oc apply -f <(istioctl kube-inject -f ${LAB_HOME}/src/recommendation/src/main/kubernetes/Deployment-v2.yml) -n ${LAB_PROJECT}
```

You can see both versions of the recommendation pods running using `oc get pods`:

```bash
oc get pods -l app=recommendation

NAME                                 READY     STATUS    RESTARTS   AGE
recommendation-v1-60483540-9snd9     2/2       Running   0          12m
recommendation-v2-2815683430-vpx4p   2/2       Running   0          15s

```

By default, Istio will round-robin incoming requests to the Recommendations _Service_
so that both `v1` and `v2` pods get equal amounts of traffic:

```bash
for i in $(seq 10); do
  curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
done
```

Approximately half of the requests above go to `v1`, and half to `v2`.

The default Kubernetes/OpenShift behavior is to round-robin load-balance across all
available pods behind a single Service. Add another replica of `v2`:


```bash
oc scale --replicas=2 deployment/recommendation-v2
```

Now, you will see double the number of requests to `v2` than for `v1`:

```bash
for i in $(seq 10); do
  curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
done
```

Go back to 1 copy:

```bash
oc scale --replicas=1 deployment/recommendation-v2
```

## Step 7: Send all traffic to `recommendation:v2`

_Route rules_ control how requests are routed within an Istio service mesh.
Route rules provide:

* **Timeouts**
* **Bounded retries** with timeout budgets and variable jitter between retries
* **Limits** on number of concurrent connections and requests to upstream services
* **Active (periodic) health checks** on each member of the load balancing pool
* **Fine-grained circuit breakers** (passive health checks) – applied per instance in the load balancing pool

Requests can be routed based on
the source and destination, HTTP header fields, and weights associated with individual service versions.
For example, a route rule could route requests to different versions of a service.

In addition to the usual OpenShift object types like `BuildConfig`, `DeploymentConfig`,
`Service` and `Route`,
you also have new object types installed as part of Istio like `RouteRule`. Adding
these objects to the running
OpenShift cluster is how you configure routing rules for Istio.

In this case, let's route all traffic to `v2`:

```bash
oc create -f ${LAB_HOME}/src/istiofiles/route-rule-recommendation-v2.yml -n ${PROJECT}
```

Inspect the rule:

```bash
oc get routerule/recommendation-default
```

And now access the `customer` service 10 times - all requests should end up talking to
`recommendation:v2`:

```bash
for i in $(seq 10); do
  curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
done
```

## Step 7: Send all traffic to `recommendation:v1`

Now let's move everyone to `v1`:

```bash
oc replace -f ${LAB_HOME}/src/istiofiles/route-rule-recommendation-v1.yml -n ${PROJECT}
```

> NOTE: We use `oc replace` instead of `oc create` since we are overlaying the previous rule

And test again:

```bash
for i in $(seq 10); do
  curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
done
```

All requests now to go `v1`.

Now let's go back to the start, and remove the rules to get back to default round-robin distribution
of requests:

```bash
oc delete -f ${LAB_HOME}/src/istiofiles/route-rule-recommendation-v1.yml -n ${PROJECT}
```

And test again:

```bash
for i in $(seq 10); do
  curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
done
```
Traffic should be equally split once again.

## Step 8: Use a Canary Deployment to slowly rollout `v2`

To start the process, let's send 10% of the users to the `v2` version, to do a canary test:

```bash
oc create -f ${LAB_HOME}/src/istiofiles/route-rule-recommendation-v1_and_v2.yml -n ${PROJECT}
```

Inspect the rule:

```bash
oc get routerule/recommendation-v1-v2
```

You can see the use of the `weight` of each route to control the distribution of traffic.

Now let's send in 10 requests:

```bash
for i in $(seq 10); do
  curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
done
```
You should see only 1 request to `v2`, and 9 requests (90%) to `v1`. In reality you may get
2 requests as our sample size is low, but if you invoked
it 10 million times you should get approximately 1 million requests to `v2`.

Now let's move it to a 75/25 split:

```bash
oc create -f ${LAB_HOME}/src/istiofiles/route-rule-recommendation-v1_and_v2_75_25.yml -n ${PROJECT}
```

And issue 10 more requests:

```bash
for i in $(seq 10); do
  curl "http://$(oc get route customer -n ${LAB_PROJECT} --template='{{ .spec.host }}')"
done
```
Now you should see 2 or 3 requests (~25%) going to `v2`. This process can be continued (and automated), slowly migrating
traffic over to the new version as it proves its worth in production over time.

Let's remove the route rules before moving on:

```bash
oc delete routerule --all
```

## Congratulations!

In this lab you learned how to deploy microservices to form a _service mesh_ using Istio.
You also learned how to do traffic shaping and routing using _Route Rules_ which instruct
the Istio sidecar proxies to distribute traffic according to specified policy.


# References

* [Istio Homepage](https://istio.io)
* [Red Hat OpenShift](https://openshift.com)
* Others...
