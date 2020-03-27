# example ocp statefulset

Common requirements for stateful applications include the ability to support sticky sessions.

One of the main goals of the OpenShift container platform is to be a world-class platform for stateless and stateful applications. In order to support stateful applications, a variety of tools are available to make virtually any application container-native. This example will walk you through the most popular tools, including headless services, sticky sessions and stateful sets, just to name a few. At the end of the example, you’ll walk through the power of the stateful set, which brings many stateful applications to life on OpenShift. However we will not cover application level session replication as this is application or middleware specific.

We want to showcase stickiness in OpenShift based on an example application with multiple replicas.

---

## Excersise

In the excercise we want to install the helm chart and showcase a simple scenario for sticky sessions in OpenShift.

Let's start by installing the resources from the helm chart via

```
$ helm install hello .
```

This installs via helm based on the default config specified in `values.yaml`
* a statefulset running our example application with 3 replica
* a ServiceAccount for our statefulset
* a SecurityContextContraint (scc) to allow the ServiceAccount used by our statefulset to run the application with a fixed UUID
* a service
* a route to access our application

To verify that our application is working as expected we can open the newly created route in a browser. You can either find the correct Route URL in the OpenShift Console (Web) or via the cli

```
$ oc get route hello
```

As long as you do not delete the cookies you are using to access the website from the browser and the pod that is serving your request is not terminated you will always end up on the same pod.

What happens when we manually kill the pod we are currently connected to? Let's find out! First we need to be quick here or OpenShift will have already spun another replica up and you might end up on a pod with the same name.

So be quick and terminate the pod that you are currently connect to (e.g. hello-0) by executing

```
$ oc delete pod hello-0
```

Quickly open your browser and refresh the page. You should get a response from another replica now.

Now let's do the same but with curl from our comandline

```
curl -ks https://$(oc get route hello -o jsonpath={.spec.host}) | grep "Server"
```

When you run it multiple times you should realize that the backend pod varies. But why?!
Well we curl the website without storing our cookie and therefore every request looks like a new user session to OpenShift.

In order to get the same behaviour as we have seen in the browser beforhand we need to store and use a cookie.

```
curl -ks -o /dev/null \
    --cookie-jar cookies.txt \
    https://$(oc get route hello \
    -o=jsonpath='{.spec.host}')
```

Perfect, now we got our cookie file. Next we can use the cookie file for multiple curl commands and realize that every request is served by the same pod.

```
$ for I in $(seq 1 8); \
  do \
      curl -ks \
      --cookie cookies.txt \
      https://$(oc get route hello -o=jsonpath='{.spec.host}') | grep "Server"; \
      printf "\n"; \
  done
```

Wonderful, that's all for now.

---

## Conclusion

We installed a statefulset in OpenShift via helm and exposed it via a route. A web browser will automatically store cookies for us and our sticky session works out of the box. We always end up on the same pod unless the pod gets terminated or we delete our cookies.

If we use curl from the commandline we did not get sticky sessions out of the box, since by default curl does not store cookies from one request to the next. We need to store our cookie manually and use it in our curl command in order to get the same behavior as with a web browser.

What happens to the application state associated to my session if the pod is terminated? It's lost. In this example we did not cover application level session replication. If a pod is terminated all sessions of that pod are lost. E.g. if your app has a login functionality and are logged in before your pod terminates, you will be logged out after the pod terminated and another pod, unaware of your session, handles your request.

---

**Source**
* https://livebook.manning.com/book/openshift-in-action
* https://kubernetes.io/docs/concepts/services-networking/service/
* https://access.redhat.com/documentation/en-us/openshift_container_platform/3.11/html/architecture/core-concepts