# Test Results

## 1. MetalLB installation

| Check | Result |
|---|---|
| Controller Pod | Running |
| Speaker Pod | Running |
| MetalLB version | v0.15.2 |
| IPAddressPool | Created |
| L2Advertisement | Created |
| Advertisement interface | ens37 |

## 2. LoadBalancer Service

| Check | Result |
|---|---|
| Service | nginx-lb |
| Type | LoadBalancer |
| External IP | 192.168.166.10 |
| Port | 80 |
| Backend endpoints | 2 |

## 3. External connectivity

The Nginx Service was accessed successfully from:

- Ubuntu Kubernetes VM
- Windows host through VMware Host-only networking

HTTP response: 200 OK

## 4. Traffic distribution

Twenty HTTP requests were sent to the LoadBalancer IP.

Both Nginx Pods showed successful GET requests in their access logs.

Result: Both backend Pods received traffic.

## 5. Pod self-healing

One Nginx Pod was manually deleted.

The Kubernetes Deployment controller automatically created
a replacement Pod.

| Check | Result |
|---|---|
| Original Pod deleted | Successful |
| Replacement Pod created | Successful |
| Desired replicas restored | 2 |
| New Pod IP | 10.244.0.39 |
| Other Pod IP | 10.244.0.37 |

## 6. Service availability after Pod deletion

After the replacement:

- The LoadBalancer IP remained 192.168.166.10.
- Twenty consecutive HTTP requests returned status 200.
- The EndpointSlice contained both healthy backend addresses.

Result: Successful.

## Conclusion

The MetalLB Layer 2 LoadBalancer configuration was successfully
deployed and tested.

External connectivity, backend traffic distribution, and
Deployment self-healing were verified.
