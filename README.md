# PostgreSQL on EKS using AWS Secrets Manager

## IRSA + Secrets Store CSI Driver (End-to-End Practical Guide)

---

## Overview

This repository provides a complete, step-by-step guide to deploy **PostgreSQL on Amazon EKS** using:

* Kubernetes StatefulSet
* AWS Secrets Manager for credentials
* IAM Roles for Service Accounts (IRSA)
* Secrets Store CSI Driver (AWS provider)
* AWS EBS CSI Driver for persistent storage

No secrets are stored in Git or hardcoded in Kubernetes manifests.

---

## Prerequisites

* AWS Account
* EKS Cluster
* Managed Node Group attached to the cluster
* IAM user with EKS + IAM permissions
* Tools installed locally:

  * awscli
  * kubectl
  * eksctl
  * helm

---

## Step 0: Configure AWS CLI

Configure AWS credentials:

```bash
aws configure
```

Verify identity:

```bash
aws sts get-caller-identity
```

---

## Step 1: Connect kubectl to EKS Cluster

Update kubeconfig for your EKS cluster:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name <EKS_CLUSTER_NAME>
```

Verify cluster connectivity:

```bash
kubectl get nodes
```

You should see worker nodes in `Ready` state.

---

## Step 2: Create Secret in AWS Secrets Manager

Create a secret with the following details:

**Secret Name**

```
eks/postgres/db02
```

**Secret Type**

```
Other type of secret
```

**Key–Value Pairs**

```
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres123
POSTGRES_DB=testdb
```

Verify the secret:

```bash
aws secretsmanager describe-secret \
  --secret-id eks/postgres/db02 \
  --region ap-south-1
```

---

## Step 3: Install Required EKS Addons

### 3.1 Install AWS EBS CSI Driver

```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster <EKS_CLUSTER_NAME> \
  --region ap-south-1 \
  --force
```

Verify:

```bash
kubectl get pods -n kube-system | grep ebs
```

---

### 3.2 Install Secrets Store CSI Driver

```bash
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts

helm repo update

helm install secrets-store-csi-driver \
  secrets-store-csi-driver/secrets-store-csi-driver \
  --namespace kube-system \
  --set syncSecret.enabled=true
```

Verify:

```bash
kubectl get pods -n kube-system | grep secrets
kubectl get crd | grep secretprovider
```

---

### 3.3 Install AWS Provider for Secrets Store CSI Driver

```bash
kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml
```

Verify:

```bash
kubectl get pods -n kube-system | grep aws
```

---

## Step 4: Configure IAM Role for Service Account (IRSA)

### 4.1 IAM Policy

Attach this policy to an IAM role:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:ap-south-1:<ACCOUNT_ID>:secret:eks/postgres/db02-*"
    }
  ]
}
```

---

### 4.2 Trust Policy

Trust relationship must allow:

```
system:serviceaccount:database:postgres-sa
```

OIDC provider must match your EKS cluster.

---

## Step 5: Apply Kubernetes Manifests

Apply files **in order**:

```bash
kubectl apply -f manifests/01-namespace.yaml
kubectl apply -f manifests/02-serviceaccount.yaml
kubectl apply -f manifests/03-secretproviderclass.yaml
kubectl apply -f manifests/04-postgres-service.yaml
kubectl apply -f manifests/05-postgres-statefulset.yaml
```

---

## Step 6: What Happens Internally

1. Pod starts using `postgres-sa`
2. IRSA provides temporary AWS credentials
3. CSI Driver fetches secrets from AWS Secrets Manager
4. Kubernetes Secret is auto-created
5. PostgreSQL reads credentials securely
6. EBS volume is attached
7. PostgreSQL initializes and starts

---

## Step 7: Verification

### 7.1 Pod and PVC

```bash
kubectl get pods -n database
kubectl get pvc -n database
```

---

### 7.2 Verify Secret Sync

```bash
kubectl get secret postgres-k8s-secret -n database -o yaml
```

---

### 7.3 Connect to PostgreSQL

```bash
kubectl exec -it postgres-0 -n database -- psql -U postgres -d testdb
```

Expected prompt:

```
testdb=#
```

---

## Repository Structure

```
.
├── README.md
└── manifests
    ├── 01-namespace.yaml
    ├── 02-serviceaccount.yaml
    ├── 03-secretproviderclass.yaml
    ├── 04-postgres-service.yaml
    └── 05-postgres-statefulset.yaml
```

---

## Completion Checklist

* PostgreSQL pod is Running
* Kubernetes secret auto-created
* Able to connect using psql
* Data persists across restarts
* No secrets exposed in Git or YAML
