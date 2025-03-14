# Equinix Metal Overview (using the [Equinix Metal CCM](https://github.com/packethost/packet-ccm))

## BGP with Equinix Metal

When deploying Kubernetes with Equinix Metal with the `--controlplane` functionality we need to pre-populate the BGP configuration in order for the control plane to be advertised and work in a HA scenario. Luckily Equinix Metal provides the capability to "look up" the configuration details (for BGP) that we need in order to advertise our virtual IP for HA functionality. We can either make use of the [Equinix Metal API](https://metal.equinix.com/developers/api/) or we can parse the [Equinix Metal Metadata service](https://metal.equinix.com/developers/docs/servers/metadata/).

**Note** If this cluster will be making use of Equinix Metal for `type:LoadBalancer` (by using the [Equinix Metal CCM](https://github.com/packethost/packet-ccm)) then we will need to ensure that nodes are set to use an external cloud-provider. Before doing a `kubeadm init|join` ensure the kubelet has the correct flags by using the following command `echo KUBELET_EXTRA_ARGS=\"--cloud-provider=external\" > /etc/default/kubelet`.

## Configure to use a container runtime

The easiest method to generate a manifest is using the container itself, below will create an alias for different container runtimes.

### containerd
`alias kube-vip="ctr run --rm --net-host ghcr.io/kube-vip/kube-vip:v0.3.8 vip /kube-vip"`

### Docker
`alias kube-vip="docker run --network host --rm ghcr.io/kube-vip/kube-vip:v0.3.8"`

## Creating HA clusters in Equinix Metal

### Creating a manifest using the API

We can enable `kube-vip` with the capability to discover the required configuration for BGP by passing the `--metal` flag and the API Key and our project ID.

```
kube-vip manifest pod \
    --interface $INTERFACE\
    --vip $VIP \
    --controlplane \
    --services \
    --bgp \
    --metal \
    --metalKey xxxxxxx \
    --metalProjectID xxxxx | tee  /etc/kubernetes/manifests/kube-vip.yaml
```

### Creating a manifest using the metadata

We can parse the metadata, *however* it requires that the tools `curl` and `jq` are installed. 

```
kube-vip manifest pod \
    --interface $INTERFACE\
    --vip $VIP \
    --controlplane \
    --services \
    --bgp \
    --peerAS $(curl https://metadata.platformequinix.com/metadata | jq '.bgp_neighbors[0].peer_as') \
    --peerAddress $(curl https://metadata.platformequinix.com/metadata | jq -r '.bgp_neighbors[0].peer_ips[0]') \
    --localAS $(curl https://metadata.platformequinix.com/metadata | jq '.bgp_neighbors[0].customer_as') \
    --bgpRouterID $(curl https://metadata.platformequinix.com/metadata | jq -r '.bgp_neighbors[0].customer_ip') | sudo tee /etc/kubernetes/manifests/vip.yaml
```

## Load Balancing servies on Equinix Metal

Below are two examples for running `type:LoadBalancer` services on worker nodes only and will create a daemonset that will run `kube-vip`. 

**NOTE** This use-case requires the [Equinix Metal CCM](https://github.com/packethost/packet-ccm) to be installed and that the cluster/kubelet is configured to use an "external" cloud provider.

### Using Annotations

This is important as the CCM will apply the BGP configuration to the [node annotations](https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/) making it easy for `kube-vip` to find the networking configuration it needs to expose load balancer addresses. The `--annotations metal.equinix.com` will cause kube-vip to "watch" the annotations of the worker node that it is running on, once all of the configuarion has been applied by the CCM then the `kube-vip` pod is ready to advertise BGP addresses for the service.

```
kube-vip manifest daemonset \
  --interface $INTERFACE \
  --services \
  --bgp \
  --annotations metal.equinix.com \
  --inCluster | k apply -f -
```

### Using the existing CCM secret 

Alternatively it is possible to create a daemonset that will use the existing CCM secret to do an API lookup, this will allow for discovering the networking configuration needed to advertise loadbalancer addresses through BGP.

```
kube-vip manifest daemonset --interface $INTERFACE \
--services \
--inCluster \
--bgp \
--metal \
--provider-config /etc/cloud-sa/cloud-sa.json | kubectl apply -f -
```

### Expose with Equinix Metal (using the `kube-vip-cloud-provider`)

Either through the CLI or through the UI, create a public IPv4 EIP address.. and this is the address you can expose through BGP!

```
# packet ip request -p xxx-bbb-ccc -f ams1 -q 1 -t public_ipv4                                                                   
+-------+---------------+--------+----------------------+
|   ID  |    ADDRESS    | PUBLIC |       CREATED        |
+-------+---------------+--------+----------------------+
| xxxxx |     1.1.1.1   | true   | 2020-11-10T15:57:39Z |
+-------+---------------+--------+----------------------+

kubectl expose deployment nginx-deployment --port=80 --type=LoadBalancer --name=nginx --load-balancer-ip=1.1.1.1
```

## Troubleshooting

If `kube-vip` has been sat waiting for a long time then you may need to investigate that the annotations have been applied correctly by doing running the `describe` on the node:

```
kubectl describe node k8s.bgp02
...
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                    node.alpha.kubernetes.io/ttl: 0
                    metal.equinix.com/node-asn: 65000
                    metal.equinix.com/peer-asn: 65530
                    metal.equinix.com/peer-ip: x.x.x.x
                    metal.equinix.com/src-ip: x.x.x.x
```

If there are errors regarding `169.254.255.1` or `169.254.255.2` in the `kube-vip` logs then the routes to the ToR switches that provide BGP peering may by missing from the nodes. They can be replaced with the below command:

```
GATEWAY_IP=$(curl https://metadata.platformequinix.com/metadata | jq -r ".network.addresses[] | select(.public == false) | .gateway")
ip route add 169.254.255.1 via $GATEWAY_IP
ip route add 169.254.255.2 via $GATEWAY_IP
```

Additionally examining the logs of the Packet CCM may reveal why the node is not yet ready.