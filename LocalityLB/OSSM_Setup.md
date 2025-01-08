# OSSM Setup

Setting up, a Multi-Primary, Multi-cluster, Multi-network mesh on OpenShift.

## Pre-Reqs and Assumptions

This is multi-cluster, so you obviously need to have 2x OpenShift clusters setup and running. The following guide assumes these are in two different regions, (eu-west-1 and eu-central-1), as this makes the kubernetes topology labels that Istio can leverage easy to use. If that is not the case, you may need to adjust some of the parameters in various manifests to make it work as expected, or manually set some topology labels.

These two OpenShift clusters require external loadbalancer support, eg the Service type LoadBalancer must be able to create required loadbalancer resources. Usual Cloud loadbalancers such as an ELB work fine, for on-prem, MetalLB or hardware LB with appropriate integrations should work too. 

It's a multi-network mesh. Yhe default and recommended way to deploy OpenShift is within its own VPC or similar routed network, so each cluster has its own private network.

It's a multi-primary mesh. Although you can use External or Primary-Remote deployments, we want a multi-primary setup for added resilience, and workload isolation, so each cluster will host its own mesh control plane and services.

OSSM3 isn't particularly heavyweight, but you'll need at least one worker (as well as the control plane setup) for this to work happily. If you want to add Kiali, tracing, opentelemetry, etc, you may need more. Multiple workers in different availability zones will also give you more opportunity to try out more complex locality load balancer scenarios. 

To test this repo and scenarios, I used 2 regions in AWS with standard resources and connectivity. Nothing complicated.

I've mostly followed, mashed together and embelished the multi-primary multi-cluster setup as per these main docs, plus a few others...

https://github.com/istio-ecosystem/sail-operator/blob/main/docs/README.md

https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/

To keep the naming sensible, I've 2x clusters, which are aligned to their region.

| Cluster Name | AWS Region   | Istio clusterName | Istio network |
|--------------|--------------|-------------------|---------------|
| Central      | eu-central-1 | cluster1          | network1      |
| West         | eu-west-1    | cluster2          | network2      |

> [!TIP]
> We're going to be running commands on 2x clusters very regularly, so its recommended to configure a kubeconfig file that contains contexts for both clusters 


## Install and Configure Components

> [!NOTE]
> This repo uses a series of Kustomize manifests to deploy each component. These are split up into logical sections, and should be fairly self-explanatory. Order is important as some manifests rely on previously deployed components, and some components can take a few minutes to deploy and reconcile.

> [!IMPORTANT]
> Review the kustomize manifests before use, and adjust to fit your environment. You can use `kustomize build /path/to/manifests/component` to check and verify the output before deploying using `oc` or `kubectl`.


### Certificates

By default OSSM creates a self-signed CA on initial startup, which is used to create and sign all keys and certificates that are used within the mesh. In a multi-primary environment these CAs are not trusted by the other istiod's in the mesh, causing a failure of comms across the mesh.

A root CA and two intermediate CAs (one for each cluster) must be created and deployed to each cluster to enable secure comms.

> [!NOTE]
> In this repo, the certs that already exist in the `certs` dir, were created following the default OpenSSL based instructions using the [Plug in CA Certificates](https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/) Istio documentation, and the openssl config files contained within that dir. You can use these certificates as-is rather then create your own if just wanting a quick test, but obviously not on anything remotely serious.

Follow the instructions in the upstream documentation above or use an existing PKI/CA, to create a root CA and intermediate certs for each cluster, and place them in the cluster specific directory within the `certs` directory. These certificates will be used by the `kustomization.yaml` to create a kubernetes secret `cacerts` within the `istio-system` namespace in the next section.

### `ossm-namespaces` - Core Namespaces

OSSM3 doesn't mandate any namespace names to use, but the usual convention is to use `istio-system` and `istio-cni`. These manifests create and label the namespaces and creates the `cacerts` secret populated by the certs above. It also adds a console banner to each cluster displaying the Cluster Name, to make it easier to identify when using the UI.

Two overlays are provided to create cluster specific resources

```shell
% oc create --context west -k manifests/ossm3-namespace/overlays/west/
% oc create --context central -k manifests/ossm3-namespace/overlays/central/
```

### `ossm3-operator` - OpenShift Service Mesh 3 Operator

OSSM3 is delivered as an Kubernetes operator from within the OpenShift OLM, so we will be using that. 

An overlay is provided to to deploy the OSSM Operator which deploys OSSM3 TP2. The Kiali Operator version suitable for this version of OSSM3 is also deployed.

```shell
% oc create --context west -k manifests/ossm3-operator/overlays/ossm3-tp2/
% oc create --context central -k manifests/ossm3-operator/overlays/ossm3-tp2/
```

Verify the Operator has been deployed successfully. This can take a fee minutes to deploy. You should see output similar to the below.

```shell
% oc get csv -n istio-system                                                    (central/default)
NAME                               DISPLAY                            VERSION      REPLACES                           PHASE
kiali-operator.v1.89.8             Kiali Operator                     1.89.8       kiali-operator.v1.89.6             Succeeded
servicemeshoperator3.v3.0.0-tp.2   Red Hat OpenShift Service Mesh 3   3.0.0-tp.2   servicemeshoperator3.v3.0.0-tp.1   Succeeded
```

### `ossm3-mesh` - Deploy Istio Mesh

An instance of the Istio Custom Resource must be defined, which the operator then uses to build and configure the mesh.

Two overlays are provided to create cluster specific resources

```shell
% oc create --context west -k manifests/ossm3-mesh/overlays/west/
% oc create --context central -k manifests/ossm3-mesh/overlays/central/
```

Verify the Istio resource has been deployed successfully. You should see output similar to the below.

```shell
% oc wait --for=jsonpath='{.status.revisions.ready}'=1 istios/default
istio.sailoperator.io/default condition met
```

### `ossm3-gateways` - Cross Network (and optionally Ingress) Gateways

OSSM3 no longer auto-deploys gateways, leaving this to cluster admins to decide which option is best for their needs.
As we are deploying a multi-cluster mesh, we need to deploy a Cross Network or East-West Gateway, to ensure traffic that needs to traverse clusters is kept within the mesh. This repo also deploys a default Ingress Gateway deployment to assist when deploying and testing some scenarios.

Two overlays are provided to create cluster specific gateway resources.

```shell
% oc create --context west -k manifests/ossm3-gateways/overlays/west/
% oc create --context central -k manifests/ossm3-gateways/overlays/central/
```

Verify the gateway resources have been deployed successfully. You should see output similar to the below.

```shell
% oc get gateways.networking.istio.io -n istio-system
NAME                    AGE
cross-network-gateway   3m4s
```

As the Cross Network Gateway leverages a LoadBalancer, we can check this is available, as it is required in the next step to configure trust between the clusters.

```shell
% oc get svc istio-eastwestgateway -n istio-system
NAME                    TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                                                           AGE
istio-eastwestgateway   LoadBalancer   172.30.105.173   aa12345678qwertyuiop.eu-west-1.elb.amazonaws.com   15021:30653/TCP,15443:32553/TCP,15012:30413/TCP,15017:31251/TCP   6m19s

% dig +short a12345678qwertyuiop.eu-west-1.elb.amazonaws.com
34.249.xxx.3
34.249.yyy.92
```

### Configure Trust Between Clusters

In a multi-primary mesh, the Istio control plane leverages the kubernetes API on each cluster to discover services, endpoints, and verify clusters. To enable this, a remote secret for each cluster needs to be generated, and added to the opposite cluster, and vica versa. 

The `istioctl` CLI has a dedicated command to generate this remote secret. 

Run the following commands, (whilst using `--context`) to create a secret for each cluster, and write it to the other cluster using `oc apply` (whilst using `--context` but for the other cluster).

*Create remote secret on West and Write to Central*
```shell
% istioctl create-remote-secret --context="west" --name=cluster2 --namespace istio-system | oc apply -f - --context="central"
```
*Create remote secret on Central and Write to West*
```shell
% istioctl create-remote-secret --context="central" --name=cluster1 --namespace istio-system | oc apply -f - --context="west"
```

At this stage the mesh should be fully working.


## Verify the Mesh

This section leans upon the default Istio [multi-mesh verification](https://istio.io/latest/docs/setup/install/multicluster/verify/) and [Istiod and Envoy troubleshooting docs0(https://istio.io/latest/docs/ops/diagnostic-tools/proxy-cmd/), alongside the default Istio `helloworld` container image to test the mesh.

### `helloworld-sample` - Deploy the HelloWorld Service

The Istio community provides a simple to use and deploy `helloworld` service which has two versions (v1 & v2), which can be deployed to show off some simple scenarios. When using a browser or cURL command to target the service it simply responds with the version and the instance that is responding.

Two overlays are provided to create cluster specific resources. In this example, `helloworld-v1` is deployed to Central, and `helloworld-v2` is deployed to West. Both services are exposed using the same kubernetes service `helloworld` and are in a namespace called `sample`. This namespace is labeled to enable istio sidecar injection.

In addition, the overlay deploys a simple cURL deployment, with a single container to the same namespace. This is to help test comms within the mesh, by cURLing from the container to the service.

```shell
% oc create --context west -k manifests/helloworld-sample/overlays/west/
% oc create --context central -k manifests/helloworld-sample/overlays/central/
```

Verify the resources have been deployed successfully. You should see output similar to the below.

```shell
% oc get po -n sample                                                              (west/default)
NAME                             READY   STATUS    RESTARTS   AGE
curl-7cd64bb6c5-bft6v            2/2     Running   0          2m14s
helloworld-v2-779454bb5f-2jjz9   2/2     Running   0          2m14s
```

Check that Istio labels have been applied.
```shell
% oc get po helloworld-v2-779454bb5f-2jjz9 -n sample -o jsonpath='{.metadata.labels}' | jq
{
  "app": "helloworld",
  "pod-template-hash": "779454bb5f",
  "security.istio.io/tlsMode": "istio",
  "service.istio.io/canonical-name": "helloworld",
  "service.istio.io/canonical-revision": "v2",
  "topology.istio.io/network": "network2",
  "version": "v2"
}
```

cURL the service from one of the cURL deployments within the clusters. In this example we use West.

```shell
% oc exec --context="west" -n sample -c curl "$(oc get pod --context="west" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
```

Repeat a few times, or use a for-loop to try repeated times. You should see round-robin load balancing occur between the v1 and v2 endpoints.

```shell
% for i in {0..10}; do
oc exec --context="west" -n sample -c curl "$(oc get pod --context="west" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
done
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
```
