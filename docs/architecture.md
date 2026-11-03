# Architecture

## Overview

This project demonstrates external access to a Kubernetes workload
using MetalLB in Layer 2 mode.

The cluster runs on a single Ubuntu VMware virtual machine.
A VMware Host-only network provides connectivity between the
Windows host and the Kubernetes node.

## Network topology

| Component | Address / Configuration |
|---|---|
| Windows VMnet2 adapter | 192.168.166.1 |
| Kubernetes node interface | ens37 |
| Kubernetes node VMnet2 IP | 192.168.166.129 |
| MetalLB address pool | 192.168.166.10-192.168.166.50 |
| Allocated LoadBalancer IP | 192.168.166.10 |
| Kubernetes Service ClusterIP | 10.103.37.246 |
| Service port | 80 |
| Pod network | 10.244.0.0/16 |

## Traffic flow

1. A client on the Windows host accesses `192.168.166.10:80`.
2. MetalLB speaker advertises the LoadBalancer IP using Layer 2.
3. The Kubernetes node receives traffic through `ens37`.
4. The Kubernetes Service forwards traffic to an available endpoint.
5. One of the Nginx Pods responds to the request.

## Kubernetes components

### MetalLB

MetalLB provides LoadBalancer functionality for this cluster.

- IPAddressPool: `vmnet2-pool`
- Address range: `192.168.166.10-192.168.166.50`
- L2Advertisement: `vmnet2-advertisement`
- Interface: `ens37`

### Deployment

The `nginx-lb` Deployment maintains two Nginx replicas.

- Image: `192.168.198.140:8443/nginx:1.28-alpine`
- Replicas: 2
- Pod selector: `app: nginx-lb`

### Service

The `nginx-lb` Service exposes the Deployment externally.

- Type: LoadBalancer
- External IP: `192.168.166.10`
- Port: 80
- Target port: 80

## Resilience

One Nginx Pod was deliberately deleted during testing.

The Deployment controller automatically created a replacement Pod,
restoring the desired replica count to two.

The Service retained its external IP and continued returning HTTP 200
responses after the replacement.

## Network considerations

The MetalLB address pool is separate from the VMware Host-only DHCP
range used in this lab.

The existing Flannel CNI configuration was preserved.
The NAT interface `ens33` was not modified.
