# MetalLB

GitOps configuration for MetalLB load balancer in the Goldentooth Kubernetes cluster, providing LoadBalancer services for on-premises infrastructure.

## Overview

MetalLB is a load balancer implementation for bare-metal Kubernetes clusters, providing the missing LoadBalancer service type. In the Goldentooth cluster, MetalLB operates in L2 mode to assign external IP addresses to services from a configured address pool.

## Configuration

### Network Configuration
- **Address Pool**: `10.4.11.0 - 10.4.15.254` (1,280 available IPs)
- **Mode**: Layer 2 (L2) announcement mode
- **Network**: Part of the cluster's infrastructure CIDR `10.4.0.0/20`

### Deployment Details
- **Helm Chart**: Official MetalLB chart v0.14.5
- **Namespace**: `metallb`
- **FRR**: Disabled (L2-only mode)
- **Speaker**: Deployed as DaemonSet on all nodes

## Components

### Core Resources

#### `IPAddressPool.yaml`
Defines the IP address range available for LoadBalancer services:
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: primary
spec:
  addresses:
    - "10.4.11.0 - 10.4.15.254"
```

#### `L2Advertisement.yaml`
Configures Layer 2 advertisement for IP addresses:
```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: primary
spec:
  ipAddressPools:
    - primary
```

#### `Application.yaml`
Argo CD Application manifest for GitOps deployment:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metallb
spec:
  source:
    repoURL: https://metallb.github.io/metallb
    chart: metallb
    targetRevision: 0.14.5
```

## L2 Mode Operation

MetalLB operates in Layer 2 mode, which means:
- **ARP/NDP**: Speaker pods respond to ARP requests for assigned IPs
- **Single Node**: Each service IP is announced from one node at a time
- **Failover**: Automatic failover if the announcing node fails
- **No BGP**: Simplified configuration without BGP routing requirements

## Integration with Goldentooth Infrastructure

### HAProxy Load Balancer
MetalLB works in conjunction with HAProxy for:
- **Frontend**: HAProxy provides SSL termination and routing
- **Backend**: MetalLB services act as HAProxy backends
- **Health Checks**: HAProxy monitors MetalLB service endpoints

### External DNS
Services with LoadBalancer IPs automatically get DNS records via ExternalDNS:
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "service.goldentooth.net"
spec:
  type: LoadBalancer
```

### Network Architecture
```
Internet → Router → HAProxy → MetalLB LoadBalancer → Kubernetes Service → Pods
```

## Service Examples

### HTTP Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: "web.goldentooth.net"
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: web-app
```

### Database Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: LoadBalancer
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: postgres
```

## Monitoring

MetalLB provides Prometheus metrics for:
- **IP Allocation**: Pool utilization, available addresses
- **Service Health**: LoadBalancer service status
- **Speaker Status**: Node advertisement health
- **Controller Metrics**: Configuration sync, API operations

## Troubleshooting

### Common Issues

#### IP Address Exhaustion
Monitor available IPs in the pool:
```bash
kubectl get ipaddresspools -n metallb
```

#### Service Pending
Check MetalLB controller logs:
```bash
kubectl logs -n metallb deployment/metallb-controller
```

#### Network Connectivity
Verify speaker pod status:
```bash
kubectl get pods -n metallb -l app.kubernetes.io/component=speaker
```

## Migration Notes

The cluster previously used BGP mode but was migrated to L2 mode for:
- **Simplified Configuration**: No BGP peer management required
- **Hardware Compatibility**: Works with consumer-grade networking equipment
- **Reduced Complexity**: Easier troubleshooting and maintenance
