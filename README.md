# E-Commerce Microservices Application

A complete microservices-based e-commerce application deployed on AWS with K3s, CI/CD, monitoring, and infrastructure as code.

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                           AWS Cloud (us-east-1)                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                         VPC (10.0.0.0/16)                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐│ │
│  │  │                   K3s Cluster (EC2)                          ││ │
│  │  │  ┌──────────┐  ┌────────────────┐  ┌─────────────────┐      ││ │
│  │  │  │ Frontend │  │ Product Service│  │  Order Service  │      ││ │
│  │  │  │ (React)  │──│   (Node.js)    │──│   (Node.js)     │      ││ │
│  │  │  │ :30080   │  │     :3001      │  │     :3002       │      ││ │
│  │  │  └──────────┘  └────────────────┘  └─────────────────┘      ││ │
│  │  │        │               │                   │                 ││ │
│  │  │  ┌─────────────────────────────────────────────────────────┐││ │
│  │  │  │     Prometheus  │  Grafana  │  Jenkins  │  CloudWatch   │││ │
│  │  │  └─────────────────────────────────────────────────────────┘││ │
│  │  └─────────────────────────────────────────────────────────────┘│ │
│  │                              │                                   │ │
│  │                    ┌─────────────────┐                          │ │
│  │                    │   RDS MySQL     │                          │ │
│  │                    │  (db.t3.micro)  │                          │ │
│  │                    └─────────────────┘                          │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

## 📁 Project Structure

```
Microservices-aws-application/
├── terraform/                    # Infrastructure as Code
│   ├── main.tf                  # Main Terraform configuration
│   ├── variables.tf             # Variable definitions
│   ├── outputs.tf               # Output values
│   ├── vpc.tf                   # VPC, subnets, NAT Gateway
│   ├── ec2-k3s.tf               # K3s EC2 instances
│   ├── security-groups.tf       # Security groups
│   ├── rds.tf                   # RDS MySQL database
│   ├── scripts/                 # K3s initialization scripts
│   └── terraform.tfvars.example # Example variables file
├── app/
│   ├── frontend/                # React frontend application
│   ├── product-service/         # Node.js product microservice
│   └── order-service/           # Node.js order microservice
├── kubernetes/
│   ├── namespace.yaml           # Kubernetes namespace
│   ├── configmap.yaml           # Application configuration
│   ├── secrets.yaml             # Sensitive data
│   ├── product-service.yaml     # Product service deployment
│   ├── order-service.yaml       # Order service deployment
│   ├── frontend.yaml            # Frontend deployment
│   ├── jenkins/                 # Jenkins deployment
│   └── monitoring/              # Prometheus, Grafana, CloudWatch
├── jenkins/
│   └── Jenkinsfile              # CI/CD pipeline
└── scripts/
    ├── deploy-all.sh            # Full deployment script
    ├── build-and-push.sh        # Build Docker images
    └── cleanup.sh               # Resource cleanup
```

## 🚀 Quick Start

### Prerequisites

- AWS CLI configured with credentials
- Terraform >= 1.0.0
- kubectl
- Docker
- Docker Hub account
- SSH key pair

### Step 1: Generate SSH Key and K3s Token

```bash
# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/k3s-key

# Generate K3s token
openssl rand -hex 32
```

### Step 2: Configure Variables

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values:
# - ssh_public_key: Content of ~/.ssh/k3s-key.pub
# - k3s_token: The token you generated
# - db_password: A secure password
```

### Step 3: Deploy Infrastructure

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

### Step 4: Wait for K3s Initialization

After Terraform completes, wait 2-3 minutes for K3s to initialize, then:

```bash
# SSH into the master node
ssh -i ~/.ssh/k3s-key ec2-user@<master-public-ip>

# Verify cluster is ready
kubectl get nodes
```

### Step 5: Build and Push Docker Images

```bash
chmod +x scripts/build-and-push.sh
./scripts/build-and-push.sh <your-dockerhub-username>
```

### Step 6: Deploy Application

SSH into the master node and apply manifests:

```bash
# On master node
kubectl apply -f kubernetes/namespace.yaml
kubectl apply -f kubernetes/configmap.yaml
kubectl apply -f kubernetes/secrets.yaml
kubectl apply -f kubernetes/product-service.yaml
kubectl apply -f kubernetes/order-service.yaml
kubectl apply -f kubernetes/frontend.yaml
```

### Step 7: Access Application

```bash
# Frontend: http://<master-public-ip>:30080
# Grafana:  http://<master-public-ip>:30090 (admin/admin123)
```

## 🔧 Services

### Frontend (React)

- Port: 80
- Endpoint: LoadBalancer URL
- Features: Product listing, cart, checkout, order history

### Product Service (Node.js)

- Port: 3001
- Endpoints:
  - `GET /api/products` - List products
  - `GET /api/products/:id` - Get product
  - `POST /api/products` - Create product
  - `PUT /api/products/:id` - Update product
  - `DELETE /api/products/:id` - Delete product
  - `GET /health` - Health check
  - `GET /metrics` - Prometheus metrics

### Order Service (Node.js)

- Port: 3002
- Endpoints:
  - `GET /api/orders` - List orders
  - `GET /api/orders/:id` - Get order
  - `POST /api/orders` - Create order
  - `PATCH /api/orders/:id/status` - Update status
  - `GET /health` - Health check
  - `GET /metrics` - Prometheus metrics

## 🖥️ Local Development

Use Docker Compose for local development:

```bash
docker-compose up -d
```

Access:

- Frontend: http://localhost
- Product Service: http://localhost:3001
- Order Service: http://localhost:3002
- Prometheus: http://localhost:9090
- Grafana: http://localhost:3000 (admin/admin123)

## 📊 Monitoring

### Prometheus

- Collects metrics from all services
- Kubernetes service discovery
- 15-second scrape interval

### Grafana

- URL: `http://<grafana-lb>:3000`
- Default credentials: admin/admin123
- Pre-configured E-Commerce dashboard

### CloudWatch

- Container insights enabled
- Custom metrics namespace: `ECommerce/Microservices`
- Log aggregation from all pods

## 🔄 CI/CD Pipeline

The Jenkins pipeline includes:

1. **Checkout** - Clone repository
2. **Build** - Build Docker images (parallel)
3. **Push** - Push to Docker Hub
4. **Deploy** - Apply Kubernetes manifests
5. **Verify** - Check deployment status

### Jenkins Setup

1. Access Jenkins at `http://<jenkins-lb>:8080`
2. Get initial admin password:
   ```bash
   kubectl exec -n jenkins $(kubectl get pods -n jenkins -o name) -- cat /var/jenkins_home/secrets/initialAdminPassword
   ```
3. Install suggested plugins
4. Add credentials:
   - Docker Hub credentials (ID: `dockerhub-credentials`)
   - AWS credentials

## 🧹 Cleanup

```bash
chmod +x scripts/cleanup.sh
./scripts/cleanup.sh
```

Or manually:

```bash
# Delete Kubernetes resources
kubectl delete namespace ecommerce
kubectl delete namespace monitoring
kubectl delete namespace jenkins

# Destroy infrastructure
cd terraform
terraform destroy
```

## ⚠️ AWS Learner Lab Notes

If using AWS Academy Learner Lab:

1. **IAM Limitations**: EKS is not supported due to IAM restrictions. This project uses K3s on EC2 instead.
2. **Session Timeout**: Lab sessions expire after 4 hours. Save your SSH key locally.
3. **Region**: Stick to us-east-1 for best compatibility
4. **Budget**: Uses minimal instance sizes (t3.medium for K3s nodes)
5. **Persistence**: RDS data persists across sessions, but you may need to re-deploy K3s
6. **Security Groups**: May need manual adjustment if Terraform fails

## 📝 API Examples

### Create Product

```bash
curl -X POST http://<product-service>/api/products \
  -H "Content-Type: application/json" \
  -d '{"name": "New Product", "price": 29.99, "stock": 100}'
```

### Create Order

```bash
curl -X POST http://<order-service>/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customer_name": "John Doe",
    "customer_email": "john@example.com",
    "shipping_address": "123 Main St",
    "items": [{"product_id": "prod-001", "quantity": 2}]
  }'
```

## 📄 License

This project is for educational purposes as part of a DevOps mini-project
