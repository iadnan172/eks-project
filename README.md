# Static Site Deployment on AWS EKS

A production-style deployment pipeline that takes a static website from a GitHub repo to a publicly accessible URL, running on a Kubernetes cluster on AWS EKS with EC2 worker nodes, fronted by an Application Load Balancer.

## Architecture

```
GitHub repo → EC2 (build) → ECR (image storage) → EKS cluster (EC2 worker nodes) → ALB → Public URL
```

- **EC2** — used as a build machine to clone the repo and build the Docker image (also doubles as the EKS worker node type)
- **ECR** — private container registry storing the built image
- **EKS** — managed Kubernetes control plane; worker nodes run the site as pods
- **AWS Load Balancer Controller** — watches Ingress resources and provisions an ALB automatically
- **ALB (Application Load Balancer)** — public entry point routing traffic into the cluster

![Architecture Diagram](https://github.com/iadnan172/eks-project/blob/main/images/architecture.png.png?raw=true)
## Tech Stack

- AWS: EC2, EKS, ECR, IAM (IRSA), VPC, ALB
- Kubernetes: Deployment, Service, Ingress
- Docker (nginx:alpine base image)
- eksctl, kubectl, Helm

## Prerequisites

- AWS CLI configured (`aws sts get-caller-identity` should work)
- Docker, `eksctl`, `kubectl`, and `helm` installed
- IAM permissions to create EKS clusters, EC2 instances, IAM roles/policies, and ELB resources

## Deployment Steps

### 1. Dockerize the static site

```dockerfile
FROM nginx:alpine
COPY . /usr/share/nginx/html
EXPOSE 80
```

Build and test locally:
```bash
docker build -t static-site .
docker run -d -p 8080:80 static-site
curl localhost:8080
```

### 2. Push the image to ECR

```bash
aws ecr create-repository --repository-name static-site --region ap-south-1

aws ecr get-login-password --region ap-south-1 | \
  docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-south-1.amazonaws.com

docker tag static-site:latest <account-id>.dkr.ecr.ap-south-1.amazonaws.com/static-site:latest
docker push <account-id>.dkr.ecr.ap-south-1.amazonaws.com/static-site:latest
```

### 3. Create the EKS cluster

```bash
eksctl create cluster \
  --name static-site-cluster \
  --region ap-south-1 \
  --nodegroup-name static-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed

aws eks update-kubeconfig --region ap-south-1 --name static-site-cluster
kubectl get nodes
```

`eksctl` provisions the VPC, subnets, security groups, IAM roles, and node group under the hood via CloudFormation.

### 4. Deploy the app (Deployment + Service)

```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl get svc
```

### 5. Install the AWS Load Balancer Controller

```bash
eksctl utils associate-iam-oidc-provider --cluster static-site-cluster --approve --region ap-south-1

curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

eksctl create iamserviceaccount \
  --cluster=static-site-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region ap-south-1

helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=static-site-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=<vpc-id>

kubectl get deployment -n kube-system aws-load-balancer-controller
```

### 6. Expose the app with an Ingress (ALB)

```bash
kubectl apply -f ingress.yaml
kubectl get ingress static-site-ingress
```

The `ADDRESS` column will populate with the ALB's public DNS name within 1-2 minutes.

## Kubernetes Manifests

**deployment.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: static-site
spec:
  replicas: 2
  selector:
    matchLabels:
      app: static-site
  template:
    metadata:
      labels:
        app: static-site
    spec:
      containers:
      - name: static-site
        image: <account-id>.dkr.ecr.ap-south-1.amazonaws.com/static-site:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: static-site-svc
spec:
  selector:
    app: static-site
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

**ingress.yaml**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: static-site-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: static-site-svc
            port:
              number: 80
```

## Issue Faced & Fixed

**403 AccessDenied on `elasticloadbalancing:DescribeListenerAttributes`**

The AWS Load Balancer Controller's IAM policy (from the v2.7.2 install docs) was missing a few newer permissions required by the controller version deployed, causing ALB provisioning to fail silently on the Ingress reconcile loop.

Fix — attached a small inline policy with the missing actions to the controller's IAM role:
```bash
aws iam put-role-policy \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --policy-name AWSLoadBalancerControllerExtraPermissions \
  --policy-document file://extra-permission.json
```
followed by a rollout restart of the controller deployment to pick up the new permissions.

## Cleanup

To avoid ongoing EC2/EKS charges after testing:
```bash
kubectl delete -f ingress.yaml
kubectl delete -f deployment.yaml
eksctl delete cluster --name static-site-cluster --region ap-south-1
```

## Author

Adnan Pathan — DevOps Engineer
[Portfolio](https://adnanpathan-portfolio.netlify.app) · [GitHub](https://github.com/iadnan172)# eks-project
