# Cinema Microservices - Kubernetes Deployment

A comprehensive microservices architecture for a cinema application, deployed on a custom Kubernetes cluster using kubeadm, with complete observability and GitOps workflow support.

## 📋 Project Overview

This project demonstrates a production-ready microservices architecture based on the [Spring Boot Microservices](https://github.com/yigiterinc/spring-boot-microservices) template. The entire application is containerized with Docker and orchestrated using Kubernetes, featuring enterprise-grade monitoring and deployment automation.

### Key Components

- **Microservices**: Discovery Server, Movie Catalog, Movie Info, and Ratings Data services
- **Container Registry**: Docker images built and pushed to a private repository
- **Kubernetes Cluster**: 3-node cluster (1 Master, 2 Workers) provisioned with kubeadm
- **Monitoring**: Prometheus & Grafana for comprehensive observability
- **GitOps**: ArgoCD for declarative, automated deployments

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│      Kubernetes Cluster (kubeadm)          │
│                                              │
│  ┌──────────────┐  ┌──────────────┐        │
│  │   Master     │  │   Worker 1   │        │
│  │   Node       │  │              │        │
│  └──────────────┘  └──────────────┘        │
│                                              │
│  ┌──────────────────────────────────┐      │
│  │         Worker 2                  │      │
│  └──────────────────────────────────┘      │
│                                              │
│  ┌─────────────────────────────────────┐   │
│  │   Microservices                     │   │
│  │  • Discovery Server                │   │
│  │  • Movie Catalog Service           │   │
│  │  • Movie Info Service              │   │
│  │  • Ratings Data Service            │   │
│  └─────────────────────────────────────┘   │
│                                              │
│  ┌─────────────────────────────────────┐   │
│  │   Observability Stack               │   │
│  │  • Prometheus (Monitoring)          │   │
│  │  • Grafana (Visualization)          │   │
│  └─────────────────────────────────────┘   │
│                                              │
│  ┌─────────────────────────────────────┐   │
│  │   ArgoCD (GitOps/CD)                │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

## 🚀 Prerequisites

### System Requirements

- Linux server or VM with SSH access
- Minimum resources per node:
  - Master: 2 CPU, 4GB RAM
  - Workers: 2 CPU, 4GB RAM each
- Docker installed on all nodes
- `kubeadm`, `kubelet`, and `kubectl` v1.28+ installed
- Network connectivity between all nodes (no firewall restrictions on pod CIDR range)

### Additional Tools

- Helm 3.x (for package management)
- kubectl configured with cluster access
- Docker credentials configured (for private registry access)

## 📦 Repository Structure

```
cinema-microservice-kubernetes/
├── README.md
├── k8s/
│   ├── deployments/
│   │   ├── discovery-deployment.yaml
│   │   ├── movie-catalog-deployment.yaml
│   │   ├── movie-info-deployment.yaml
│   │   └── ratings-data-deployment.yaml
│   └── services/
│       ├── discovery-service.yaml
│       ├── movie-catalog-service.yaml
│       ├── movie-info-service.yaml
│       └── ratings-data-service.yaml
```

Each microservice has:
- **Deployment**: Pod specifications, container images, resource limits, and replicas
- **Service**: Kubernetes service for internal/external access and load balancing

## 🔧 Installation & Deployment

### Step 1: Cluster Initialization (kubeadm)

On the **master node**, initialize the cluster:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --apiserver-advertise-address=<MASTER_IP>
```

Configure kubectl access:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Retrieve the join token for worker nodes:

```bash
kubeadm token create --print-join-command
```

On each **worker node**, join the cluster:

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
```

### Step 2: Install Container Network Interface (CNI)

Deploy Flannel or Calico:

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Verify all nodes are ready:

```bash
kubectl get nodes
```

All nodes should show `STATUS Ready`.

### Step 3: Deploy Microservices

Update image references in deployment files to point to your private Docker registry:

```yaml
image: your-registry/cinema-discovery:latest
image: your-registry/cinema-movie-catalog:latest
image: your-registry/cinema-movie-info:latest
image: your-registry/cinema-ratings-data:latest
```

Deploy all services:

```bash
kubectl apply -f k8s/
```

Verify deployments:

```bash
kubectl get deployments
kubectl get pods
kubectl get svc
```

### Step 4: Install Prometheus & Grafana

Using Helm (recommended):

```bash
# Add Prometheus community Helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# Install Prometheus
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

# Get Grafana admin password
kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```

Access Grafana:

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
# Navigate to http://localhost:3000
# Default credentials: admin / <generated-password>
```

### Step 5: Install ArgoCD

Install ArgoCD with Helm:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --set server.service.type=LoadBalancer
```

Retrieve ArgoCD admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
```

Access ArgoCD UI:

```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
# Navigate to https://localhost:8080
# Username: admin, Password: <retrieved-password>
```

## 🎬 Microservices Overview

### Discovery Server
- **Purpose**: Service registry for all microservices (Eureka)
- **Port**: 8761
- **Endpoint**: `http://discovery-service:8761`

### Movie Catalog Service
- **Purpose**: Manages movie catalog and information aggregation
- **Port**: 8080
- **Endpoints**: `/movies`, `/movies/{id}`

### Movie Info Service
- **Purpose**: Provides detailed movie information
- **Port**: 8081
- **Endpoints**: `/info/{movieId}`

### Ratings Data Service
- **Purpose**: Manages user ratings and reviews
- **Port**: 8082
- **Endpoints**: `/ratings/{movieId}`

## 📊 Monitoring

### Prometheus
- **Purpose**: Metrics collection and time-series database
- **Scrape interval**: 30 seconds
- **Retention**: 15 days
- **Namespace**: `monitoring`

### Grafana
- **Purpose**: Visualization and alerting dashboards
- **Pre-installed dashboards**: Kubernetes cluster monitoring, node exporter
- **Custom dashboards**: Configure for microservices metrics
- **Namespace**: `monitoring`

Access metrics endpoint: `http://prometheus-operated:9090`

### Important Metrics

Monitor these key metrics for production readiness:

```
# Pod metrics
container_memory_usage_bytes
container_cpu_usage_seconds_total

# HTTP request metrics (exposed by Spring Boot actuator)
http_requests_total
http_request_duration_seconds

# JVM metrics
jvm_memory_used_bytes
jvm_threads_live
```

## 🔄 GitOps with ArgoCD

### Workflow

1. **Git Repository**: Changes pushed to your k8s folder repository
2. **ArgoCD Detection**: ArgoCD continuously watches the repository
3. **Automatic Sync**: Detects changes and automatically syncs to cluster
4. **Reconciliation**: Ensures cluster state matches desired state in Git

### Create ArgoCD Application

```bash
kubectl apply -f - <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cinema-microservices
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-username/cinema-microservices-k8s.git
    targetRevision: main
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### Enable Auto-Sync

ArgoCD will automatically deploy changes when you push to the `k8s/` folder in your repository.

## 🐳 Docker Image Build & Push

### Build Images

```bash
docker build -t your-registry/cinema-discovery:latest -f Dockerfile.discovery .
docker build -t your-registry/cinema-movie-catalog:latest -f Dockerfile.movie-catalog .
docker build -t your-registry/cinema-movie-info:latest -f Dockerfile.movie-info .
docker build -t your-registry/cinema-ratings-data:latest -f Dockerfile.ratings-data .
```

### Push to Registry

```bash
docker login your-registry

docker push your-registry/cinema-discovery:latest
docker push your-registry/cinema-movie-catalog:latest
docker push your-registry/cinema-movie-info:latest
docker push your-registry/cinema-ratings-data:latest
```

## 📝 Common Commands

### Cluster Management

```bash
# Check cluster status
kubectl cluster-info
kubectl get nodes
kubectl get node -o wide

# Check all resources
kubectl get all
kubectl get all -A (all namespaces)

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs <pod-name> --tail=50 -f (follow logs)

# Describe resources
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>

# Scale deployments
kubectl scale deployment <deployment-name> --replicas=3
```

### Debugging

```bash
# Execute commands in pod
kubectl exec -it <pod-name> -- /bin/bash

# Port forward for local access
kubectl port-forward svc/<service-name> 8080:8080

# Check resource usage
kubectl top nodes
kubectl top pods

# Check events
kubectl get events --sort-by='.lastTimestamp'
```

### Update Images

```bash
# Update deployment with new image
kubectl set image deployment/<deployment-name> \
  <container-name>=your-registry/new-image:tag

# Or edit deployment directly
kubectl edit deployment <deployment-name>
```

## 🔐 Security Considerations

### Production Checklist

- [ ] Use private Docker registry with authentication
- [ ] Implement RBAC (Role-Based Access Control)
- [ ] Enable network policies for pod-to-pod communication
- [ ] Use secrets for sensitive data (database passwords, API keys)
- [ ] Configure resource limits and requests for all pods
- [ ] Enable pod security policies
- [ ] Set up ingress controller with TLS/SSL
- [ ] Enable audit logging on etcd
- [ ] Regularly backup etcd data

### Create ImagePullSecret

```bash
kubectl create secret docker-registry private-registry \
  --docker-server=your-registry \
  --docker-username=username \
  --docker-password=password \
  --docker-email=email@example.com
```

Update deployments to use this secret:

```yaml
imagePullSecrets:
  - name: private-registry
```

## 🚨 Troubleshooting

### Pods stuck in Pending state

```bash
kubectl describe pod <pod-name>
# Check for resource constraints or image pull errors
```

### Nodes not ready

```bash
kubectl describe node <node-name>
# Check node conditions and disk space
sudo systemctl status kubelet
```

### Service discovery issues

```bash
kubectl exec -it <pod-name> -- nslookup <service-name>
# Verify DNS resolution within cluster
```

### Prometheus not scraping metrics

```bash
# Check ServiceMonitor configuration
kubectl get servicemonitor -A
# Verify labels match in Prometheus scrape config
kubectl get prometheus -o yaml
```

## 📚 Useful Resources

- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [kubeadm Setup Guide](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)
- [Spring Boot on Kubernetes](https://spring.io/guides/gs/spring-boot-docker/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Original Spring Boot Microservices Project](https://github.com/yigiterinc/spring-boot-microservices)

## 📄 License

This project is based on the Spring Boot Microservices template. Please refer to the original repository for license information.

## 👤 Author

Cinema Microservices - Kubernetes Edition  
Created as a demonstration of modern microservices architecture with enterprise-grade DevOps practices.

---

**Last Updated**: January 2026  
**Kubernetes Version**: 1.28+  
**Status**: Production Ready
