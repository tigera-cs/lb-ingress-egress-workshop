# Lab 1: Calico LoadBalancer

## In this lab

This lab provides the instructions to:

* [Overview](#overview)
* [Create a LoadBalancer IPPool](#create-a-loadbalancer-ippool)
* [Advertise LoadBalancer IPs over BGP](#advertise-loadbalancer-ips-over-bgp)
* [Expose an application with a LoadBalancer Service](#expose-an-application-with-a-loadbalancer-service)
* [Preserve the client source IP with `externalTrafficPolicy: Local`](#preserve-the-client-source-ip-with-externaltrafficpolicy-local)
* [Request a specific IP address](#request-a-specific-ip-address)
* [Optional: Maglev load balancing (eBPF)](#optional-maglev-load-balancing-ebpf)
* [Clean up](#clean-up)

---

## Overview

A Kubernetes Service of `type: LoadBalancer` needs something to give it an external IP address. In a public cloud, the cloud's load balancer does that. On-premises, Calico Enterprise does it with **LoadBalancer IP address management (IPAM)**:

* The LoadBalancer controller in `calico-kube-controllers` assigns each LoadBalancer Service an IP address from an IPPool reserved for `LoadBalancer` use.
* Calico advertises those addresses over BGP (`serviceLoadBalancerIPs` in the BGPConfiguration), so the upstream router (the `bastion` in this lab) can reach them.
* With `externalTrafficPolicy: Cluster` (the default), every node advertises the whole LoadBalancer range. With `externalTrafficPolicy: Local`, only the nodes running a backing pod advertise a `/32` route for the Service.

Calico Enterprise does not create a LoadBalancer IPPool for you. You create one in this lab.

All commands run on the **bastion**.

---

## Create a LoadBalancer IPPool

### 1. Check the LoadBalancer controller

The LoadBalancer controller runs in `calico-kube-controllers` and is enabled by default. Confirm it assigns IP addresses to all LoadBalancer Services:

```bash
kubectl get kubecontrollersconfiguration default -o jsonpath='{.spec.controllers.loadBalancer}{"\n"}'
```

You should see `"assignIPs":"AllServices"` in the output. If not, enable it:

```bash
kubectl patch kubecontrollersconfiguration default --type=merge --patch '{"spec":{"controllers":{"loadBalancer":{"assignIPs":"AllServices"}}}}'
```

> [!NOTE]
> The other mode, `RequestedServicesOnly`, only assigns IPs to Services that carry a Calico IPAM annotation or set `spec.loadBalancerClass: calico`. Use it when another load balancer implementation shares the cluster.

### 2. Create the IPPool

Create an IPPool whose addresses can only be used by LoadBalancer Services.

* `allowedUses: [LoadBalancer]` reserves the pool for LoadBalancer Services. It cannot be combined with `Workload` or `Tunnel`, so no pod ever gets one of these addresses.
* `assignmentMode: Automatic` lets the controller hand out addresses without being asked. `Manual` reserves the pool for Services that request it by annotation.
* `10.50.0.0/26` sits inside `10.50.0.0/24`, which the bastion's BIRD import filter already accepts. It does not overlap the egress gateway pool used in Lab 3.

```yaml
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: loadbalancer-ip-pool
spec:
  cidr: 10.50.0.0/26
  blockSize: 31
  natOutgoing: true
  disabled: false
  assignmentMode: Automatic
  allowedUses:
    - LoadBalancer
EOF
```

### 3. List the IPPools

```bash
kubectl get ippools -o custom-columns=NAME:.metadata.name,CIDR:.spec.cidr,USES:.spec.allowedUses
```

You should see `loadbalancer-ip-pool` with `10.50.0.0/26` and `[LoadBalancer]`, next to the pod IPPools.

---

## Advertise LoadBalancer IPs over BGP

### 1. Check the routes on the bastion

```bash
ip route
```

There is no route for `10.50.0.0/26` yet.

### 2. Peer the cluster with the bastion

Create the BGPPeer to the bastion, the upstream router (AS `64512`). If it already exists from an earlier lab, `kubectl apply` leaves it unchanged.

```yaml
kubectl apply -f - <<EOF
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: bgppeer-global-64512
spec:
  peerIP: 10.0.1.10
  asNumber: 64512
EOF
```

### 3. Advertise the LoadBalancer range

Add the LoadBalancer range to the default BGPConfiguration. `serviceLoadBalancerIPs` tells Calico to advertise LoadBalancer Service IPs from that range.

```bash
kubectl patch bgpconfigurations default --type=merge -p '{"spec":{"serviceLoadBalancerIPs":[{"cidr":"10.50.0.0/26"}]}}'
```

> [!NOTE]
> If the command reports that `default` is not found, create it with `kubectl apply` instead: a BGPConfiguration named `default` with the same `serviceLoadBalancerIPs` list.

Verify the change:

```bash
kubectl get bgpconfigurations default -o jsonpath='{.spec.serviceLoadBalancerIPs}{"\n"}'
```

### 4. Check the BGP sessions on the bastion

```bash
sudo birdc show protocols
```

The sessions to `control1`, `worker1` and `worker2` should be `Established`.

---

## Expose an application with a LoadBalancer Service

### 1. Deploy the application

Deploy two nginx replicas and a Service of `type: LoadBalancer` in a new namespace.

```yaml
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: lb-demo
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: lb-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: lb-demo
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
EOF
```

### 2. Check the external IP

```bash
kubectl get svc -n lb-demo nginx
```

`EXTERNAL-IP` shows an address from `10.50.0.0/26`, assigned by Calico. Save it in a variable:

```bash
LB_IP=$(kubectl get svc -n lb-demo nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'); echo $LB_IP
```

> [!NOTE]
> If `EXTERNAL-IP` stays `<pending>`, the pool has no free addresses or the controller is not assigning IPs. Re-check [the LoadBalancer controller](#1-check-the-loadbalancer-controller) and the IPPool.

### 3. Check the routes on the bastion

```bash
ip route | grep -A3 10.50
```

With the default `externalTrafficPolicy: Cluster`, every node advertises the LoadBalancer range, so the bastion has an ECMP route with a next hop through each node.

### 4. Reach the Service from outside the cluster

```bash
curl -s $LB_IP | grep title
```

You should see `<title>Welcome to nginx!</title>`. The bastion is outside the cluster: the request followed the BGP route to a node, and the node forwarded it to an nginx pod.

---

## Preserve the client source IP with `externalTrafficPolicy: Local`

With `externalTrafficPolicy: Cluster`, any node accepts the traffic and may forward it to a pod on another node, replacing the client IP with its own (SNAT) on the way. With `Local`, only nodes that run a backing pod receive traffic, so there is no second hop and the pod sees the real client IP.

### 1. Scale to one replica and switch to Local

```bash
kubectl scale deployment -n lb-demo nginx --replicas=1
kubectl patch svc -n lb-demo nginx -p '{"spec":{"externalTrafficPolicy":"Local"}}'
```

### 2. Find the node running the pod

```bash
kubectl get pods -n lb-demo -o wide
```

### 3. Check the routes on the bastion

```bash
ip route | grep $LB_IP
```

There is now a `/32` route for `$LB_IP` with a single next hop: the node from the previous step.

### 4. Check the client IP the pod sees

```bash
curl -s $LB_IP > /dev/null
kubectl logs -n lb-demo deploy/nginx --tail=1
```

The log line starts with `10.0.1.10`, the bastion's own address. With `Cluster`, it could show a node IP instead.

---

## Request a specific IP address

A Service can ask for a particular address with the `projectcalico.org/loadBalancerIPs` annotation. The address must be free and inside a LoadBalancer IPPool. If it is taken, Calico does not fall back to another address.

```bash
kubectl annotate svc -n lb-demo nginx projectcalico.org/loadBalancerIPs='["10.50.0.10"]' --overwrite
kubectl get svc -n lb-demo nginx
```

`EXTERNAL-IP` changes to `10.50.0.10`. To choose a pool rather than an address, use `projectcalico.org/ipv4pools: '["<pool name>"]'`.

---

## Optional: Maglev load balancing (eBPF)

> [!IMPORTANT]
> This part needs **Calico Enterprise 3.23 or later** running the **eBPF dataplane** with `bpfExternalServiceMode: DSR` (direct server return). Switching the lab to eBPF restarts `calico-node` on every node and disables kube-proxy, so the instructor normally runs this part as a demo. On AWS, DSR also needs all nodes in one subnet and the source/destination check disabled.

Maglev turns Calico Enterprise into a distributed, horizontally scalable load balancer for external traffic. Every node builds the same Maglev consistent-hash lookup table for a Service, so whichever node the upstream router's ECMP picks, a given connection maps to the same backend pod. If a node goes down and the router moves the flow to another node, that node picks the same backend, and the connection survives.

Maglev does not apply to traffic from pods to Services or to NodePort connections, and it ignores `externalTrafficPolicy`: it chooses from all valid backends.

### 1. Confirm the dataplane and DSR mode

```bash
kubectl get installation default -o jsonpath='{.spec.calicoNetwork.linuxDataplane}{"\n"}'
kubectl get felixconfiguration default -o jsonpath='{.spec.bpfExternalServiceMode}{"\n"}'
```

You need `BPF` and `DSR`. To turn on DSR on a cluster already running eBPF:

```bash
kubectl patch felixconfiguration default --type=merge -p '{"spec":{"bpfExternalServiceMode":"DSR"}}'
```

### 2. Return to externalTrafficPolicy Cluster and two replicas

```bash
kubectl patch svc -n lb-demo nginx -p '{"spec":{"externalTrafficPolicy":"Cluster"}}'
kubectl scale deployment -n lb-demo nginx --replicas=2
```

### 3. Enable Maglev on the Service

Maglev is turned on per Service with an annotation:

```bash
kubectl annotate svc -n lb-demo nginx lb.projectcalico.org/external-traffic-strategy=maglev
```

### 4. Verify the Service is programmed for Maglev

Pick any `calico-node` pod and dump its eBPF NAT table:

```bash
NODE_POD=$(kubectl get pods -n calico-system -l k8s-app=calico-node -o jsonpath='{.items[0].metadata.name}')
kubectl -n calico-system exec $NODE_POD -c calico-node -- calico-node -bpf nat dump | grep maglev
```

Look for the `$LB_IP` entry with `flags maglev`, for example:

```text
10.50.0.10 port 80 proto 6 id 7 count 2 local 1 flags maglev
```

### 5. Size the lookup tables

Felix reserves memory for the Maglev tables based on two FelixConfiguration fields, both defaulting to `100` (range 1–3000): `bpfMaglevMaxServices` and `bpfMaglevMaxEndpointsPerService`. Raise them if you expect more Maglev Services or more backends per Service.

> [!TIP]
> For planned node maintenance, annotate the node with `lb.projectcalico.org/maintenance=exclude-local-backends`. New connections stop using its local backends, while existing ones continue.

---

## Clean up

Delete the demo application. **Keep** the LoadBalancer IPPool, the BGPPeer and the BGPConfiguration: the next labs use them.

```bash
kubectl delete namespace lb-demo
```

> **Congratulations! You have completed Lab 1: Calico LoadBalancer.**

---

[Workshop overview](../README.md) | [Next: Lab 2: Calico Ingress Gateway →](../2.Calico-Ingress-Gateway/README.md)
