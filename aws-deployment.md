# ClickHouse on AWS EKS — Deployment Guide

> **Operator:** Official ClickHouse (`clickhouse.com/v1alpha1`)
> **Cluster:** `clickhouse-cluster` — ap-southeast-1
> **Topology:** 2 shards × 2 replicas, 3-node Keeper

---

## Prerequisites

```bash
aws --version       # AWS CLI v2
eksctl version      # eksctl 0.180+
kubectl version     # kubectl 1.28+
helm version        # Helm 3.x
```

---

## Step 1 — Authenticate

```bash
# From workspace root
bash scripts/update-aws-credentials.sh

# Verify
aws sts get-caller-identity
```

---

## Step 2 — Create EKS Cluster

From the `clickhouse-operator/` directory:

```bash
eksctl create cluster -f eks-cluster.yaml
```

Takes ~15–20 minutes. Creates:
- VPC across 3 AZs (ap-southeast-1a/b/c)
- 2 × `t3.large` nodes (clickhouse-nodes, `role: clickhouse`)
- 2 × `t3.medium` nodes (keeper-nodes, `role: keeper`)

Verify:

```bash
kubectl get nodes -o wide
# Should show 4 nodes
```

---

## Step 3 — EBS CSI Driver

OIDC is blocked by SCP. Use EKS Pod Identity instead.

```bash
# Create IAM role
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam create-role \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": ["sts:AssumeRole", "sts:TagSession"]
    }]
  }'

aws iam attach-role-policy \
  --role-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

# Install Pod Identity Agent first, then EBS CSI
aws eks create-addon \
  --addon-name eks-pod-identity-agent \
  --cluster-name clickhouse-cluster \
  --region ap-southeast-1

aws eks create-addon \
  --addon-name aws-ebs-csi-driver \
  --cluster-name clickhouse-cluster \
  --region ap-southeast-1

# Associate Pod Identity with the EBS CSI service account
aws eks create-pod-identity-association \
  --cluster-name clickhouse-cluster \
  --region ap-southeast-1 \
  --namespace kube-system \
  --service-account ebs-csi-controller-sa \
  --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole
```

---

## Step 4 — GP3 StorageClass

From the `clickhouse-operator/` directory:

```bash
kubectl apply -f storageclass-gp3.yaml

# Remove old gp2 default
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# Verify — gp3 should show (default)
kubectl get storageclass
```

---

## Step 5 — Install cert-manager

The ClickHouse operator uses webhooks that require TLS certificates issued by cert-manager.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Wait for cert-manager to be ready before proceeding
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=120s
```

Verify:

```bash
kubectl get pods -n cert-manager
# cert-manager-xxxx             1/1   Running
# cert-manager-cainjector-xxxx  1/1   Running
# cert-manager-webhook-xxxx     1/1   Running
```

---

## Step 6 — Install Official ClickHouse Operator

```bash
helm install clickhouse-operator oci://registry-1.docker.io/clickhouse/clickhouse-operator \
  --namespace clickhouse-operator-system \
  --create-namespace
```

Verify:

```bash
kubectl get pods -n clickhouse-operator-system
# clickhouse-operator-controller-manager-xxxx   1/1   Running
```

---

## Step 7 — Deploy ClickHouse Keeper (3-node)

From the `clickhouse-operator/` directory:

```bash
kubectl apply -f keeper-aws.yaml

# Wait for all 3 Keeper pods
kubectl get pods -n default -w
# keeper-keeper-0-0-0   1/1   Running
# keeper-keeper-1-0-0   1/1   Running
# keeper-keeper-2-0-0   1/1   Running
```

---

## Step 8 — Deploy ClickHouse (2 Shards × 2 Replicas)

From the `clickhouse-operator/` directory:

```bash
kubectl apply -f clickhouse-aws.yaml

# Wait for all 4 ClickHouse pods
kubectl get pods -n default -w
# clickhouse-clickhouse-0-0-0   1/1   Running
# clickhouse-clickhouse-0-1-0   1/1   Running
# clickhouse-clickhouse-1-0-0   1/1   Running
# clickhouse-clickhouse-1-1-0   1/1   Running
```

---

## Step 9 — Expose via NLB

The official operator does not auto-create a LoadBalancer service. Apply it manually:

```bash
kubectl apply -f nlb-service.yaml

# Wait for NLB DNS (~2 min to provision)
kubectl get svc clickhouse-nlb -n default -w
```

---

## Step 10 — Connect

```bash
clickhouse client \
  --host <NLB-DNS-from-above> \
  --port 9000
```

> Do NOT use `localhost` or `kubectl port-forward` — Docker Desktop automatically
> maps LoadBalancer services to localhost, which conflicts with local ports.

Verify cluster topology:

```sql
SELECT cluster, shard_num, replica_num, host_name, is_local
FROM system.clusters
WHERE cluster = 'cluster'
ORDER BY shard_num, replica_num;
```

---

## Quick Reference

```bash
# 0. Authenticate
bash scripts/update-aws-credentials.sh

# 1. Create EKS cluster (~15-20 min) — from clickhouse-operator/
eksctl create cluster -f eks-cluster.yaml

# 2. EBS CSI (Pod Identity)
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws iam create-role --role-name AmazonEKS_EBS_CSI_DriverRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"pods.eks.amazonaws.com"},"Action":["sts:AssumeRole","sts:TagSession"]}]}'
aws iam attach-role-policy --role-name AmazonEKS_EBS_CSI_DriverRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy
aws eks create-addon --addon-name eks-pod-identity-agent \
  --cluster-name clickhouse-cluster --region ap-southeast-1
aws eks create-addon --addon-name aws-ebs-csi-driver \
  --cluster-name clickhouse-cluster --region ap-southeast-1
aws eks create-pod-identity-association \
  --cluster-name clickhouse-cluster --region ap-southeast-1 \
  --namespace kube-system --service-account ebs-csi-controller-sa \
  --role-arn arn:aws:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole

# 3. GP3 StorageClass — from clickhouse-operator/
kubectl apply -f storageclass-gp3.yaml
kubectl patch storageclass gp2 -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'

# 4. cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
kubectl wait --for=condition=Ready pods --all -n cert-manager --timeout=120s

# 5. Official ClickHouse operator
helm install clickhouse-operator oci://registry-1.docker.io/clickhouse/clickhouse-operator \
  --namespace clickhouse-operator-system --create-namespace

# 6. Keeper + ClickHouse — from clickhouse-operator/
kubectl apply -f keeper-aws.yaml
kubectl get pods -n default -w   # wait for 3 keeper pods Running
kubectl apply -f clickhouse-aws.yaml
kubectl get pods -n default -w   # wait for 4 clickhouse pods Running

# 7. Expose NLB
kubectl apply -f nlb-service.yaml
kubectl get svc clickhouse-nlb -n default

# 8. Connect
clickhouse client --host <NLB-DNS> --port 9000
```

---

## Pausing to Save Cost

The EC2 nodes are the main cost driver. When not using the cluster, scale them to zero:

```bash
# Scale down all nodes (stops EC2 billing, EBS volumes are retained)
eksctl scale nodegroup \
  --cluster clickhouse-cluster \
  --region ap-southeast-1 \
  --name clickhouse-nodes \
  --nodes 0 --nodes-min 0

eksctl scale nodegroup \
  --cluster clickhouse-cluster \
  --region ap-southeast-1 \
  --name keeper-nodes \
  --nodes 0 --nodes-min 0
```

When you want to use it again:

```bash
# Scale back up
eksctl scale nodegroup \
  --cluster clickhouse-cluster \
  --region ap-southeast-1 \
  --name clickhouse-nodes \
  --nodes 2 --nodes-min 2

eksctl scale nodegroup \
  --cluster clickhouse-cluster \
  --region ap-southeast-1 \
  --name keeper-nodes \
  --nodes 2 --nodes-min 2

# Redeploy (pods were evicted when nodes went to 0) — from clickhouse-operator/
kubectl apply -f keeper-aws.yaml
kubectl get pods -n default -w      # wait for 3 Keeper pods Running
kubectl apply -f clickhouse-aws.yaml
kubectl get pods -n default -w      # wait for 4 ClickHouse pods Running
kubectl apply -f nlb-service.yaml
kubectl get svc clickhouse-nlb -n default   # get NLB DNS
```

> **What this saves:** EC2 costs (~$0.10/hr for t3.large × 2, ~$0.05/hr for t3.medium × 2)
>
> **What still runs:** EKS control plane (~$0.10/hr = ~$2.4/day), EBS volumes (~$0.08/GB/month)
>
> **Data:** EBS volumes are retained when nodes scale to 0, so ClickHouse data survives.

---

## Tear Down (Full — Zero Cost)

Run in order:

```bash
# 1. Delete ClickHouse resources and NLB
kubectl delete -f nlb-service.yaml
kubectl delete -f clickhouse-aws.yaml
kubectl delete -f keeper-aws.yaml

# 2. Delete all PVCs (removes EBS volumes — data is lost)
kubectl delete pvc --all -n default

# 3. Uninstall ClickHouse operator
helm uninstall clickhouse-operator -n clickhouse-operator-system

# 4. Uninstall cert-manager
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# 5. Delete EKS cluster — removes all EC2 nodes, control plane, NLB, VPC (~10 min)
eksctl delete cluster --name clickhouse-cluster --region ap-southeast-1
```

After this your AWS account has no running resources and zero ongoing cost. To redeploy, follow the guide from Step 1.
