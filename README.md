## How to create a custom shard router not managed by the operator

**NOTE:** Before deploying a custom router, please make sure to read and understand the Disclaimer section of this document.

### Deploying a custom router

#### Adding a router shard with the customizations

This approach does not requires to disable the Ingress Router Operator because the customized router is installed through a custom `Deployment` as a router shard.
Using a router shard requires an additional application FQDN (the so colled ingress wildcard domain).
The DNS entry for the FQDN needs to be managed manually or via the External DNS operator.

#### Requirements

DNS management needed

## Deploy the test application:

Before starting deploying a customized router we will need an application to be used as a test.

1. Create the app:

```
$ oc new-app https://github.com/sclorg/cakephp-ex
```

2. Expose the service:

You may need to set the proper route hostname

```
$ ROUTE_HOSTNAME=cachephp-ex.shard.ocp.example.com
$ oc expose svc cakephp-ex --hostname ${ROUTE_HOSTNAME}
```

4. Optional: in case you are using a shard, you will need to annotate the route in order to "land" on the proper router instance, `type=shard` is just an example here, the label must match the `routeSelector` of the shard:

```
$ oc label route cakephp-ex -n cakephp-ex type=sharded
```

## Creating an unmanaged router shard

### Segregate the default router - OPTIONAL

**ACTION GOAL:** run the default router pods on a predefinite set of worker nodes. Is not needed if you are willing to execute the default router and the shard on the same set of nodes

In the following steps we will segregate the default router to be hosted only on nodes with the `router-sharded: no`.

Add the `nodePlacement` to the default IngressController:

```
oc edit IngressController default -n openshift-ingress-operator

[...]
spec:
  nodePlacement:
    nodeSelector:
      matchLabels:
        router-sharded: "no"
        node-role.kubernetes.io/worker: ""
[...]
```

### Excluding the sharded routes from the default router - DESIRED

**ACTION GOAL:** exclude from the default router all the routes with a specific label

We also need to skip the routes with `type: sharded` label from the default router.

Skip the routes with `type: sharded` from the default router, in this way the default router will ignore them:

```
oc patch \
  -n openshift-ingress-operator \
  IngressController/default \
  --type='merge' \
  -p '{"spec":{"routeSelector":{"matchExpressions":[{"key":"type","operator":"NotIn","values":["sharded"]}]}}}'
```

### Create a shard from the Ingress Operator

**ACTION GOAL:** let the Ingress Operator to create a shard (a new `Deployment` of a router). This can be useful to have all the resources in order to manually create a shard.

This section documents how to create a shard using the Ingress Operator, deployng an additional router on the cluster.

**NOTE:** the deployed router cannot be customized because is managed by the Ingress operator, every change will be reconciliated.
This is useful just to have all the needed resources to use as a skeleton for the manual deployment.

```
$ export DOMAIN_NAME=shard.ocp.example.com

$ oc create -f - <<EOF
apiVersion: operator.openshift.io/v1
kind: IngressController
metadata:
  name: sharded
  namespace: openshift-ingress-operator
spec:
  domain: ${DOMAIN_NAME}
  nodePlacement:
    nodeSelector:
      matchLabels:
        node-role.kubernetes.io/worker: ""
        router-sharded: "yes"
  routeSelector:
    matchLabels:
      type: sharded
status: {}
EOF
```

You can now backup all the objects created by the Ingress Operator: `oc adm inspect ns/openshift-ingress --dest-dir=./openshift-ingress.backup`

Before creating a manual shard you will need to remove the managed shard: `oc delete IngressController -n openshift-ingress-operator sharded`

### How to manually create a shard

**ACTION GOAL:** install a `Deployment` and all the needed resources to execute an Ingress Router. This router is not managed by the Ingress Controller, thus can be customized.

In order to be able to customize the router configuration of a shard, the router must not be managed by the Ingress Controller, otherwise every change will be reconciliated by the operator.
For that reason we have to deploy an "unmanaged" router. This repo contains an Helm chart with all the needed resources.
The Helm templates are created out of shard from a 4.12.28 OCP cluster, deploying this Helm chart on a different version requires a double-check of the resources.

1. Clone this repo `git clone https://github.com/samuano76/OpenShift-Sispi-CustomRoute-HAProxy26.git`
2. Configure the `helm/values.yaml`:
    - `ingressFQDN` is the wildcard application FQDN for the shard
    - `tlsCert` and `tlsKey` are the certificate and the key used by the router (must match the `ingressFQDN`)
    - `routeLabels` defines the route selector that this shard will use
    - `image` is the router image, you can get it from the default router: `oc get deploy -n openshift-ingress router-default -o jsonpath="{.spec.template.spec.containers[0].image}"`
    - `useInternalLB` if an Internal Load Balancer should be instantiate or not, in case is used accepted values are `aws`, `azure, `gcp`.
    - `service` section to custimize the  external service used to expose the router
       - `annotations` annotations to add to the service
       - `type` the  service type
       - `allocateLoadBalancerNodePorts` if the service `spec.allocateLoadBalancerNodePorts` sould be set, to be used only with service type `LoadBalancer`
       - `httpNodePort` and `httpsNodePort` the node ports to set if service type is `NodePort`
    - `useHostNetwork` if the router pod should be exposed with `hostNetwork`
    - `httpRouterPort`, `httpsRouterPort` `metricsRouterPort` ports to use on the router pod
    - `replicas` configure the number of router pods
    - `customHAProxyTemplate` section for a custom HAProxy template file
        - `useCustomTemplate` if a custom template should be used (true of false)
        - `file` the file to use as custom template, you can obtain the default one with `oc rsh -n openshift-ingress deploy/${ROUTER_NAME} cat haproxy-config.template > haproxy-config.template`
3. Install the chart: `helm install -n openshift-ingress sharded ./helm/`
4. Verify the pods are running: `oc get pods,svc,deploy -n openshift-ingress`

The helm chart deploys a `LoadBalancer` service to expose the router.
We need to create a DNS A record for the wildcard application FQDN of the shard pointg to the the external IP of the load balancer (see the next sections for that).

The chart will deploy a router into the `openshift-ingress` namespace which is not handled by the Ingress Operator.
If the `customHAProxyTemplate.useCustomTemplate` is not set the deployed router will not use a custom template.



### Router customization - OPTIONAL

**ACTION GOAL:** apply the HAProxy template customizations

1. Get the deployments names:

```
oc get deploy -n openshift-ingress
```

Take the appropriate deployment, depending on which router you want to customize: the default or an unmanaged shard.

2. Get the default template:

```
oc rsh -n openshift-ingress deploy/${ROUTER_NAME} cat haproxy-config.template > haproxy-config.template
```

2. Apply the patch:

3. Create the configmap with the template:

```
oc create configmap router-template --from-file=haproxy-config.template -n openshift-ingress
```

4. Mount the new template:

```
oc -n openshift-ingress set volume deploy/${ROUTER_NAME} --add --overwrite --name=template-volume --mount-path=/var/lib/haproxy/conf/custom  --source='{"configMap": { "name": "router-template"}}'
```

5. Set the `TEMPLATE_FILE` varialbe to use the new template

```
oc -n openshift-ingress set env deploy/${ROUTER_NAME} TEMPLATE_FILE=/var/lib/haproxy/conf/custom/haproxy-config.template
```
