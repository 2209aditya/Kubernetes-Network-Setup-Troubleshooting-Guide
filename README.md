# 🌐 Kubernetes Network Setup & Troubleshooting Guide

A complete guide to understanding, setting up, and troubleshooting Kubernetes networking.

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Network](https://img.shields.io/badge/Network-Setup-blue?style=for-the-badge)](https://github.com)

## 📑 Table of Contents

- [Understanding Kubernetes Networking](#understanding-kubernetes-networking)
- [Network Models Overview](#network-models-overview)
- [CNI Plugin Installation](#cni-plugin-installation)
- [Service Types & Configuration](#service-types--configuration)
- [Network Policies](#network-policies)
- [DNS Configuration](#dns-configuration)
- [Ingress Setup](#ingress-setup)
- [Common Network Issues & Solutions](#common-network-issues--solutions)
- [Network Debugging Tools](#network-debugging-tools)
- [Best Practices](#best-practices)

---

## 🎯 Understanding Kubernetes Networking

Kubernetes networking addresses four main concerns:

1. **Container-to-Container** communication (within a Pod)
2. **Pod-to-Pod** communication (across the cluster)
3. **Pod-to-Service** communication (service discovery)
4. **External-to-Service** communication (ingress/egress)

### Key Networking Principles

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                   │
│                                                         │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐    │
│  │  Pod A   │────▶ │ Service  │◀──── │  Pod B   │    │
│  │ 10.1.1.2 │      │ 10.96.0.1│      │ 10.1.2.3 │    │
│  └──────────┘      └──────────┘      └──────────┘    │
│       │                  │                  │          │
│       └──────────────────┴──────────────────┘          │
│                    CNI Plugin                           │
└─────────────────────────────────────────────────────────┘
```

**Core Requirements:**
- Every Pod gets its own IP address
- Pods can communicate with all other Pods without NAT
- Nodes can communicate with all Pods without NAT
- The IP a Pod sees itself as is the same IP others see it as

---

## 🔌 Network Models Overview

### Popular CNI Plugins Comparison

| CNI Plugin | Network Model | Performance | Complexity | Best For |
|------------|---------------|-------------|------------|----------|
| **Calico** | Layer 3 BGP | High | Medium | Production, Network Policies |
| **Flannel** | Overlay (VXLAN) | Medium | Low | Simple setups, Small clusters |
| **Weave Net** | Mesh Overlay | Medium | Low | Easy setup, Auto-discovery |
| **Cilium** | eBPF | Very High | High | Advanced features, Security |
| **Canal** | Calico + Flannel | High | Medium | Best of both worlds |

---

## 🚀 CNI Plugin Installation

### Prerequisites

```bash
# Ensure kernel modules are loaded
sudo modprobe br_netfilter
sudo modprobe overlay

# Enable IP forwarding
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

### Option 1: Calico (Recommended for Production)

**Features:** Network policies, BGP routing, high performance

```bash
# Install Calico operator
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml

# Download custom resources
curl https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml -O

# Edit the CIDR if needed (default is 192.168.0.0/16)
# Match this with your kubeadm --pod-network-cidr flag

# Apply the configuration
kubectl create -f custom-resources.yaml

# Verify installation
kubectl get pods -n calico-system

# Check Calico node status
kubectl get nodes -o wide
```

**Configuration Example:**

```yaml
# calico-custom-resources.yaml
apiVersion: operator.tigera.io/v1
kind: Installation
metadata:
  name: default
spec:
  calicoNetwork:
    ipPools:
    - blockSize: 26
      cidr: 192.168.0.0/16
      encapsulation: VXLANCrossSubnet
      natOutgoing: Enabled
      nodeSelector: all()
```

### Option 2: Flannel (Easiest Setup)

**Features:** Simple, reliable, VXLAN overlay

```bash
# Apply Flannel
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

# Verify
kubectl get pods -n kube-flannel

# Check configuration
kubectl get configmap -n kube-flannel kube-flannel-cfg -o yaml
```

**Flannel Configuration:**

```yaml
# kube-flannel-config.yaml
net-conf.json: |
  {
    "Network": "10.244.0.0/16",
    "Backend": {
      "Type": "vxlan",
      "Port": 8472
    }
  }
```

### Option 3: Weave Net

**Features:** Simple mesh network, automatic discovery

```bash
# Install Weave Net
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Verify
kubectl get pods -n kube-system | grep weave

# Check status
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status
```

### Option 4: Cilium (Advanced)

**Features:** eBPF-based, high performance, advanced security

```bash
# Install Cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

# Install Cilium
cilium install --version 1.14.5

# Verify
cilium status --wait

# Run connectivity test
cilium connectivity test
```

---

## 🎛️ Service Types & Configuration

### 1. ClusterIP (Internal Only)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-internal-service
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80        # Service port
      targetPort: 8080  # Container port
```

**Use Case:** Internal microservice communication

### 2. NodePort (External Access via Node IP)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-nodeport-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 30080  # External port (30000-32767)
```

**Access:** `http://<node-ip>:30080`

### 3. LoadBalancer (Cloud Provider)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-loadbalancer-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
```

**Use Case:** Production external access on cloud platforms

### 4. ExternalName (DNS CNAME)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: database.example.com
```

---

## 🔒 Network Policies

### Enable Network Policies

Network policies require CNI plugin support (Calico, Cilium, Weave Net).

### Example 1: Deny All Traffic (Baseline)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

### Example 2: Allow Specific Traffic

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### Example 3: Allow DNS and External Access

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-and-external
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Egress
  egress:
  # Allow DNS
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: UDP
      port: 53
  # Allow external HTTPS
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32  # Exclude metadata service
    ports:
    - protocol: TCP
      port: 443
```

---

## 🌍 DNS Configuration

Kubernetes uses CoreDNS for service discovery.

### Check CoreDNS Status

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

### Custom DNS Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
           lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
           ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
           max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }
```

### Test DNS Resolution

```bash
# Create test pod
kubectl run test-dns --image=busybox --restart=Never -- sleep 3600

# Test DNS
kubectl exec test-dns -- nslookup kubernetes.default
kubectl exec test-dns -- nslookup my-service.default.svc.cluster.local

# Cleanup
kubectl delete pod test-dns
```

---

## 🚪 Ingress Setup

### Install NGINX Ingress Controller

```bash
# Using Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace

# Or using kubectl
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.5/deploy/static/provider/cloud/deploy.yaml

# Verify
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Ingress Resource Example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

---

## 🔧 Common Network Issues & Solutions

### Issue 1: Pods Can't Communicate

**Symptoms:**
```bash
# Pods in different nodes can't reach each other
kubectl exec pod-a -- ping pod-b-ip  # Request timeout
```

**Solutions:**

```bash
# 1. Check CNI plugin status
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave|cilium'

# 2. Verify pod network CIDR
kubectl cluster-info dump | grep -m 1 cluster-cidr
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range

# 3. Check node connectivity
kubectl get nodes -o wide
ping <node-ip>

# 4. Verify iptables rules
sudo iptables -L -n -v | grep cali  # For Calico
sudo iptables -t nat -L -n -v | grep KUBE

# 5. Check routing
kubectl exec <pod-name> -- ip route
kubectl exec <pod-name> -- route -n

# 6. Restart CNI pods
kubectl delete pod -n kube-system -l k8s-app=calico-node
# Or for Flannel
kubectl delete pod -n kube-flannel --all
```

**Prevention:**
- Ensure pod network CIDR doesn't overlap with node network
- Verify CNI plugin is properly installed before deploying workloads

---

### Issue 2: Service Not Accessible

**Symptoms:**
```bash
# Service endpoint returns error
curl http://my-service  # Connection refused
```

**Solutions:**

```bash
# 1. Check service endpoints
kubectl get svc my-service
kubectl get endpoints my-service
# If endpoints are empty, no pods match the selector

# 2. Verify pod labels match service selector
kubectl get pods --show-labels
kubectl describe svc my-service | grep Selector

# 3. Check if pods are ready
kubectl get pods -o wide
kubectl describe pod <pod-name>

# 4. Test from within cluster
kubectl run debug --image=nicolaka/netshoot -it --rm -- bash
curl http://my-service.default.svc.cluster.local

# 5. Check kube-proxy
kubectl get pods -n kube-system | grep kube-proxy
kubectl logs -n kube-system kube-proxy-xxxxx

# 6. Verify iptables rules for service
sudo iptables-save | grep my-service
```

**Quick Fix:**
```bash
# Recreate service
kubectl delete svc my-service
kubectl apply -f my-service.yaml

# Restart kube-proxy
kubectl delete pod -n kube-system -l k8s-app=kube-proxy
```

---

### Issue 3: DNS Resolution Failures

**Symptoms:**
```bash
kubectl exec pod-name -- nslookup kubernetes.default
# Server: 10.96.0.10
# Address: 10.96.0.10:53
# ** server can't find kubernetes.default: NXDOMAIN
```

**Solutions:**

```bash
# 1. Check CoreDNS status
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns

# 2. Verify DNS service
kubectl get svc -n kube-system kube-dns
# Should be at 10.96.0.10 (or your cluster DNS IP)

# 3. Check pod DNS configuration
kubectl exec <pod-name> -- cat /etc/resolv.conf
# Should contain: nameserver 10.96.0.10

# 4. Test DNS manually
kubectl exec <pod-name> -- nslookup kubernetes.default.svc.cluster.local 10.96.0.10

# 5. Restart CoreDNS
kubectl rollout restart deployment/coredns -n kube-system

# 6. Check for network policy blocking DNS
kubectl get networkpolicies --all-namespaces
```

**Fix CoreDNS ConfigMap:**
```bash
kubectl edit configmap coredns -n kube-system
# Ensure forward . /etc/resolv.conf is present
# Restart CoreDNS after editing
```

---

### Issue 4: Network Policy Blocking Traffic

**Symptoms:**
```bash
# Pods can't communicate after applying network policy
curl: (7) Failed to connect to service port 80: Connection refused
```

**Solutions:**

```bash
# 1. Check existing network policies
kubectl get networkpolicies --all-namespaces
kubectl describe networkpolicy <policy-name> -n <namespace>

# 2. Test without network policies
kubectl delete networkpolicy <policy-name> -n <namespace>
# Test connectivity
# If it works, the policy is too restrictive

# 3. Check policy selectors
kubectl get pods --show-labels -n <namespace>
# Ensure labels match policy selectors

# 4. Add DNS exception to egress rules
# Most applications need DNS - ensure UDP port 53 is allowed
```

**Debugging Network Policy:**
```yaml
# Temporary allow-all policy for testing
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-temp
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - {}
  egress:
  - {}
```

---

### Issue 5: NodePort Not Accessible

**Symptoms:**
```bash
curl http://<node-ip>:30080  # Connection timeout
```

**Solutions:**

```bash
# 1. Verify NodePort service
kubectl get svc -o wide
# Check nodePort value (30000-32767)

# 2. Check firewall rules
# AWS Security Group
# Azure NSG
# GCP Firewall
# On-prem: iptables/firewalld

# 3. Verify kube-proxy mode
kubectl logs -n kube-system kube-proxy-xxxxx | grep "Using"
# Should show iptables or ipvs

# 4. Check iptables rules
sudo iptables -t nat -L KUBE-NODEPORTS -n
sudo iptables -t nat -L KUBE-SERVICES -n | grep <nodePort>

# 5. Test from node itself
ssh <node-ip>
curl localhost:30080

# 6. Check if service endpoints exist
kubectl get endpoints <service-name>
```

**Cloud Provider Fixes:**

```bash
# AWS: Open security group
aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 30080 \
  --cidr 0.0.0.0/0

# GCP: Create firewall rule
gcloud compute firewall-rules create allow-nodeport \
  --allow tcp:30000-32767 \
  --source-ranges 0.0.0.0/0
```

---

### Issue 6: Pod Can't Access External Services

**Symptoms:**
```bash
kubectl exec pod-name -- curl https://google.com
# curl: (6) Could not resolve host: google.com
```

**Solutions:**

```bash
# 1. Check DNS resolution first
kubectl exec pod-name -- nslookup google.com
kubectl exec pod-name -- cat /etc/resolv.conf

# 2. Verify network policies allow egress
kubectl get networkpolicies -n <namespace>

# 3. Check NAT/masquerading
kubectl get nodes -o yaml | grep PodCIDR
sudo iptables -t nat -L POSTROUTING -n | grep MASQUERADE

# 4. Test with IP directly
kubectl exec pod-name -- curl -I 8.8.8.8

# 5. Check CNI configuration
# For Calico
kubectl get ippools -o yaml | grep natOutgoing
# Should be true

# 6. Verify node can reach external services
ssh <node-ip>
curl https://google.com
```

**Fix NAT for Calico:**
```bash
# Enable NAT outgoing
kubectl patch ippool default-ipv4-ippool -p '{"spec":{"natOutgoing":true}}'
```

---

### Issue 7: High Network Latency

**Symptoms:**
```bash
# Slow response times between pods
time kubectl exec pod-a -- curl pod-b-service  # Takes >1s
```

**Solutions:**

```bash
# 1. Check pod resource limits
kubectl top pods
kubectl describe pod <pod-name> | grep -A 5 Limits

# 2. Test network throughput
kubectl run iperf-server --image=networkstatic/iperf3 -- -s
kubectl run iperf-client --image=networkstatic/iperf3 -- -c <server-ip> -t 30

# 3. Check CNI MTU settings
kubectl exec -n kube-system <cni-pod> -- ip link show
# Typical MTU: 1450 for VXLAN, 1500 for non-overlay

# 4. Monitor node network
kubectl top nodes
ssh <node-ip>
ifconfig
netstat -i

# 5. Check DNS cache
# Increase CoreDNS cache TTL
kubectl edit configmap coredns -n kube-system
# Change: cache 30 to cache 60

# 6. Use host network for performance-critical pods
```

**Optimize CoreDNS:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        kubernetes cluster.local in-addr.arpa ip6.arpa {
           pods insecure
           fallthrough in-addr.arpa ip6.arpa
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 60  # Increased from 30
        loop
        reload
        loadbalance
    }
```

---

### Issue 8: Intermittent Connection Drops

**Symptoms:**
```bash
# Connections randomly fail
curl: (52) Empty reply from server
curl: (56) Recv failure: Connection reset by peer
```

**Solutions:**

```bash
# 1. Check pod readiness probes
kubectl describe pod <pod-name> | grep -A 10 Readiness

# 2. Look for pod restarts
kubectl get pods -w
kubectl get events --sort-by='.lastTimestamp'

# 3. Check kube-proxy sync
kubectl logs -n kube-system kube-proxy-xxxxx | grep -i error

# 4. Monitor iptables rules churn
watch -n 1 'sudo iptables -t nat -L KUBE-SERVICES | wc -l'

# 5. Check for connection tracking table overflow
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 6. Increase conntrack table size
sudo sysctl -w net.netfilter.nf_conntrack_max=1048576
```

**Persistent Fix:**
```bash
# Add to /etc/sysctl.conf
cat <<EOF | sudo tee -a /etc/sysctl.conf
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
EOF

sudo sysctl -p
```

---

## 🛠️ Network Debugging Tools

### Essential Debugging Pod

```bash
# Create a debugging pod with network tools
kubectl run netshoot --image=nicolaka/netshoot -it --rm -- bash

# Inside the pod:
# - ping, traceroute, curl, wget
# - nslookup, dig, host
# - tcpdump, netstat, ss
# - iperf3, mtr, nmap
```

### Useful Commands

```bash
# 1. Check pod network connectivity
kubectl exec <pod> -- ping -c 3 <target-ip>
kubectl exec <pod> -- curl -v http://<service>
kubectl exec <pod> -- nc -zv <host> <port>

# 2. DNS debugging
kubectl exec <pod> -- nslookup <service>.<namespace>.svc.cluster.local
kubectl exec <pod> -- dig +short <service>.<namespace>.svc.cluster.local

# 3. Check routes
kubectl exec <pod> -- ip route
kubectl exec <pod> -- traceroute <destination>

# 4. Monitor traffic
kubectl exec <pod> -- tcpdump -i any -n port 80

# 5. Check listening ports
kubectl exec <pod> -- netstat -tuln
kubectl exec <pod> -- ss -tuln

# 6. Test bandwidth
# Start server pod
kubectl run iperf-server --image=networkstatic/iperf3 --port=5201 -- -s

# Create service
kubectl expose pod iperf-server --port=5201

# Run client
kubectl run iperf-client --image=networkstatic/iperf3 --rm -it -- \
  -c iperf-server.default.svc.cluster.local -t 20
```

### CNI-Specific Debugging

**Calico:**
```bash
# Check Calico node status
kubectl exec -n calico-system calicoctl -- node status

# View routes
kubectl exec -n calico-system calicoctl -- get bgpPeer

# Check IP pools
kubectl get ippools -o yaml

# Node-to-node mesh status
calicoctl node status
```

**Flannel:**
```bash
# Check Flannel configuration
kubectl get configmap -n kube-flannel kube-flannel-cfg -o yaml

# View subnet allocation
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.podCIDR}{"\n"}{end}'
```

**Cilium:**
```bash
# Connectivity test
cilium connectivity test

# Check Cilium status
cilium status

# Monitor traffic
cilium monitor
```

---

## ✅ Best Practices

### 1. Network Design

- **Plan IP addressing carefully** - Avoid overlapping CIDRs
  ```
  Node Network:     10.0.0.0/16
  Pod Network:      192.168.0.0/16
  Service Network:  10.96.0.0/12
  ```

- **Use network policies** from day one
  ```yaml
  # Start with deny-all, then whitelist
  kind: NetworkPolicy
  metadata:
    name: default-deny-all
  spec:
    podSelector: {}
    policyTypes:
    - Ingress
    - Egress
  ```

- **Implement proper segmentation**
  - Separate namespaces for different environments
  - Use labels consistently
  - Apply policies per namespace

### 2. Service Design

- **Prefer ClusterIP** for internal communication
- **Use headless services** for StatefulSets
  ```yaml
  spec:
    clusterIP: None  # Headless
  ```
- **Implement health checks** properly
  ```yaml
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
  ```

### 3. DNS Optimization

- **Cache DNS responses** in applications
- **Use FQDN** for cross-namespace communication
  ```
  service-name.namespace.svc.cluster.local
  ```
- **Monitor CoreDNS** performance
  ```bash
  kubectl top pods -n kube-system -l k8s-app=kube-dns
  ```

### 4. Monitoring & Observability

- **Implement network monitoring**
  - Use Prometheus + Grafana
  - Monitor CNI plugin metrics
  - Track service response times

- **Set up logging**
  ```bash
  # Enable CNI plugin logging
  kubectl logs -n kube-system -l k8s-app=calico-node
  ```

- **Use tracing** for microservices
  - Implement Jaeger or Zipkin
  - Add correlation IDs

### 5. Security

- **Encrypt in-transit traffic**
  - Use service mesh (Istio, Linkerd)
  - Enable mTLS between services

- **Restrict egress traffic**
  ```yaml
  # Allow only specific external endpoints
  egress:
  - to:
    - ipBlock:
        cidr: 52.1.2.3/32  # Specific external service
  ```

- **Regular security audits**
  ```bash
  # Check for pods without network policies
  kubectl get pods --all-namespaces -o json | \
    jq '.items[] | select(.metadata.namespace != "kube-system") | 
    {namespace: .metadata.namespace, name: .metadata.name}'
  ```

### 6. Performance Tuning

- **Optimize MTU** for your network
  ```bash
  # Test optimal MTU
  ping -M do -s 1472 <target>  # Start here, adjust down if needed
  ```

- **Use host networking** for performance-critical pods
  ```yaml
  spec:
    hostNetwork: true
  ```

- **Enable IPVS** mode for kube-proxy
  ```bash
  kubectl edit configmap kube-proxy -n kube-system
  # Change mode: "" to mode: "ipvs"
  ```

### 7. Disaster Recovery

- **Document network configuration**
- **Backup CNI configurations**
  ```bash
  kubectl get cm -n kube-system -o yaml > cni-config-backup.yaml
  ```
- **Test network failover scenarios**
- **Have rollback procedures ready**

---

## 📚 Additional Resources

### Official Documentation
- [Kubernetes Networking](https://kubernetes.io/docs/concepts/cluster-administration/networking/)
- [Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Service Types](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

### CNI Plugin Documentation
- [Calico](https://docs.projectcalico.org/)
- [Flannel](https://github.com/flannel-io/flannel)
- [Cilium](https://docs.cilium.io/)
- [Weave Net](https://www.weave.works/docs/net/latest/overview/)

### Tools & Utilities
- [Netshoot](https://github.com/nicolaka/netshoot) - Network troubleshooting container
- [Kube-ops-view](https://github.com/hjacobs/kube-ops-view) - Visual cluster view
- [Goldpinger](https://github.com/bloomberg/goldpinger) - Network connectivity monitoring

### Community
- [
