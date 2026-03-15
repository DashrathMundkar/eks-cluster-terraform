# EKS + Istio on AWS (Terraform)

This project provisions an AWS EKS cluster and networking with Terraform, then deploys NGINX with Istio routing manifests.

## Project structure

```text
.
├── terraform/
│   ├── providers.tf
│   ├── variables.tf
│   ├── vpc.tf
│   ├── eks.tf
│   ├── outputs.tf
│   └── terraform.tfvars
└── k8s/
    ├── nginx.yaml
    ├── istio-ingressgateway.yaml
    ├── gateway.yaml
    └── virtual-service.yaml
```

## 1) Prerequisites

Install and configure:

- Terraform >= 1.5
- AWS CLI v2
- kubectl
- Access to an AWS account with permissions to create VPC, IAM, and EKS resources
- AWS CLI profile used in `terraform/providers.tf`

Verify tools:

```bash
terraform version
aws --version
kubectl version --client
```

## 2) Review Terraform inputs

Open `terraform/terraform.tfvars` and confirm values:

- `vpc_cidr_range`
- `private_subnet_cidr`
- `public_subnet_cidr`
- `aws_region`
- `cluster_name`
- `cluster_version`

If needed, update values before deployment.

## 3) Deploy infrastructure with Terraform

```bash
cd terraform
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

When apply completes, Terraform outputs `cluster_endpoint` from `terraform/outputs.tf`.

## 4) Configure kubeconfig for the new cluster

Run this from the repository root (replace placeholders from `terraform.tfvars`):

```bash
aws eks update-kubeconfig \
  --region <aws_region> \
  --name <cluster_name>
```

Quick check:

```bash
kubectl get nodes
```

## 5) Deploy Istio ingress components and routing manifests using NLB

## Steps

### 5.1. Add and update the Istio Helm repository:

```bash
helm repo add istio https://istio-release.storage.googleapis.com/charts
helm repo update
```

### 5.2. Create the Istio system namespace and install core components:

```bash
kubectl create namespace istio-system
helm install istio-base istio/base -n istio-system
helm install istiod istio/istiod -n istio-system
kubectl get pods -n istio-system
```

### 5.3. Install the Istio ingress gateway:

```bash
helm install istio-ingressgateway istio/gateway -n istio-system
kubectl get pods -n istio-system
kubectl get svc -n istio-system istio-ingressgateway
```
you should see the `istio-ingressgateway` service of type `LoadBalancer` with an external IP pending. This is where we will add the NLB annotation to ensure it provisions an NLB instead of a classic ELB.

Now add the NLB annotation to the `istio-ingressgateway` service:

```
kubectl patch svc istio-ingressgateway -n istio-system -p '{"metadata": {"annotations": {"service.beta.kubernetes.io/aws-load-balancer-type": "nlb"}}}'
```

### 5.4 Update your domain placeholders

Edit both files and replace `ADD YOUR DOMAIN NAME HERE` with your real host:

- `k8s/gateway.yaml`
- `k8s/virtual-service.yaml`

### 5.5 Create app namespace

`k8s/nginx.yaml` uses namespace `nginx`, so create it first:

```bash
kubectl create namespace nginx
```

### 5.6 Apply manifests in order

```bash
kubectl apply -f k8s/nginx.yaml
kubectl apply -f k8s/istio-ingressgateway.yaml
kubectl apply -f k8s/gateway.yaml
kubectl apply -f k8s/virtual-service.yaml
```

## 6) Verify deployment

Check workloads and services:

```bash
kubectl get pods -n nginx
kubectl get svc -n nginx
kubectl get svc -n istio-system istio-ingressgateway
kubectl get gateway -n istio-system
kubectl get virtualservice -n istio-system
```

Get load balancer hostname:

```bash
kubectl get svc istio-ingressgateway -n istio-system -o wide
```

Then test route (after DNS points to the LB):

```bash
curl http://<your-domain>/myapp
```

## 7) Destroy infrastructure (when done)

```bash
cd terraform
terraform destroy
```

## Architecture diagram

- SVG: `arch.png`