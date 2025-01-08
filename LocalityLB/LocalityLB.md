# Locality Load Balancing

Istio uses Locality to control load balancing strategies. Under the hood it simply exploits the locality load balancing functionality provided by the Envoy Proxy, configuring each Envoy proxy accordingly to meet the required goals.

Locality in Istio is simply how it defines a geographic location of a workload in the mesh, based on region/zone/subzone. This is translated into Envoy config, so much is tied into how Envoy understands and implements its idea of locality and load balancing, and how Istio interprets that config.

Within Kubernetes envs, Istio uses the default Kubernetes Topology labels (`topology.kubernetes.io/region` and `topology.kubernetes.io/zone`) to provide region and zone definition to Envoy. Subzone is set by Istio using the `topology.istio.io/subzone` label. All labels are applied and function at the node level.

Out of the box, OSSM3 has LocalityLB enabled by default, however it is not configured, and so simply load balances requests in a round-robin style, as was observed when verifying the mesh earlier.

OSSM3 supports all three of Istios Locality Load Balancer strategies, namely *failover*, *distribute* and *failoverPriority*.

Relevant and/or useful docs that were used to formulate this guide.
https://istio.io/latest/docs/ops/configuration/traffic-management/multicluster/
https://istio.io/latest/docs/tasks/traffic-management/locality-load-balancing/
https://istio.io/latest/docs/reference/config/networking/destination-rule/#LocalityLoadBalancerSetting

## Simple Failover (Outlier)

As LocalityLB is enabled by default with OSSM, the only thing needed to configure failover is to simply configure Outlier Detection on a DestinationRule for the service. Outlier Detection within Envoy is a passive health check and depends on the control plane (in this case the local Istiod) to ascertain upstream service availability.

Due to this, enabling outlier detection results in a cluster failover strategy, where all service requests rather than be round-robined between clusters based on service endpoints, traffic is controlled based on availability of services the local Istiod manages within a cluster. The end result is service traffic is contained to that cluster until a failure of the local service occurs, upon which Istio will failover to use the service in the second cluster.

To show this in action, deploy the `outlier-detection` overlay to the West cluster, which will deploy a DestinationRule resource that enables outlier detection. 

```shell
% oc create --context west -k manifests/ossm3-localitylb/overlays/outlier-detection
```

> [!NOTE]
> We are only deploying the DestinationRule to one cluster, so that we can clearly test and show the differences from both clusters in the mesh.

### Test

Using our earler cURL for-loop from the West cluster, we can see that traffic is constrained within the West cluster.

```shell
% for i in {0..10}; do
oc exec --context="west" -n sample -c curl "$(oc get pod --context="west" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
done
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
```

Whereas, from the Central cluster, round-robin load balancing still occurs.
```shell
% for i in {0..10}; do
oc exec --context="central" -n sample -c curl "$(oc get pod --context="central" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
done
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
```

If we artificially fail the local v2 pod on the West cluster whilst running our loop, we can see that the mesh dynamically fails over the service to use the v1 pod in the Central cluster, and reinstates back to the v2 pod in the West cluster when it has recovered.

```shell
% for i in {0..30}; do
oc exec --context="west" -n sample -c curl "$(oc get pod --context="west" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
done
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9
Hello version: v2, instance: helloworld-v2-779454bb5f-2jjz9    <<< v2 Pod scaled to 0
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw    
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt    <<< v2 Pod scaled to 1
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
```

### Cleanup

Remove the DestinationRule 

```shell
% oc delete --context west -k manifests/ossm3-localitylb/overlays/outlier-detection
destinationrule.networking.istio.io "helloworld-dest-rule" deleted
```

## Weighted Distribution

The `distribute` strategy provides a weighted method of distributing traffic across the mesh. Again, this is Envoy functionality based on its locality load balancing features.

Weighting depends on defining an originating locality (region/zone/subzone) and assigning weights to specific region/zone/subzone locality segments, which Envoy then uses to load balance accordingly within the mesh. 

In our 2 cluster setup, our clusters are placed in two regions, `eu-west-1` and `eu-central-1`, and we only have one worker, hence we only have one zone per region. From/To localities can be wildcarded at any segment to suit your distribution needs.

The following configuration distributes from all of our West cluster nodes (in region `eu-west-1` and zone `eu-west-1c`), and distributes traffic 90% to West cluster nodes, and 10% to Central cluster nodes.

```yaml
    distribute:
    - from: eu-west-1/eu-west-1c/*
        to:
        "eu-west-1/eu-west-1c/*": 90
        "eu-central-1/eu-central-1b/*": 10
```

To show this in action, deploy the `weighted-distribution` overlay to the West cluster, which will deploy a DestinationRule resource that creates the distribution described above. 

```shell
% oc create --context west -k manifests/ossm3-localitylb/overlays/weighted-distribution
```

### Test

Using our earler cURL for-loop from the West cluster, we can see that traffic is constrained within the West cluster.

```shell
% for i in {0..10}; do
oc exec --context="west" -n sample -c curl "$(oc get pod --context="west" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
done
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
```

Here we can clearly see 90% of the traffic stays within the West cluster (v2 instance), and 10% goes to the Central cluster (v1 instance).

Again we can scale the v2 instance down/up as before to prove failover when the v2 instance is offline and fail back when it it recovers.

```shell
% for i in {0..30}; do
oc exec --context="west" -n sample -c curl "$(oc get pod --context="west" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
done
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt
Hello version: v2, instance: helloworld-v2-779454bb5f-g4kvt    <<< v2 pod scaled to 0
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb    <<< v2 pod scaled to 1
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
```

### Cleanup

Remove the DestinationRule 

```shell
% oc delete --context west -k manifests/ossm3-localitylb/overlays/weighted-distribution
destinationrule.networking.istio.io "helloworld-dest-rule" deleted
```

## Failover Priority

The `failoverPriority` strategy provides a more granular approach, using an ordered list of labels to sort endpoints to create priority based load balancing. 
The Istio and Envoy documentation explains this pretty well, but simplistically, assuming 3 labels are used to sort endpoints, those that match 3 labels, have highest priority (P0), those that match 2 next highest (P1), 1 match less priority (P2), and zero match the least (P3).

Unlike the other Locality Load Balancer strategies, as failoverPriority sorts and prioritises endpoints, arbitrary labels as well as locality, can be used to make decisions.

The following config is a simple 2 label example. In this example endpoints that match both labels will have the highest priority (P0), match one label will have middle priority, and match no labels, the lowest priority

```yaml
    failoverPriority:
    - "app=helloworld"
    - "version=v2"
```

With the helloworld service we currently have deployed, this means...
The service in the West cluster, matches both labels (`app=helloworld` and `version=v2`), and hence has highest priority - (P0)
The service in the Central cluster, matches only one label (`app=helloworld`) and hence has lower priority - (P1)

To show this in action, deploy the `failover-priority` overlay to the West cluster, which will deploy a DestinationRule resource that creates the failoverPriority strategy described above. 

```shell
% oc create --context west -k manifests/ossm3-localitylb/overlays/failover-priority
```

### Test

Using our earler cURL for-loop from the West cluster, we can see that traffic is constrained to the v2 instance on the West cluster.

```shell
% for i in {0..10}; do
oc exec --context="west" -n sample -c curl "$(oc get pod --context="west" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
done
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
```

And again, we can scale the v2 instance down/up as before to prove failover to the v1 instance when the v2 instance is offline, and recovery back to the v2 instance when it recovers.

```shell
% for i in {0..30}; do
oc exec --context="west" -n sample -c curl "$(oc get pod --context="west" -n sample -l app=curl -o jsonpath='{.items[0].metadata.name}')" -- curl -sS helloworld.sample:5000/hello;
done
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb
Hello version: v2, instance: helloworld-v2-779454bb5f-n55kb   <<< Scale v2 to zero
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v1, instance: helloworld-v1-69ff8fc747-cz9vw
Hello version: v2, instance: helloworld-v2-779454bb5f-fdmkr   <<< Scale v2 to 1
Hello version: v2, instance: helloworld-v2-779454bb5f-fdmkr
Hello version: v2, instance: helloworld-v2-779454bb5f-fdmkr
Hello version: v2, instance: helloworld-v2-779454bb5f-fdmkr
Hello version: v2, instance: helloworld-v2-779454bb5f-fdmkr
Hello version: v2, instance: helloworld-v2-779454bb5f-fdmkr
Hello version: v2, instance: helloworld-v2-779454bb5f-fdmkr
Hello version: v2, instance: helloworld-v2-779454bb5f-fdmkr
```

### Cleanup

Remove the DestinationRule 

```shell
% oc delete --context west -k manifests/ossm3-localitylb/overlays/failover-priority
destinationrule.networking.istio.io "helloworld-dest-rule" deleted
```