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
| Bastion Host | EC2 instance (Amazon Linux 2 / Ubuntu) with internet access |
| IAM Role on Bastion | EC2 instance role with `eks:DescribeCluster` permission |

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
7. Name the role: `2048iam`
8. Click **Create role**
9. **Copy the Role ARN** — you will need it in Step 5

---

## Step 3 — Install Required Software on Bastion Host

SSH into your bastion host and install the following tools.

### 3.1 Install kubectl (v1.29)

```bash
curl -o kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

**Expected output:**
```
Client Version: v1.29.0
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

### 3.2 Install AWS CLI v2

```bash
sudo apt install unzip -y
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
```

**Expected output:**
```
aws-cli/2.x.x Python/3.x.x Linux/x86_64
```

### 3.3 Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Verify installation
helm version
```

**Expected output:**
```
version.BuildInfo{Version:"v3.x.x", ...}
```

> **Note:** The script URL uses `get-helm-3` (not `get-helm-4`) — this installs the latest stable Helm 3 release.

---

## Step 4 — Connect Bastion to EKS Cluster

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

## Step 5 — Install AWS Load Balancer Controller

### 5.1 Create the Kubernetes ServiceAccount

```bash
kubectl create serviceaccount aws-load-balancer-controller -n kube-system
```

### 5.2 Annotate the ServiceAccount with the IAM Role ARN

```bash
kubectl annotate serviceaccount aws-load-balancer-controller \
  -n kube-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::497617901838:role/2048iam
```

### 5.3 Install cert-manager (required dependency)

```bash
kubectl apply --validate=false \
  -f https://github.com/jetstack/cert-manager/releases/download/v1.13.5/cert-manager.yaml

# Wait for cert-manager pods to be Running
kubectl get pods -n cert-manager
```

### 5.4 Add the EKS Helm chart repo

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
```

### 5.5 Install the ALB Controller

```bash
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=test \
  --set serviceAccount.create=false \
  --set vpcId=vpc-0db6315056a249f3b \
  --set serviceAccount.name=aws-load-balancer-controller
```

### 5.6 Verify the controller is running

```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```

**Expected output:**
```
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   2/2     2            2           1m
```

---

## Step 6 — Deploy the 2048 Game

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

## Step 7 — Verify Deployment and Access the Game

### 7.1 Check all resources

```bash
# Check pods are Running
kubectl get pods -n game-2048

# Check service
kubectl get svc -n game-2048

# Check ingress — wait for ADDRESS to be populated (2-5 minutes)
kubectl get ingress -n game-2048
```

### 7.2 Get the ALB DNS name

```bash
kubectl get ingress ingress-2048 -n game-2048 \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

**Example output:**
```
k8s-game2048-ingress2-xxxxxxxxxxxx.ap-south-1.elb.amazonaws.com
```

Open this URL in your browser — the 2048 game will load.

### 7.3 Verify ALB in AWS Console

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
| ServiceAccount annotation missing | Re-run the `kubectl annotate` command in Step 5.2 |
| `aws` command not found | Re-run the AWS CLI install steps in Step 3.2 |
| `helm` command not found | Re-run the Helm install steps in Step 3.3 |

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

Then delete the IAM Role `2048iam` and its policy from the [IAM Console](https://console.aws.amazon.com/iam/home).

---

## References

- [AWS Load Balancer Controller Docs](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [EKS Workshop — 2048 Game](https://www.eksworkshop.com/)
- [cert-manager Releases](https://github.com/jetstack/cert-manager/releases)
- [Helm Installation Guide](https://helm.sh/docs/intro/install/)
- [kubectl Installation Guide](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)
