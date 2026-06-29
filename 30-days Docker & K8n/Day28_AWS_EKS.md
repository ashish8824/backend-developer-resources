# Day 28: Managed Kubernetes on AWS EKS

**Goal for today:** Understand what AWS EKS is, how it differs from self-managed Kubernetes and from ECS Fargate (which you already use), how to create and configure an EKS cluster, how IAM integrates with Kubernetes (IRSA), how the AWS Load Balancer Controller provisions ALBs from Ingress objects, how to use Fargate profiles for serverless nodes, and how to connect your entire CI/CD pipeline to EKS. By the end, you should be able to confidently work with Kubernetes on AWS at a production level.

**Format:** WHAT → WHY → HOW for every concept.

---

## 0. Quick Recap (open your teaching session with this)

- Day 9: AWS ECR — pushing images, authentication
- Day 27: CI/CD pipeline — GitHub Actions deploying to Kubernetes
- Your background: PulseBloom and QueueCare deployed on ECS Fargate

**Today's connecting thought:**
> You already know ECS Fargate — you've deployed production apps with it. EKS is Kubernetes running on AWS with deep AWS service integration. Today we map everything you know about Kubernetes (Days 11–27) onto AWS infrastructure, and draw the line between ECS Fargate and EKS so you know exactly when to use which.

---

## 1. ECS Fargate vs EKS — When to Use Which

Since you already use ECS Fargate, this comparison is your anchor point for today.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ECS FARGATE vs EKS                                │
├───────────────────────┬─────────────────────────────────────────────┤
│  ECS FARGATE          │  EKS                                         │
├───────────────────────┼─────────────────────────────────────────────┤
│ AWS-proprietary       │ Industry-standard Kubernetes                 │
│ Simpler to start      │ More complex, more powerful                  │
│ Less config needed    │ Full control over every component            │
│ No node management    │ Nodes or Fargate profiles                    │
│ Task Definitions      │ Pod/Deployment YAML                          │
│ ALB/NLB via AWS       │ AWS LBC or Nginx Ingress                     │
│ IAM task roles        │ IAM Roles for Service Accounts (IRSA)       │
│ No ecosystem          │ Huge Helm/CNCF ecosystem                     │
│ Vendor lock-in        │ Portable across clouds                       │
│ Great for: simple,    │ Great for: complex, multi-team,              │
│ serverless workloads  │ microservices, needs K8s ecosystem           │
└───────────────────────┴─────────────────────────────────────────────┘
```

**Teaching line:**
> "ECS Fargate is like hiring a fully-managed catering company — you tell them what food you want (Task Definition) and they handle everything else. EKS is like running your own restaurant kitchen — you control every aspect (Kubernetes), but you can hire AWS to manage the kitchen infrastructure (managed control plane). The food (your app) is the same; the operational model is different."

**When to choose EKS over ECS:**
- You need Kubernetes-specific tooling (Helm, Argo CD, Istio, Keda, Knative)
- Multi-cloud strategy (same K8s manifests work on GKE, AKS, on-premises)
- Team already knows Kubernetes
- Complex microservices needing advanced networking (NetworkPolicies, service mesh)
- Need StatefulSets with fine-grained storage control

---

## 2. What is Amazon EKS?

### WHAT
**Amazon EKS (Elastic Kubernetes Service)** is a managed Kubernetes service. AWS runs and manages the Kubernetes Control Plane (API Server, etcd, Scheduler, Controller Manager) across multiple Availability Zones with automatic upgrades and high availability. You manage only the worker nodes (or use Fargate for serverless nodes).

### WHY EKS over self-managed Kubernetes
```
Self-managed K8s on EC2:
  → You install and configure every control plane component
  → You manage etcd backups
  → You handle control plane HA across AZs
  → You patch and upgrade everything
  → High operational overhead

EKS:
  → Control plane fully managed by AWS (you never SSH into it)
  → etcd automatically backed up
  → Multi-AZ control plane (99.95% SLA)
  → One-command upgrades
  → Integrated with IAM, VPC, ALB, CloudWatch, ECR
  → You focus on running apps, not managing K8s infrastructure
```

### EKS Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        AWS Account                                  │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │           EKS Control Plane (AWS Managed)                    │   │
│  │  API Server │ etcd │ Scheduler │ Controller Manager          │   │
│  │  Multi-AZ, HA, automatically patched and upgraded           │   │
│  └─────────────────────┬───────────────────────────────────────┘   │
│                         │ kubelet connects via                      │
│                         │ EKS endpoint (HTTPS)                      │
│                         ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    Your VPC                                   │  │
│  │                                                               │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │  │
│  │  │  Node Group  │  │  Node Group  │  │   Fargate    │         │  │
│  │  │  (EC2 AZ-a)  │  │  (EC2 AZ-b)  │  │  Profile     │         │  │
│  │  │             │  │             │  │  (Serverless)│         │  │
│  │  │  [Pod][Pod]  │  │  [Pod][Pod]  │  │  [Pod][Pod]  │         │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘          │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────┐    │  │
│  │  │  AWS Load Balancer Controller → ALB → Services/Ingress│   │  │
│  │  └──────────────────────────────────────────────────────┘    │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 3. Creating an EKS Cluster

### Option 1: eksctl (Recommended — simplest)

`eksctl` is the official CLI tool for EKS, maintained by AWS and Weaveworks.

```bash
# Install eksctl
# macOS
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl

# Linux
curl --silent --location \
  "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
  | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Verify
eksctl version
```

### Simple cluster creation

```bash
# Create a basic cluster (takes ~15-20 minutes)
eksctl create cluster \
  --name taskapi-prod \
  --region ap-south-1 \
  --version 1.29 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 5 \
  --managed                # managed node group (AWS manages node lifecycle)
```

### Production cluster with config file

```yaml
# cluster-config.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: taskapi-prod
  region: ap-south-1
  version: "1.29"
  tags:
    Environment: production
    Team: platform
    Project: taskapi

# VPC configuration
vpc:
  cidr: "10.0.0.0/16"
  nat:
    gateway: HighlyAvailable   # NAT gateway in each AZ

# IAM configuration
iam:
  withOIDC: true               # enable IRSA (IAM Roles for Service Accounts)

# Managed node groups
managedNodeGroups:
  # ── General workloads ────────────────────────────────────────────────────────
  - name: general-workers
    instanceType: t3.xlarge
    minSize: 2
    maxSize: 10
    desiredCapacity: 3
    availabilityZones:
      - ap-south-1a
      - ap-south-1b
      - ap-south-1c
    volumeSize: 50             # EBS volume per node (GB)
    labels:
      role: worker
      node-type: general
    tags:
      NodeGroup: general-workers
    iam:
      attachPolicyARNs:
        - arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
        - arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy
        - arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

  # ── Spot instances for non-critical workloads (cheaper) ─────────────────────
  - name: spot-workers
    instanceTypes:
      - t3.large
      - t3.xlarge
      - t3a.large
    spot: true
    minSize: 0
    maxSize: 10
    desiredCapacity: 2
    labels:
      role: spot-worker
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule

# Fargate profiles (serverless nodes for specific workloads)
fargateProfiles:
  - name: monitoring
    selectors:
      - namespace: monitoring
  - name: kube-system
    selectors:
      - namespace: kube-system

# Add-ons
addons:
  - name: vpc-cni                      # AWS VPC CNI (native pod IPs)
    version: latest
  - name: coredns
    version: latest
  - name: kube-proxy
    version: latest
  - name: aws-ebs-csi-driver           # EBS volumes for PVCs
    version: latest
    wellKnownPolicies:
      ebsCSIController: true
```

```bash
# Create cluster from config file
eksctl create cluster -f cluster-config.yaml

# This creates:
# ✅ VPC with public/private subnets in 3 AZs
# ✅ NAT Gateways
# ✅ EKS Control Plane
# ✅ Managed Node Groups
# ✅ Fargate Profiles
# ✅ IAM roles for nodes
# ✅ Security groups
# ✅ kubeconfig updated automatically

# After creation — verify
kubectl get nodes
kubectl get nodes -o wide    # shows node IPs, instance types, AZs
```

---

## 4. Node Groups vs Fargate Profiles

### Managed Node Groups (EC2-based)

```
You define: instance type, min/max/desired, AZs, disk size
AWS manages: AMI updates, node lifecycle, draining during updates

Pros:
  ✅ Full control over instance type
  ✅ Can use GPU instances (ml.g4dn.*)
  ✅ Can use Spot instances (60-90% cheaper)
  ✅ Better for StatefulSets (persistent storage)
  ✅ Run DaemonSets

Cons:
  ❌ You still manage node scaling (Cluster Autoscaler or Karpenter needed)
  ❌ Pay for idle node capacity
  ❌ Node maintenance and upgrades
```

### Fargate Profiles (Serverless)

```
You define: namespace + label selectors
AWS manages: everything (no node concept — each pod gets its own micro-VM)

Pros:
  ✅ Zero node management
  ✅ Pay per pod (not per node)
  ✅ Built-in isolation (each pod in its own VM)
  ✅ Scales to zero
  ✅ No DaemonSet overhead

Cons:
  ❌ No DaemonSets (no nodes to run on)
  ❌ No GPU support
  ❌ No EBS volumes (only EFS)
  ❌ Slower pod startup (~1-2 min vs seconds for EC2)
  ❌ Some K8s features limited
  ❌ Max 4 vCPU / 30 GB RAM per pod
```

### When to use each

```bash
# Mixed strategy (common in production):
# → EC2 Node Groups: stateful workloads, DaemonSets, GPU, high-perf
# → Fargate: stateless workloads, batch jobs, dev/test environments

# Example Fargate profile — only runs pods in specific namespaces
eksctl create fargateprofile \
  --cluster taskapi-prod \
  --name development \
  --namespace development \
  --region ap-south-1
```

---

## 5. IAM Integration — IRSA (IAM Roles for Service Accounts)

### The Problem
Your Pods running on EKS often need to access AWS services:
- API pod needs to read/write S3 (file uploads)
- Backup CronJob needs to write to S3
- App needs to read from AWS Secrets Manager
- AWS Load Balancer Controller needs to manage ALBs

How do you give Pods AWS permissions WITHOUT hardcoding IAM credentials?

### The Solution — IRSA
**IAM Roles for Service Accounts (IRSA)** links a Kubernetes ServiceAccount to an AWS IAM Role using OIDC federation. When a Pod uses that ServiceAccount, AWS automatically provides temporary credentials — no static credentials needed.

```
Pod uses ServiceAccount "s3-writer"
        │
        │ ServiceAccount has annotation:
        │ eks.amazonaws.com/role-arn: arn:aws:iam::123:role/taskapi-s3-role
        ▼
AWS OIDC Provider validates the Pod's identity
        │
        ▼
AWS STS issues temporary credentials for taskapi-s3-role
        │
        ▼
Pod can now access S3 with those permissions
No static access keys anywhere!
```

### Setting Up IRSA

**Step 1: Enable OIDC provider for the cluster (if not done in eksctl config)**
```bash
eksctl utils associate-iam-oidc-provider \
  --cluster taskapi-prod \
  --region ap-south-1 \
  --approve
```

**Step 2: Create IAM Role for your service**
```bash
# Example: IAM role for S3 access
eksctl create iamserviceaccount \
  --cluster taskapi-prod \
  --region ap-south-1 \
  --namespace production \
  --name taskapi-s3-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess \
  --approve \
  --override-existing-serviceaccounts

# This creates:
# ✅ IAM Role with trust policy for this specific ServiceAccount
# ✅ Kubernetes ServiceAccount with the role ARN annotation
```

**Step 3: Reference ServiceAccount in your Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskapi
  namespace: production
spec:
  template:
    spec:
      serviceAccountName: taskapi-s3-sa   # ← use the IRSA service account
      containers:
        - name: api
          image: taskapi:2.0
          # No AWS credentials needed!
          # The SDK automatically picks up credentials from the mounted token
```

**Step 4: Use AWS SDK in your app — credentials are automatic**
```javascript
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');

// No credentials config needed — IRSA provides them automatically
const s3Client = new S3Client({ region: 'ap-south-1' });

async function uploadFile(buffer, key) {
  await s3Client.send(new PutObjectCommand({
    Bucket: 'taskapi-uploads',
    Key: key,
    Body: buffer
  }));
}
```

---

## 6. AWS Load Balancer Controller — Native ALB Integration

### WHAT
The **AWS Load Balancer Controller** is a Kubernetes controller that watches for Ingress and Service objects and automatically provisions AWS Application Load Balancers (ALB) and Network Load Balancers (NLB).

### WHY over nginx-ingress on EKS
```
nginx-ingress on EKS:
  → Creates a Classic/NLB pointing to nginx pods
  → Extra hop: ALB → nginx pod → your pod
  → You manage nginx capacity

AWS Load Balancer Controller:
  → Creates ALB directly (no nginx pod)
  → Direct: ALB → your pod (target groups)
  → AWS manages ALB capacity
  → Native WAF, SSL, access logs integration
  → Pod Readiness Gates (pod not registered to ALB until ready)
```

### Installing AWS Load Balancer Controller

```bash
# Step 1: Create IAM policy for the controller
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.7.0/docs/install/iam_policy.json

aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Step 2: Create service account with IRSA
eksctl create iamserviceaccount \
  --cluster taskapi-prod \
  --region ap-south-1 \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):policy/AWSLoadBalancerControllerIAMPolicy \
  --approve

# Step 3: Install via Helm
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  --namespace kube-system \
  --set clusterName=taskapi-prod \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=$(aws eks describe-cluster \
    --name taskapi-prod \
    --query "cluster.resourcesVpcConfig.vpcId" \
    --output text)

# Verify
kubectl get deployment aws-load-balancer-controller -n kube-system
```

### Ingress with ALB Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: taskapi-ingress
  namespace: production
  annotations:
    # ── ALB Controller ─────────────────────────────────────────────────────────
    kubernetes.io/ingress.class: alb             # use ALB (not nginx)
    alb.ingress.kubernetes.io/scheme: internet-facing    # public-facing ALB
    alb.ingress.kubernetes.io/target-type: ip    # route directly to pod IPs

    # ── SSL ────────────────────────────────────────────────────────────────────
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-south-1:123:certificate/abc
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'

    # ── Health check ───────────────────────────────────────────────────────────
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "3"

    # ── WAF (Web Application Firewall) ─────────────────────────────────────────
    alb.ingress.kubernetes.io/wafv2-acl-arn: arn:aws:wafv2:ap-south-1:123:...

    # ── Access logs ────────────────────────────────────────────────────────────
    alb.ingress.kubernetes.io/load-balancer-attributes: >
      access_logs.s3.enabled=true,
      access_logs.s3.bucket=taskapi-alb-logs,
      idle_timeout.timeout_seconds=60

    # ── ALB group (share one ALB across multiple Ingresses) ────────────────────
    alb.ingress.kubernetes.io/group.name: taskapi-shared-alb

spec:
  ingressClassName: alb
  rules:
    - host: api.taskapp.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 3000
```

---

## 7. EBS Volumes on EKS — StorageClass Setup

For PersistentVolumeClaims on EKS, you need the EBS CSI Driver (already added in our eksctl config):

```yaml
# Production StorageClass for EKS
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # make this the default
provisioner: ebs.csi.aws.com
parameters:
  type: gp3                       # gp3 is cheaper and faster than gp2
  iops: "3000"
  throughput: "125"
  encrypted: "true"               # encrypt all volumes at rest
  kmsKeyId: arn:aws:kms:ap-south-1:123:key/abc  # optional: custom KMS key
reclaimPolicy: Retain             # NEVER auto-delete production data
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer   # provision in same AZ as pod
```

```bash
kubectl apply -f storageclass.yaml

# Test PVC
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-pvc
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 10Gi
  storageClassName: gp3-encrypted
EOF

kubectl get pvc test-pvc    # should become Bound within 30s
```

---

## 8. Cluster Autoscaler — Automatic Node Scaling

### WHAT
**Cluster Autoscaler** automatically adds/removes nodes when:
- Pods can't be scheduled (insufficient resources) → add nodes
- Nodes are underutilised for 10+ minutes → remove nodes

### HOW

```bash
# Create IAM policy for Cluster Autoscaler
eksctl create iamserviceaccount \
  --cluster taskapi-prod \
  --region ap-south-1 \
  --namespace kube-system \
  --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::aws:policy/AutoScalingFullAccess \
  --approve

# Deploy Cluster Autoscaler
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=taskapi-prod \
  --set awsRegion=ap-south-1 \
  --set rbac.serviceAccount.create=false \
  --set rbac.serviceAccount.name=cluster-autoscaler
```

```bash
# Watch autoscaler in action
kubectl logs -l app.kubernetes.io/name=cluster-autoscaler -n kube-system -f

# Scale your deployment beyond current node capacity
kubectl scale deployment taskapi --replicas=50 -n production
# Watch: new EC2 instances join the cluster automatically
kubectl get nodes -w

# Scale back down
kubectl scale deployment taskapi --replicas=3 -n production
# Watch: underutilised nodes are drained and terminated
```

---

## 9. AWS Secrets Manager Integration — External Secrets on EKS

Instead of managing Kubernetes Secrets manually, pull them from AWS Secrets Manager:

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace

# Create IRSA for External Secrets to access Secrets Manager
eksctl create iamserviceaccount \
  --cluster taskapi-prod \
  --region ap-south-1 \
  --namespace external-secrets \
  --name external-secrets-sa \
  --attach-policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite \
  --approve
```

```yaml
# ClusterSecretStore — defines where secrets come from
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-manager
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets

---
# ExternalSecret — pulls specific secrets from AWS
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: taskapi-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: taskapi-secrets     # creates a K8s Secret with this name
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: production/taskapi     # AWS Secrets Manager secret name
        property: db_password        # JSON key in the secret
    - secretKey: JWT_SECRET
      remoteRef:
        key: production/taskapi
        property: jwt_secret
    - secretKey: DATABASE_URL
      remoteRef:
        key: production/taskapi
        property: database_url
```

---

## 10. Connecting Your CI/CD Pipeline to EKS

Update your GitHub Actions pipeline from Day 27 to work with EKS:

```yaml
# In .github/workflows/deploy-prod.yml

- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ap-south-1

# EKS-specific: update kubeconfig
- name: Update kubeconfig for EKS
  run: |
    aws eks update-kubeconfig \
      --name taskapi-prod \
      --region ap-south-1
      # Optional: --role-arn arn:aws:iam::123:role/deploy-role

- name: Deploy with Helm
  run: |
    helm upgrade --install taskapi ./charts/taskapi \
      --namespace production \
      -f charts/taskapi/values-prod.yaml \
      --set image.repository=123456789.dkr.ecr.ap-south-1.amazonaws.com/taskapi \
      --set image.tag=${{ github.sha }} \
      --atomic \
      --timeout 15m
```

**IAM permissions required for the CI/CD IAM user:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "eks:DescribeCluster",
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "*"
    }
  ]
}
```

**And the EKS aws-auth ConfigMap must allow the IAM user:**
```bash
# Grant the CI/CD IAM user kubectl access
eksctl create iamidentitymapping \
  --cluster taskapi-prod \
  --region ap-south-1 \
  --arn arn:aws:iam::123456789:user/github-actions-deploy \
  --username github-actions \
  --group system:masters    # or more restrictive group
```

---

## 11. EKS Useful Commands

```bash
# ── Cluster management ─────────────────────────────────────────────────────────
eksctl get cluster --region ap-south-1
eksctl get nodegroup --cluster taskapi-prod --region ap-south-1
eksctl scale nodegroup --cluster taskapi-prod \
  --name general-workers --nodes 5 --region ap-south-1

# ── Upgrade cluster version ────────────────────────────────────────────────────
eksctl upgrade cluster --name taskapi-prod --version 1.30 --region ap-south-1 --approve
eksctl upgrade nodegroup --cluster taskapi-prod --name general-workers --region ap-south-1

# ── Node management ────────────────────────────────────────────────────────────
kubectl get nodes -o wide                          # show IPs, AZs, instance types
kubectl describe node ip-10-0-1-5.ec2.internal    # full node details
kubectl cordon <node>     # mark node unschedulable (maintenance)
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data  # evict all pods
kubectl uncordon <node>   # make node schedulable again

# ── AWS-specific debugging ────────────────────────────────────────────────────
# Check if ALB was created for an Ingress
kubectl describe ingress taskapi-ingress -n production
# Look for: "Address: k8s-prod-taskapi-abc.ap-south-1.elb.amazonaws.com"

# Check AWS Load Balancer Controller logs
kubectl logs -l app.kubernetes.io/name=aws-load-balancer-controller \
  -n kube-system --tail=50

# Check EBS volumes created for PVCs
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/cluster/taskapi-prod,Values=owned" \
  --query 'Volumes[*].{ID:VolumeId,Size:Size,State:State,AZ:AvailabilityZone}'

# Check IRSA is working (from inside a pod)
kubectl exec -it <pod> -n production -- \
  aws sts get-caller-identity
# Should show the IAM role ARN, not the node's instance profile

# ── Fargate ───────────────────────────────────────────────────────────────────
eksctl get fargateprofile --cluster taskapi-prod --region ap-south-1
# Check which pods are on Fargate vs EC2
kubectl get pods -n production -o wide
# Fargate pods show a node name like: fargate-ip-10-0-1-5.ap-south-1.compute.internal

# ── Delete cluster (⚠️ destroys everything) ─────────────────────────────────
eksctl delete cluster --name taskapi-prod --region ap-south-1
```

---

## 12. EKS Cost Optimisation

```
EKS control plane: $0.10/hour = ~$72/month per cluster
  → Fixed cost regardless of workload size

EC2 Node Groups:
  → t3.medium (2 vCPU, 4 GB): ~$0.042/hr = ~$30/month per node
  → Use Spot instances: 60-90% cheaper (tolerate interruptions)
  → Use Savings Plans or Reserved Instances for baseline capacity

Fargate:
  → $0.04048/vCPU/hour + $0.004445/GB/hour
  → Only pay when pods are running
  → Great for dev/test that scales to zero

Cost optimisation strategies:
  1. Use Spot instances for non-critical workloads
  2. Cluster Autoscaler + scale to zero at night (dev clusters)
  3. Use gp3 EBS instead of gp2 (same performance, 20% cheaper)
  4. Fargate for batch jobs and low-traffic dev environments
  5. Karpenter instead of Cluster Autoscaler (better bin packing)
  6. Use Savings Plans for EKS + EC2 together
```

---

## 13. EKS vs Self-Managed vs ECS — Final Decision Matrix

```
┌──────────────────────────────────────────────────────────────────────┐
│  Deployment Option     │ Manage K8s? │ Manage Nodes? │ Best for      │
├────────────────────────┼─────────────┼───────────────┼───────────────┤
│ Self-managed K8s on EC2│ YES (hard)  │ YES           │ Full control  │
│ Amazon EKS + EC2       │ NO (AWS)    │ YES           │ Flexibility   │
│ Amazon EKS + Fargate   │ NO (AWS)    │ NO            │ Serverless K8s│
│ Amazon ECS Fargate     │ N/A (ECS)   │ NO            │ Simplicity    │
└──────────────────────────────────────────────────────────────────────┘

Decision guide:
→ Small team, simple apps, AWS-only, fast start → ECS Fargate (like your current setup)
→ Need K8s ecosystem, Helm, multi-cloud, complex apps → EKS + EC2
→ Want K8s but hate node management → EKS + Fargate (hybrid)
→ Need maximum control → Self-managed K8s (rarely justified)
```

---

## 14. Hands-On Practice Tasks

```bash
# 1. Install eksctl and verify
eksctl version

# 2. Create a small test cluster (be mindful of AWS costs)
eksctl create cluster \
  --name taskapi-learning \
  --region ap-south-1 \
  --version 1.29 \
  --node-type t3.small \
  --nodes 2 \
  --managed

# 3. Verify nodes joined
kubectl get nodes -o wide

# 4. Deploy your TaskAPI Helm chart
helm upgrade --install taskapi ./charts/taskapi \
  -f charts/taskapi/values-staging.yaml \
  --namespace staging --create-namespace

# 5. Install AWS Load Balancer Controller
# (follow Section 6 steps)

# 6. Create an Ingress with ALB annotations
# Watch the ALB appear in AWS Console → EC2 → Load Balancers

# 7. Set up IRSA for S3 access
# (follow Section 5 steps)

# 8. Verify IRSA works
kubectl exec -it deploy/taskapi -n staging -- \
  aws sts get-caller-identity

# 9. Install Cluster Autoscaler and test
kubectl scale deployment taskapi --replicas=20 -n staging
kubectl get nodes -w   # watch new nodes join

# 10. ⚠️ Delete the cluster when done (avoid AWS charges)
eksctl delete cluster --name taskapi-learning --region ap-south-1
```

---

## 15. Quiz — Test Yourself / Test Others

1. What does AWS manage in EKS vs what do you manage?
2. What is the main difference between EKS with EC2 node groups and EKS with Fargate profiles?
3. What is IRSA and why is it better than using static AWS access keys inside Pods?
4. What does the AWS Load Balancer Controller do and why is it preferred over nginx-ingress on EKS?
5. What is `volumeBindingMode: WaitForFirstConsumer` critical for on EKS?
6. What does `eksctl create iamserviceaccount` create in both AWS and Kubernetes?
7. How do you grant a GitHub Actions IAM user access to run `kubectl` commands against EKS?
8. When would you choose ECS Fargate over EKS?
9. What is the `aws-auth` ConfigMap and what is it used for?
10. You deployed an Ingress on EKS but no ALB appeared. What are the most likely causes?

<details>
<summary>Answers</summary>

1. AWS manages: the entire Control Plane (API Server, etcd, Scheduler, Controller Manager), multi-AZ HA, etcd backups, control plane security patching, and control plane version upgrades. You manage: worker nodes (EC2 instance types, node group sizing, AMI updates) OR use Fargate to avoid node management, all Kubernetes workloads (Deployments, Services, etc.), add-ons (VPC CNI, CoreDNS, EBS CSI Driver), networking configuration, and cluster upgrades (though AWS makes these easier).

2. EC2 Node Groups run your Pods on EC2 instances you partially manage — you choose instance type, configure min/max nodes, and handle node upgrades. Fargate Profiles run each Pod in its own isolated micro-VM fully managed by AWS — no EC2 instances, no node maintenance, pay-per-pod. EC2 is better for DaemonSets, StatefulSets, GPU, and performance-sensitive workloads. Fargate is better for stateless workloads where you want zero node management.

3. IRSA (IAM Roles for Service Accounts) links a Kubernetes ServiceAccount to an AWS IAM Role via OIDC federation. When a Pod uses that ServiceAccount, AWS STS automatically provides temporary, auto-rotating credentials scoped to just that role. No static access keys are ever stored anywhere. Static keys are: a security risk if they leak (permanent credentials), difficult to rotate, not scoped to specific Pods, and violate least-privilege principles.

4. The AWS Load Balancer Controller watches Ingress objects and directly provisions AWS Application Load Balancers (ALB) that route traffic straight to Pod IPs (target-type: ip). This eliminates the extra nginx proxy hop, integrates natively with AWS WAF, ACM certificates, access logs, and Shield protection. It also supports ALB Groups — multiple Ingresses sharing one ALB, reducing cost.

5. EBS volumes are Availability Zone-specific — an EBS volume created in ap-south-1a cannot be mounted by a Pod on a node in ap-south-1b. `WaitForFirstConsumer` delays EBS volume provisioning until a Pod using the PVC is actually scheduled to a node, then provisions the volume in the same AZ as that node. Without it, the PVC might provision in the wrong AZ and the Pod can never mount it.

6. In AWS: it creates an IAM Role with a trust policy allowing the specific EKS cluster's OIDC provider to assume it, scoped to the specific Kubernetes namespace and service account name. In Kubernetes: it creates a ServiceAccount in the specified namespace with the annotation `eks.amazonaws.com/role-arn: <role-arn>`. Together these enable IRSA — when a Pod uses this ServiceAccount, AWS automatically provides credentials for the IAM Role.

7. Two steps: (1) Give the IAM user/role the `eks:DescribeCluster` permission and ECR push permissions. (2) Map the IAM user/role in the `aws-auth` ConfigMap using `eksctl create iamidentitymapping --arn <iam-arn> --username <k8s-username> --group <k8s-group>`. The group determines what kubectl permissions the user has — `system:masters` for full admin, or a custom RBAC group for limited access.

8. Choose ECS Fargate over EKS when: your team doesn't know Kubernetes and the learning curve isn't justified, you have simple containerised services without complex orchestration needs, you're AWS-only with no multi-cloud requirements, you want the fastest path to production without infrastructure expertise, and you don't need the broader Kubernetes ecosystem (Helm, Argo CD, service mesh, etc.). ECS Fargate is operationally simpler but less portable and extensible.

9. The `aws-auth` ConfigMap in the `kube-system` namespace maps AWS IAM identities (IAM users, IAM roles) to Kubernetes RBAC identities (usernames and groups). When `kubectl` or any tool authenticates to EKS using AWS credentials, the EKS API Server checks this ConfigMap to determine what Kubernetes permissions that IAM identity has. Without an entry in `aws-auth`, even AWS account owners can't access the cluster via kubectl.

10. Most likely causes: (1) AWS Load Balancer Controller is not installed or not running — check `kubectl get pods -n kube-system | grep aws-load-balancer`. (2) The `kubernetes.io/ingress.class: alb` annotation or `ingressClassName: alb` is missing from the Ingress. (3) The IngressClass resource doesn't exist or isn't named `alb`. (4) The subnets are not tagged correctly for ALB discovery — public subnets need `kubernetes.io/role/elb: 1`. (5) The ALB Controller's IAM role doesn't have permissions to create load balancers.

</details>

---

## 16. Summary

After today you can explain and implement:
- **ECS Fargate vs EKS** — when to use each, operational differences, mapping your existing knowledge
- **EKS architecture** — managed control plane, VPC placement, node groups, Fargate profiles
- **eksctl** — cluster creation with config file, node groups, Fargate profiles, add-ons
- **Node Groups vs Fargate** — trade-offs, Spot instances, DaemonSet limitations on Fargate
- **IRSA** — why it's better than static keys, OIDC federation, `iamserviceaccount` setup, SDK auto-config
- **AWS Load Balancer Controller** — ALB provisioning from Ingress, target-type: ip, WAF, ALB groups
- **EBS CSI Driver** — gp3 StorageClass, encryption, WaitForFirstConsumer for AZ alignment
- **Cluster Autoscaler** — automatic node scaling based on pending pods and utilisation
- **External Secrets Operator** — pulling secrets from AWS Secrets Manager into K8s Secrets
- **CI/CD with EKS** — `aws eks update-kubeconfig`, aws-auth mapping, required IAM permissions
- **Cost optimisation** — Spot instances, Fargate for batch, gp3, Savings Plans

**Coming up on Day 29:** Kubernetes Security — RBAC (Role-Based Access Control), ServiceAccounts, Pod Security Standards, network security hardening, and secrets management best practices. The complete security layer for a production cluster.
