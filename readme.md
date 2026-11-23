# 2048 Game Deployment on Amazon EKS with AWS Load Balancer Controller

This guide walks you through deploying the classic **2048 game** on **Amazon EKS** (Elastic Kubernetes Service) and exposing it to the internet using the **AWS Application Load Balancer (ALB) Controller**.

---

## Architecture Overview

```
Internet
   │
   ▼
AWS Application Load Balancer (ALB)
   │
   ▼
EKS Ingress (ingress-2048)
   │
   ▼
Kubernetes Service (service-2048 — NodePort)
   │
   ▼
Kubernetes Deployment (deployment-2048 — 2 replicas)
   │
   ▼
Docker Image: public.ecr.aws/l6m2t8p7/docker-2048:latest
```

---

## Prerequisites

| Requirement | Details |
|---|---|
| AWS Account | Account with EKS, IAM, EC2 permissions |
| EKS Cluster | Cluster named `test` in `ap-south-1` |
| Node Group | At least one managed node group attached to the cluster |
| Bastion Host | EC2 instance with access to the EKS cluster |
| Tools on Bastion | `kubectl`, `aws cli v2`, `helm` installed |

---

## Step 1 — Enable OIDC Provider for the EKS Cluster

The ALB Controller uses **IAM Roles for Service Accounts (IRSA)**. An OIDC provider must be associated with your cluster.

### 1.1 Get the OIDC URL from EKS Console

1. Open the [EKS Console](https://ap-south-1.console.aws.amazon.com/eks/clusters?region=ap-south-1)
2. Click on your cluster **test**
3. Go to **Configuration** tab → **Authentication** section
4. Copy the **OpenID Connect provider URL**
   - Example: `https://oidc.eks.ap-south-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE`

### 1.2 Register the OIDC Provider in IAM

1. Open the [IAM Console](https://console.aws.amazon.com/iam/home#/providers)
2. Click **Identity Providers** → **Add provider**
3. Select **OpenID Connect**
4. Paste the OIDC URL in **Provider URL** → click **Get thumbprint**
5. Set **Audience** to `sts.amazonaws.com`
6. Click **Add provider**

---

## Step 2 — Create IAM Policy and Role for ALB Controller

### 2.1 Create the IAM Policy

1. Download the policy document:
   ```bash
   curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.2/docs/install/iam_policy.json
   ```
2. Open the [IAM Console](https://console.aws.amazon.com/iam/home#/policies) → **Create policy**
3. Switch to the **JSON** tab → paste the contents of `iam_policy.json`
4. Name the policy: `AWSLoadBalancerControllerIAMPolicy`
5. Click **Create policy**

### 2.2 Create the IAM Role (Web Identity / IRSA)

1. Open [IAM Console](https://console.aws.amazon.com/iam/home#/roles) → **Create role**
2. Select **Web identity** as the trusted entity type
3. Choose your cluster's OIDC provider from the dropdown
4. Set **Audience** to `sts.amazonaws.com`
5. Add a condition:
   - **Key:** `oidc.eks.ap-south-1.amazonaws.com/id/<YOUR_OIDC_ID>:sub`
   - **Condition:** `StringEquals`
   - **Value:** `system:serviceaccount:kube-system:aws-load-balancer-controller`
6. Click **Next** → attach the `AWSLoadBalancerControllerIAMPolicy` policy
7. Name the role: `AmazonEKSLoadBalancerControllerRole`
8. Click **Create role**
9. **Copy the Role ARN** — you will need it in Step 3

---

## Step 3 — Connect Bastion to EKS Cluster

SSH into your bastion host and run:

```bash
# Configure kubeconfig
aws eks update-kubeconfig --region ap-south-1 --name test

# Verify cluster connectivity
kubectl get nodes
```

**Expected output:**
```
NAME                                        STATUS   ROLES    AGE   VERSION
ip-10-0-x-x.ap-south-1.compute.internal    Ready    <none>   5m    v1.29.x
```

> Wait for the cluster to be in **Active** state and nodes to show **Ready** before proceeding.

---

## Step 4 — Install AWS Load Balancer Controller

### 4.1 Get your VPC ID

1. Open [EKS Console](https://ap-south-1.console.aws.amazon.com/eks/clusters/test?region=ap-south-1) → cluster **test**
2. Click the **Networking** tab
3. Copy the **VPC ID** (e.g., `vpc-0abc1234def567890`)

### 4.2 Create the Kubernetes ServiceAccount

Replace `<ROLE_ARN>` with the ARN copied in Step 2.2:

```bash
kubectl apply -f - <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
  annotations:
    eks.amazonaws.com/role-arn: <ROLE_ARN>
EOF
```

### 4.3 Install cert-manager (required dependency)

```bash
kubectl apply --validate=false \
  -f https://github.com/jetstack/cert-manager/releases/download/v1.13.5/cert-manager.yaml

# Wait for cert-manager pods to be Running
kubectl get pods -n cert-manager
```

### 4.4 Install the ALB Controller via Helm

Replace `<YOUR_VPC_ID>` with the VPC ID from Step 4.1:

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=test \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=<YOUR_VPC_ID>
```

### 4.5 Verify the controller is running

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

**Expected output:**
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           1m
```

---

## Step 5 — Deploy the 2048 Game

Apply the full manifest (Namespace + Deployment + Service + Ingress):

```bash
kubectl apply -f - <<EOF
---
apiVersion: v1
kind: Namespace
metadata:
  name: game-2048
---
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: game-2048
  name: deployment-2048
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: app-2048
  replicas: 2
  template:
    metadata:
      labels:
        app.kubernetes.io/name: app-2048
    spec:
      containers:
      - image: public.ecr.aws/l6m2t8p7/docker-2048:latest
        imagePullPolicy: Always
        name: app-2048
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  namespace: game-2048
  name: service-2048
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
  type: NodePort
  selector:
    app.kubernetes.io/name: app-2048
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  namespace: game-2048
  name: ingress-2048
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: service-2048
            port:
              number: 80
EOF
```

---

## Step 6 — Verify Deployment and Access the Game

### 6.1 Check all resources

```bash
# Check pods are Running
kubectl get pods -n game-2048

# Check service
kubectl get svc -n game-2048

# Check ingress — wait for ADDRESS to be populated (2-5 minutes)
kubectl get ingress -n game-2048
```

### 6.2 Get the ALB DNS name

```bash
kubectl get ingress ingress-2048 -n game-2048 \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Example output:**
```
k8s-game2048-ingress2-xxxxxxxxxxxx.ap-south-1.elb.amazonaws.com
```

Open this URL in your browser — the 2048 game will load.

### 6.3 Verify ALB in AWS Console

1. Open [EC2 → Load Balancers](https://ap-south-1.console.aws.amazon.com/ec2/home?region=ap-south-1#LoadBalancers:)
2. A new ALB named `k8s-game2048-...` should be in **Active** state
3. Under **Target groups**, verify all targets are **healthy**

---

## Troubleshooting

| Issue | Command / Fix |
|---|---|
| Pods not starting | `kubectl describe pod -n game-2048 <pod-name>` |
| ALB not created | `kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller` |
| Targets unhealthy | Allow inbound TCP port 80 on node security group from ALB security group |
| OIDC error on IRSA | Verify OIDC provider ARN matches the trust policy in the IAM role |
| Nodes not Ready | Check node group status: EKS Console → cluster → **Compute** tab |
| `kubectl` unauthorized | Re-run `aws eks update-kubeconfig --region ap-south-1 --name test` |

---

## Cleanup

To delete all resources and avoid ongoing charges:

```bash
# Delete the game application
kubectl delete namespace game-2048

# Uninstall the ALB Controller
helm uninstall aws-load-balancer-controller -n kube-system

# Delete cert-manager
kubectl delete -f https://github.com/jetstack/cert-manager/releases/download/v1.13.5/cert-manager.yaml

# Delete the ServiceAccount
kubectl delete serviceaccount aws-load-balancer-controller -n kube-system
```

Then delete the IAM Role and Policy from the [IAM Console](https://console.aws.amazon.com/iam/home).

---

## References

- [AWS Load Balancer Controller Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [EKS Workshop — 2048 Game](https://www.eksworkshop.com/)
- [cert-manager Releases](https://github.com/jetstack/cert-manager/releases)
