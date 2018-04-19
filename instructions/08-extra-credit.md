<div>
 <div style="float: left"><a href="./07-blacklisting.md"><span>&lt;&lt;&nbsp;Previous</span></a></div>
<div style="float: right"><span>&nbsp;</span></div>
<div>
<br/>

# Smart routing (Canary Deployment)

In this lab we will learn how to do **Smart Routing** with Istio route rules using user agent request header.

What is your user-agent? 
https://www.whoishostingthis.com/tools/user-agent/

**NOTE**: 

The "user-agent" header being forwarded in the Customer and Preferences controllers in order for route rule modifications around recommendation

## What you will learn

* Route rules using request header

## Step 1

Send all traffic to **recommendation v1** 

```sh
oc create -f $ISTIO_LAB_HOME/src/istiofiles/route-rule-recommendation-v1.yml -n $ISTIO_LAB_PROJECT
```

## Step 2

Send all safari users to **recommendation v2** 

```sh
oc create -f $ISTIO_LAB_HOME/src/istiofiles/route-rule-safari-recommendation-v2.yml -n tutorial -n $ISTIO_LAB_PROJECT

istioctl get routerules -n tutorial
```
## Step 3 

Lets test the route rules, 

```sh
HOST=$(oc get route customer -n ${ISTIO_LAB_PROJECT} --template='{{ .spec.host }}')
curl -A Safari $HOST
```
you should see the above command sending all requests to **recommendation v2**.

```sh
HOST=$(oc get route customer -n ${ISTIO_LAB_PROJECT} --template='{{ .spec.host }}')
curl -A Firefox $HOST
```

you should see the above command sending all requests to **recommendation v1**.

## Step 4

Lets remove the Safari rule

```sh
istioctl delete routerule recommendation-safari -n $ISTIO_LAB_PROJECT
```

# Congratulations!

If you've made it this far, congratulations! You've learned the basics of the service mesh concept using OpenShift and Istio.

But you've only scratched the surface of the power of the service mesh for your containerized applications on OpenShift. For a more in-depth journey with Istio and OpenShift, Please visit the online tutorial [Learn Istio on OpenShift](https://learn.openshift.com/servicemesh), which is available 24 hours a day and provides a much more comprehensive set of exercises that cannot be covered in the short time we have today.

Congratulations on making it this far and we hope you enjoy the rest of Red Hat Summit 2018!

<div>
 <div style="float: left"><a href="./07-blacklisting.md"><span>&lt;&lt;&nbsp;Previous</span></a></div>
<div>