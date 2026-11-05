# BGP Architecture and Test Results

## 1. Overview

This phase extends the existing single-node Kubernetes LoadBalancer lab with MetalLB BGP mode and an external FRRouting (FRR) peer.

The original Layer 2 configuration remains in place. BGP uses a separate IP address pool and a dedicated LoadBalancer Service.

## 2. Network Topology

| Component | Configuration |
|---|---|
| Kubernetes node | 192.168.166.129 |
| Kubernetes node interface | ens37 |
| MetalLB local ASN | 64512 |
| FRR peer ASN | 64513 |
| Host-side veth address | 10.255.255.1/30 |
| FRR namespace veth address | 10.255.255.2/30 |
| BGP peer address | 10.255.255.2 |
| BGP TCP port | 179 |
| BGP LoadBalancer VIP | 192.168.166.60/32 |

### BGP topology

```text
Kubernetes Node                 FRR network namespace
MetalLB Speaker                 frr-router
ASN 64512                       FRRouting ASN 64513
     |                                  |
     | 10.255.255.1/30                  | 10.255.255.2/30
     +------------- veth pair ----------+
                    BGP / TCP 179

Advertised prefix: 192.168.166.60/32
```

## 3. MetalLB Configuration

The BGP configuration uses three resources:

- `BGPPeer`: Defines the local and remote ASNs and peer addresses.
- `IPAddressPool`: Reserves `192.168.166.60/32` for BGP.
- `BGPAdvertisement`: Advertises this pool to `frr-peer`.

The BGP pool uses `autoAssign: false`, so the address is allocated explicitly to the `nginx-bgp-lb` Service.

### BGP peer

- Resource: `frr-peer`
- Local ASN: `64512`
- Peer ASN: `64513`
- Peer address: `10.255.255.2`
- Source address: `10.255.255.1`

### BGP address pool

- Pool: `bgp-vip-pool`
- Address: `192.168.166.60/32`
- Automatic allocation: Disabled

### BGP advertisement

- Resource: `bgp-vip-advertisement`
- Address pool: `bgp-vip-pool`
- Peer: `frr-peer`

## 4. FRRouting Configuration

FRRouting runs inside a dedicated Linux network namespace named `frr-router`.

The FRR router uses ASN `64513`, while MetalLB uses ASN `64512`.

A veth pair connects the two routing environments:

- Host side: `10.255.255.1/30`
- FRR side: `10.255.255.2/30`

The BGP session uses TCP port 179.

### Routing policies

FRR applies an inbound route map that permits only `192.168.166.60/32` from the MetalLB peer.

An outbound route map denies all routes advertised from FRR to MetalLB.

This restricts the route exchange to the intended LoadBalancer VIP.

## 5. Kubernetes BGP Service

The `nginx-bgp-lb` Service:

- Uses type `LoadBalancer`.
- Receives VIP `192.168.166.60`.
- Selects Pods with label `app: nginx-lb`.
- Exposes TCP port 80.
- Uses the existing Nginx Deployment.

The original `nginx-lb` Service remains unchanged.

## 6. Layer 2 and BGP Coexistence

The original Layer 2 configuration remains intact:

| Component | Configuration |
|---|---|
| Layer 2 pool | `vmnet2-pool` |
| Layer 2 address range | 192.168.166.10–192.168.166.50 |
| Existing Layer 2 VIP | 192.168.166.10 |
| Layer 2 advertisement | `vmnet2-advertisement` |
| Advertisement interface | ens37 |

The BGP VIP `192.168.166.60/32` belongs to a separate pool and is advertised through the FRR peer.

The existing Flannel CNI configuration was preserved. The NAT interface `ens33` was not modified.

## 7. Phase 2: BGP Validation

The following checks were performed after applying the BGP resources.

| Check | Result |
|---|---|
| BGPPeer resource | Created |
| IPAddressPool | Created |
| BGPAdvertisement | Created |
| BGP LoadBalancer Service | Created |
| VIP allocation | 192.168.166.60 |
| MetalLB announcement protocol | BGP |
| BGP session | Established |
| Advertised prefix | 192.168.166.60/32 |
| FRR route installation | Successful |
| HTTP response from VIP | 200 OK |

### BGP session verification

FRR reported the MetalLB peer as established.

The BGP summary showed one received prefix:

```text
Neighbor       AS       State/PfxRcd
10.255.255.1   64512    1
```

### Route verification

FRR received and installed the advertised route:

```text
*> 192.168.166.60/32
   Next Hop: 10.255.255.1
   Path: 64512 i
```

The routing table confirmed:

```text
Routing entry for 192.168.166.60/32
Known via "bgp", distance 20, metric 0, best
* 10.255.255.1, via veth-frr
```

### HTTP validation

The BGP LoadBalancer VIP was tested using HTTP:

```text
curl http://192.168.166.60/
HTTP/1.1 200 OK
```

Result: Successful.

## 8. Phase 3: BGP Failure and Recovery Testing

A controlled failure test was performed by stopping FRRouting inside the `frr-router` namespace.

The Kubernetes cluster, MetalLB configuration, existing Layer 2 Service, and Nginx Deployment were not intentionally modified during this test.

### 8.1 Baseline

Before stopping FRR:

- BGP session was established.
- One prefix was received.
- Route `192.168.166.60/32` was installed in the FRR routing table.
- The BGP LoadBalancer Service retained VIP `192.168.166.60`.

### 8.2 FRR shutdown

FRR was stopped using:

```bash
sudo /usr/lib/frr/frrinit.sh stop frr-router
```

The MetalLB speaker logs reported:

- BGP session down.
- Peer de-configured notification.
- Connection refused during reconnection attempts.

The FRR namespace interfaces remained operational. The veth connectivity check succeeded:

```text
ping 10.255.255.1
2 packets transmitted, 2 received
0% packet loss
```

### 8.3 Route withdrawal

The FRR routing table was checked while the routing daemons were stopped:

```bash
sudo ip netns exec frr-router ip route show 192.168.166.60/32
```

No route was returned.

This confirmed that the BGP route was no longer installed in the FRR namespace routing table.

The Kubernetes Service continued to retain its allocated VIP. This is expected because VIP allocation and BGP session availability are separate mechanisms.

### 8.4 FRR recovery

FRR was restarted:

```bash
sudo /usr/lib/frr/frrinit.sh start frr-router
```

The BGP session was re-established.

The FRR BGP summary showed:

```text
Neighbor       AS       State/PfxRcd
10.255.255.1   64512    1
```

The route was advertised again and reinstalled:

```text
*> 192.168.166.60/32
   Next Hop: 10.255.255.1
   Path: 64512 i
```

The routing table again identified the route as learned through BGP.

### 8.5 Post-recovery validation

The BGP VIP was tested again:

```text
curl http://192.168.166.60/
HTTP 200
```

The original Layer 2 and BGP Services retained their respective addresses:

| Service | External IP |
|---|---|
| nginx-lb | 192.168.166.10 |
| nginx-bgp-lb | 192.168.166.60 |

Result: Successful BGP session recovery and route re-advertisement.

## 9. Test Summary

| Test | Result |
|---|---|
| BGP peer establishment | Successful |
| BGP route advertisement | Successful |
| FRR route installation | Successful |
| HTTP access to BGP VIP | HTTP 200 |
| FRR shutdown detection | Successful |
| BGP session failure detection | Successful |
| Route withdrawal from FRR | Verified |
| BGP session recovery | Successful |
| Route re-advertisement | Successful |
| HTTP access after recovery | HTTP 200 |
| Existing Layer 2 Service preservation | Verified |

## 10. Lab Limitations

FRR runs inside a manually created Linux network namespace. The namespace and veth pair are not configured to survive a host reboot.

BGP session establishment, route exchange, route withdrawal, and recovery were verified.

HTTP access to the BGP VIP returned HTTP 200 before and after the failure test.

Client traffic routed through FRR from a separate subnet was not tested. Because the Windows host and Kubernetes node share the same subnet, successful HTTP access to the VIP does not prove that traffic traversed the FRR router.

This project is a single-node Kubernetes networking lab. It does not demonstrate multi-node BGP high availability or production routing.

## Conclusion

This project demonstrates two MetalLB operating modes in a self-managed Kubernetes environment: Layer 2 advertisement and BGP route advertisement.

The BGP implementation integrates MetalLB with an external FRRouting peer, advertises a dedicated LoadBalancer VIP, and verifies route installation.

Controlled failure testing confirms that FRR shutdown causes the BGP session to go down and the route to be withdrawn from the FRR routing table. Restarting FRR restores the BGP session and route advertisement.

The original Layer 2 configuration and Nginx Service remain intact throughout the BGP extension.
