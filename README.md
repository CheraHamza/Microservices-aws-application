# E-Commerce Microservices Application

A complete microservices-based e-commerce application deployed on AWS with K3s, monitoring (Prometheus/Grafana), and Infrastructure as Code (Terraform).

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                           AWS Cloud (us-east-1)                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │                         VPC (10.0.0.0/16)                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐│ │
│  │  │                   K3s Cluster (3 EC2 nodes)                  ││ │
│  │  │  ┌──────────┐  ┌────────────────┐  ┌─────────────────┐      ││ │
│  │  │  │ Frontend │  │ Product Service│  │  Order Service  │      ││ │
│  │  │  │ (React)  │──│   (Node.js)    │──│   (Node.js)     │      ││ │
│  │  │  │   :80    │  │     :3001      │  │     :3002       │      ││ │
│  │  │  └──────────┘  └────────────────┘  └─────────────────┘      ││ │
│  │  │        │               │                   │                 ││ │
│  │  │  ┌─────────────────────────────────────────────────────────┐││ │
│  │  │  │     Prometheus      │      Grafana      │    Jenkins    │││ │
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

## 🛠️ Tech Stack

| Component            | Technology                     |
| -------------------- | ------------------------------ |
| **Infrastructure**   | Terraform, AWS (EC2, VPC, RDS) |
| **Kubernetes**       | K3s (lightweight K8s)          |
| **Frontend**         | React 18, Nginx                |
| **Backend**          | Node.js 18, Express            |
| **Database**         | MySQL 8.0 (AWS RDS)            |
| **Monitoring**       | Prometheus, Grafana            |
| **CI/CD**            | Jenkins                        |
| **Containerization** | Docker, Docker Hub             |

## 📁 Project Structure

```
Microservices-aws-application/
├── terraform/                    # Infrastructure as Code
│   ├── main.tf                  # Provider configuration
│   ├── variables.tf             # Variable definitions
│   ├── outputs.tf               # Output values
│   ├── vpc.tf                   # VPC, subnets, NAT Gateway
│   ├── ec2-k3s.tf               # K3s EC2 instances & security groups
│   ├── security-groups.tf       # RDS security groups
│   ├── rds.tf                   # RDS MySQL database
│   └── terraform.tfvars.example # Example variables file
├── app/
│   ├── frontend/                # React frontend application
│   ├── product-service/         # Node.js product microservice
│   └── order-service/           # Node.js order microservice
├── kubernetes/
│   ├── namespace.yaml           # Kubernetes namespace
│   ├── configmap.yaml           # Application configuration
│   ├── secrets.yaml             # Database credentials
│   ├── product-service.yaml     # Product service deployment
│   ├── order-service.yaml       # Order service deployment
│   ├── frontend.yaml            # Frontend deployment
│   ├── jenkins/                 # Jenkins deployment
│   └── monitoring/              # Prometheus & Grafana
├── jenkins/
│   └── Jenkinsfile              # CI/CD pipeline definition
├── docker-compose.yml           # Local development
└── README.md
```

## 🚀 Deployment Guide

### Prerequisites

- AWS Account (or AWS Academy Learner Lab)
- AWS CLI configured
- Terraform >= 1.0.0
- Docker & Docker Hub account
- SSH key pair

### Step 1: Clone and Configure

```bash
git clone https://github.com/CheraHamza/Microservices-aws-application.git
cd Microservices-aws-application

# Generate SSH key
ssh-keygen -t rsa -b 4096 -f ~/.ssh/k3s-key

# Configure Terraform variables
cd terraform
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars with your values
```

### Step 2: Deploy AWS Infrastructure

```bash
cd terraform
terraform init
terraform plan
terraform apply
```

This creates:

- VPC with public/private subnets
- 3 EC2 instances (1 master, 2 workers)
- RDS MySQL database
- Security groups

### Step 3: Install K3s on Master Node

```bash
# SSH into master
ssh -i ~/.ssh/k3s-key ec2-user@<MASTER_PUBLIC_IP>

# Install K3s (skip SELinux for Amazon Linux 2)
curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_SELINUX_RPM=true sh -s - server \
    --write-kubeconfig-mode=644 \
    --disable=traefik \
    --node-name="k3s-master"

# Setup kubectl
sudo mkdir -p /home/ec2-user/.kube
sudo cp /etc/rancher/k3s/k3s.yaml /home/ec2-user/.kube/config
sudo chown -R ec2-user:ec2-user /home/ec2-user/.kube
export KUBECONFIG=/home/ec2-user/.kube/config
echo 'export KUBECONFIG=/home/ec2-user/.kube/config' >> ~/.bashrc

# Get node token for workers
sudo cat /var/lib/rancher/k3s/server/node-token
```

### Step 4: Join Worker Nodes

SSH into each worker and run:

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_SELINUX_RPM=true \
    K3S_URL=https://<MASTER_PRIVATE_IP>:6443 \
    K3S_TOKEN="<NODE_TOKEN>" sh -
```

Verify on master:

```bash
kubectl get nodes
# Should show 3 nodes: k3s-master + 2 workers
```

### Step 5: Create Database Tables

```bash
# On master node, connect to RDS
mysql -h <RDS_ENDPOINT> -P 3306 -u <DB_USER> -p<DB_PASSWORD> <DB_NAME>
```

```sql
CREATE TABLE products (
    id VARCHAR(36) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INT DEFAULT 0,
    category VARCHAR(100),
    image_url VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id VARCHAR(36) PRIMARY KEY,
    customer_name VARCHAR(255) NOT NULL,
    customer_email VARCHAR(255) NOT NULL,
    shipping_address TEXT,
    total_amount DECIMAL(10, 2) NOT NULL,
    status ENUM('pending', 'confirmed', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id VARCHAR(36) PRIMARY KEY,
    order_id VARCHAR(36) NOT NULL,
    product_id VARCHAR(36) NOT NULL,
    quantity INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

-- Insert sample products
INSERT INTO products (id, name, description, price, stock, category) VALUES
('prod-001', 'Wireless Headphones', 'High-quality wireless headphones', 99.99, 50, 'Electronics'),
('prod-002', 'Smart Watch', 'Fitness tracking smart watch', 199.99, 30, 'Electronics'),
('prod-003', 'Laptop Stand', 'Ergonomic aluminum laptop stand', 49.99, 100, 'Accessories');
```

### Step 6: Build and Push Docker Images

```bash
# On your local machine
docker login

docker build -t <DOCKERHUB_USER>/ecommerce-frontend:latest ./app/frontend
docker build -t <DOCKERHUB_USER>/product-service:latest ./app/product-service
docker build -t <DOCKERHUB_USER>/order-service:latest ./app/order-service

docker push <DOCKERHUB_USER>/ecommerce-frontend:latest
docker push <DOCKERHUB_USER>/product-service:latest
docker push <DOCKERHUB_USER>/order-service:latest
```

### Step 7: Update Kubernetes Manifests

Edit `kubernetes/secrets.yaml` with your RDS endpoint:

```yaml
stringData:
  DB_HOST: "<RDS_ENDPOINT>"
  DB_USER: "<DB_USER>"
  DB_PASSWORD: "<DB_PASSWORD>"
```

Update image names in `kubernetes/*.yaml` files with your Docker Hub username.

### Step 8: Deploy to Kubernetes

```bash
# Copy manifests to master
scp -i ~/.ssh/k3s-key -r ./kubernetes ec2-user@<MASTER_IP>:~/

# SSH into master and deploy
ssh -i ~/.ssh/k3s-key ec2-user@<MASTER_IP>

kubectl create namespace ecommerce
kubectl create namespace monitoring
kubectl create namespace jenkins

kubectl apply -f ~/kubernetes/secrets.yaml
kubectl apply -f ~/kubernetes/configmap.yaml
kubectl apply -f ~/kubernetes/product-service.yaml
kubectl apply -f ~/kubernetes/order-service.yaml
kubectl apply -f ~/kubernetes/frontend.yaml
kubectl apply -f ~/kubernetes/monitoring/
kubectl apply -f ~/kubernetes/jenkins/

# Verify
kubectl get pods --all-namespaces
```

### Step 9: Access the Application

| Service      | URL                                                |
| ------------ | -------------------------------------------------- |
| **Frontend** | http://MASTER_IP                             |
| **Grafana**  | http://MASTER_IP:NodePort (admin/admin123) |
| **Jenkins**  | http://MASTER_IP:NodePort                          |

Get NodePorts:

```bash
kubectl get svc --all-namespaces
```

## 🔧 API Endpoints

### Product Service (Port 3001)

| Endpoint            | Method | Description        |
| ------------------- | ------ | ------------------ |
| `/api/products`     | GET    | List all products  |
| `/api/products/:id` | GET    | Get single product |
| `/api/products`     | POST   | Create product     |
| `/health`           | GET    | Health check       |
| `/metrics`          | GET    | Prometheus metrics |

### Order Service (Port 3002)

| Endpoint          | Method | Description        |
| ----------------- | ------ | ------------------ |
| `/api/orders`     | GET    | List all orders    |
| `/api/orders/:id` | GET    | Get single order   |
| `/api/orders`     | POST   | Create order       |
| `/health`         | GET    | Health check       |
| `/metrics`        | GET    | Prometheus metrics |

## 📊 Monitoring Setup

### Grafana Configuration

1. Go to **Connections** → **Data Sources** → **Add data source**
2. Select **Prometheus**
3. URL: `http://prometheus:9090`
4. Click **Save & Test**
5. Import dashboard ID: **3662** for Prometheus overview

## 🧹 Cleanup

```bash
# Delete Kubernetes resources
kubectl delete namespace ecommerce
kubectl delete namespace monitoring
kubectl delete namespace jenkins

# Destroy AWS infrastructure
cd terraform
terraform destroy
```

## ⚠️ AWS Learner Lab Notes

- **EKS not available**: IAM restrictions prevent EKS usage. K3s on EC2 is the alternative.
- **Session timeout**: Labs expire after 4 hours of inactivity
- **Region**: Use us-east-1 for compatibility
- **Costs**: Uses t3.medium instances to minimize costs
- **RDS persistence**: Database persists across sessions
