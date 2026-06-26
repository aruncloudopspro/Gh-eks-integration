# 🚀 Deploy to Amazon EKS using GitHub Actions OIDC (Without AWS Access Keys)

<p align="center">

![GitHub Actions](https://img.shields.io/badge/GitHub-Actions-2088FF?style=for-the-badge\&logo=github-actions\&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-EKS-FF9900?style=for-the-badge\&logo=amazonaws\&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-v1.34-326CE5?style=for-the-badge\&logo=kubernetes\&logoColor=white)
![OIDC](https://img.shields.io/badge/OIDC-WebIdentity-success?style=for-the-badge)
![Terraform](https://img.shields.io/badge/CI/CD-GitOps-blue?style=for-the-badge)

</p>

---

# 📌 Overview

This project demonstrates **passwordless authentication** between **GitHub Actions** and **Amazon EKS** using **OpenID Connect (OIDC)**.

Instead of storing long-lived AWS Access Keys inside GitHub Secrets, GitHub dynamically requests a **JWT Token**, which AWS validates through an **OIDC Identity Provider**.

After successful validation, AWS Security Token Service (STS) issues **temporary credentials**, allowing GitHub Actions to securely deploy applications into Amazon EKS.

This is the **AWS recommended production-grade authentication mechanism**.

---

# 🏗️ Architecture

```text
                        Developer
                            │
                            │ Git Push
                            ▼
                  GitHub Repository
                            │
                            ▼
                 GitHub Actions Workflow
                            │
             Generates OIDC JWT Token
                            │
                            ▼
                AWS IAM OIDC Provider
                            │
                    Validates Token
                            │
                            ▼
      STS AssumeRoleWithWebIdentity()
                            │
          Temporary AWS Credentials
                            │
                            ▼
                  Amazon EKS Cluster
                            │
                     kubectl / Helm
                            │
                            ▼
                 Kubernetes Deployment
```

---

# 🔄 Complete Workflow

```text
Developer pushes code
          │
          ▼
GitHub Actions Workflow Starts
          │
          ▼
GitHub generates OIDC JWT Token
          │
          ▼
AWS validates JWT using OIDC Provider
          │
          ▼
STS AssumeRoleWithWebIdentity
          │
          ▼
Temporary Credentials Issued
          │
          ▼
Update kubeconfig
          │
          ▼
kubectl / Helm Deployment
          │
          ▼
Application deployed successfully
```

---

# ⭐ Why OIDC?

Traditional CI/CD pipelines store:

❌ AWS Access Key

❌ AWS Secret Key

inside GitHub Secrets.

Problems:

* Credentials can expire.
* Secrets may leak.
* Manual rotation required.
* Higher security risk.

Using OIDC:

✅ No AWS Access Keys

✅ Temporary Credentials

✅ Least Privilege

✅ Automatic Token Generation

✅ AWS Recommended

---

# 📋 Prerequisites

Before starting this lab, make sure you have:

| Requirement                 | Status |
| --------------------------- | ------ |
| AWS Account                 | ✅      |
| GitHub Repository           | ✅      |
| Existing Amazon EKS Cluster | ✅      |
| IAM Permissions             | ✅      |
| kubectl Installed           | ✅      |
| Basic Kubernetes Knowledge  | ✅      |
| AWS CLI Configured          | ✅      |

---

# 🛠 Step 1 — Create Amazon EKS Cluster

For this demo, we'll create the cluster using **eksctl**.

## Install eksctl (Ubuntu)

```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

---

## Install eksctl (Amazon Linux)

```bash
curl --silent --location \
"https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
| tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin

eksctl version
```

> **Note:** The above installation works for Ubuntu and Amazon Linux. If you're using Red Hat Enterprise Linux, follow the official `eksctl` installation instructions.

---

## Create EKS Cluster

```bash
eksctl create cluster \
--name demo \
--region ap-south-1 \
--version 1.34 \
--nodegroup-name linux-nodes \
--node-type t2.micro \
--nodes 2
```

Cluster creation usually takes **15–20 minutes**.

---

## Verify Cluster

```bash
kubectl get nodes
```

Expected output:

```text
NAME                                          STATUS   ROLES    AGE
ip-192-168-xx-xx.ap-south-1.compute.internal  Ready    <none>
ip-192-168-yy-yy.ap-south-1.compute.internal  Ready    <none>
```

---

## Delete Cluster (Cleanup)

After completing the demo, don't forget to delete the cluster to avoid unnecessary AWS charges.

```bash
eksctl delete cluster \
--region ap-south-1 \
--name demo
```

---

# 🔐 Step 2 — Configure GitHub OIDC Provider in AWS

## 📌 Why Do We Need an OIDC Provider?

Before AWS can trust GitHub Actions, it must know **who is issuing the identity token**.

GitHub acts as the **Identity Provider (IdP)** and AWS IAM must be configured to trust GitHub's OIDC endpoint.

Without an OIDC Provider:

❌ AWS has no way to verify that the JWT token really came from GitHub.

---

## 🏗 Authentication Flow

```text
GitHub Actions
       │
       │ Generates JWT Token
       ▼
https://token.actions.githubusercontent.com
       │
       │ Identity Provider (IdP)
       ▼
AWS IAM OIDC Provider
       │
       │ Validate JWT
       ▼
AWS STS
       │
       ▼
Temporary AWS Credentials
```

---

# What is OIDC?

**OIDC (OpenID Connect)** is an identity layer built on top of OAuth 2.0.

Instead of using usernames and passwords, applications exchange **signed JWT tokens** to prove identity.

In our case:

* **Identity Provider:** GitHub
* **Service Provider:** AWS IAM
* **Authentication Service:** AWS STS

---

# Create OIDC Provider

Navigate to:

```text
AWS Console
    ↓
IAM
    ↓
Identity Providers
    ↓
Add Provider
```

---

## Provider Configuration

| Setting       | Value                                         |
| ------------- | --------------------------------------------- |
| Provider Type | OpenID Connect                                |
| Provider URL  | `https://token.actions.githubusercontent.com` |
| Audience      | `sts.amazonaws.com`                           |

---

## What Do These Values Mean?

### Provider URL

```text
https://token.actions.githubusercontent.com
```

This tells AWS:

> "Only trust identity tokens issued by GitHub."

---

### Audience

```text
sts.amazonaws.com
```

Audience specifies **who the token is intended for**.

GitHub creates a token specifically for AWS STS.

AWS checks:

```text
JWT aud == sts.amazonaws.com ?
```

If Yes ✅

Continue authentication.

Otherwise ❌

Reject request.

---

# Internal Working

When GitHub Actions starts a workflow, GitHub generates a signed JWT.

Example:

```json
{
  "iss": "https://token.actions.githubusercontent.com",
  "sub": "repo:aruncloudopspro/demo:ref:refs/heads/main",
  "aud": "sts.amazonaws.com"
}
```

Meaning:

| Claim | Description            |
| ----- | ---------------------- |
| iss   | Token issued by GitHub |
| sub   | Repository & Branch    |
| aud   | Intended for AWS STS   |

AWS validates all three values before issuing credentials.

---

# Create OIDC Provider Using AWS CLI

```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

---

# Verify OIDC Provider

```bash
aws iam list-open-id-connect-providers
```

Expected Output

```text
arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com
```

---

# Step 3 — Create IAM Policy

The GitHub Actions workflow requires permissions to interact with Amazon EKS.

Below is a sample policy.

> **Note:** In production, follow the principle of least privilege instead of granting broad permissions.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect":"Allow",
      "Action":[
        "eks:DescribeCluster"
      ],
      "Resource":"*"
    }
  ]
}
```

Save the policy as:

```text
GitHubActionsEKSAccessPolicy
```

---

# Step 4 — Create IAM Role

Navigate to:

```text
IAM
   ↓
Roles
   ↓
Create Role
```

Choose:

```text
Trusted Entity

↓

Web Identity
```

Identity Provider

```text
token.actions.githubusercontent.com
```

Audience

```text
sts.amazonaws.com
```

Attach the policy created in the previous step.

Role Name

```text
GitHubActionsEKSRole
```

---

# Trust Relationship

The Trust Policy determines **who can assume this IAM Role**.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:aruncloudopspro/demo:*"
        }
      }
    }
  ]
}
```

---

# Understanding the Trust Policy

| Field     | Purpose                            |
| --------- | ---------------------------------- |
| Principal | GitHub OIDC Provider               |
| Action    | AssumeRoleWithWebIdentity          |
| aud       | Only AWS STS tokens                |
| sub       | Restrict to your GitHub repository |

This prevents other GitHub repositories from assuming your IAM role.

---

# Step 5 — Map IAM Role to Kubernetes RBAC

Authentication alone is **not enough**.

AWS IAM confirms **who you are**.

Kubernetes RBAC decides **what you can do**.

Both are required.

---

## Edit aws-auth ConfigMap

```bash
kubectl edit configmap aws-auth -n kube-system
```

Add the IAM Role:

```yaml
mapRoles: |
  - rolearn: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsEKSRole
    username: github-actions
    groups:
      - system:masters
```

---

## Verify

```bash
kubectl describe configmap aws-auth -n kube-system
```

---

# ⚠ Production Recommendation

The example above uses:

```text
system:masters
```

This grants **cluster-admin** permissions.

For production:

✅ Create dedicated Kubernetes Roles

✅ Create RoleBindings

✅ Grant only the minimum permissions required

Avoid granting full administrator access unless absolutely necessary.

---

# Authentication vs Authorization

```text
GitHub Actions
        │
        ▼
OIDC JWT
        │
        ▼
AWS IAM
(Authentication)
        │
        ▼
Temporary Credentials
        │
        ▼
Amazon EKS
        │
        ▼
Kubernetes RBAC
(Authorization)
        │
        ▼
kubectl apply
```

Remember:

* IAM Authentication answers:

  > "Who are you?"

* Kubernetes RBAC answers:

  > "What are you allowed to do?"

Both must succeed for deployment to work.

---

# 💡 Interview Questions

### Why do we need an OIDC Provider?

Because AWS must verify that the JWT token was genuinely issued by GitHub.

---

### Why is `sts.amazonaws.com` used as the audience?

It tells AWS that the JWT token is intended specifically for AWS Security Token Service.

---

### Why is `aws-auth` required?

IAM authenticates the user, but Kubernetes still needs RBAC rules to authorize actions inside the cluster.

---

### Can another GitHub repository assume this IAM Role?

No.

The Trust Policy restricts access using the `sub` claim, allowing only the specified repository (and optionally branch).

---
# 🚀 Step 6 — Configure GitHub Actions Workflow

Now that AWS trusts GitHub through OIDC and the IAM Role has been mapped to Kubernetes RBAC, it's time to automate deployments.

Whenever code is pushed to the **main** branch:

* ✅ GitHub generates an OIDC JWT token
* ✅ AWS validates the token
* ✅ STS issues temporary AWS credentials
* ✅ kubeconfig is updated
* ✅ kubectl connects to Amazon EKS
* ✅ Application is deployed automatically

---

# 🏗 Deployment Flow

```text
Developer
    │
git push origin main
    │
    ▼
GitHub Repository
    │
    ▼
GitHub Actions
    │
    ▼
Generate OIDC JWT
    │
    ▼
AWS IAM OIDC Provider
    │
Validate Token
    │
    ▼
STS AssumeRoleWithWebIdentity
    │
Temporary Credentials
    │
    ▼
aws eks update-kubeconfig
    │
    ▼
kubectl apply
    │
    ▼
Amazon EKS
```

---

# 📂 Repository Structure

```text
.
├── .github
│   └── workflows
│       └── deploy.yml
│
├── deployment.yaml
├── service.yaml
└── README.md
```

---

# Create GitHub Actions Workflow

Create the following directory:

```text
.github/workflows/
```

Create:

```text
deploy.yml
```

---

# GitHub Actions Workflow

```yaml
name: Deploy to Amazon EKS

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:

  deploy:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout Repository
        uses: actions/checkout@v4

      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsEKSRole
          aws-region: ap-south-1

      - name: Install kubectl
        uses: azure/setup-kubectl@v4

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
          --region ap-south-1 \
          --name demo

      - name: Verify Cluster Access
        run: kubectl get nodes

      - name: Deploy Application
        run: |
          kubectl apply -f deployment.yaml
          kubectl apply -f service.yaml
```

---

# Understanding Every Step

## 1️⃣ Checkout Repository

```yaml
uses: actions/checkout@v4
```

Downloads your GitHub repository into the GitHub Actions runner.

Without this step, your Kubernetes YAML files won't be available.

---

## 2️⃣ Configure AWS Credentials

```yaml
uses: aws-actions/configure-aws-credentials@v4
```

This action:

* Requests an OIDC JWT token from GitHub
* Sends it to AWS STS
* Assumes the IAM Role
* Receives temporary AWS credentials

No AWS Access Keys are required.

---

## 3️⃣ Install kubectl

```yaml
uses: azure/setup-kubectl@v4
```

Installs the Kubernetes CLI.

Without kubectl, GitHub Actions cannot communicate with EKS.

---

## 4️⃣ Update kubeconfig

```bash
aws eks update-kubeconfig \
--region ap-south-1 \
--name demo
```

Downloads the cluster endpoint and authentication configuration.

Creates:

```text
~/.kube/config
```

---

## 5️⃣ Verify Cluster Access

```bash
kubectl get nodes
```

Expected Output

```text
NAME                                            STATUS
ip-192-168-xx-xx.ap-south-1.compute.internal    Ready
ip-192-168-yy-yy.ap-south-1.compute.internal    Ready
```

This confirms GitHub Actions successfully authenticated with EKS.

---

## 6️⃣ Deploy Application

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

Expected Output

```text
deployment.apps/nginx created

service/nginx-service created
```

---

# Step 7 — Push the Code

```bash
git add .

git commit -m "Configure GitHub Actions OIDC"

git push origin main
```

GitHub Actions starts automatically.

---

# View Workflow Execution

Navigate to:

```text
GitHub Repository
      ↓
Actions
      ↓
Deploy to Amazon EKS
```

You should see:

```text
✔ Checkout Repository

✔ Configure AWS Credentials

✔ Install kubectl

✔ Update kubeconfig

✔ Verify Cluster Access

✔ Deploy Application
```

---

# Verify Deployment

Connect to your EKS Cluster.

```bash
kubectl get pods

kubectl get deployments

kubectl get svc
```

Example

```text
NAME                          READY

nginx-deployment              2/2
```

---

# Internal Authentication Flow

```text
GitHub Runner
      │
      │ Generate JWT
      ▼
OIDC Token
      │
      ▼
AWS IAM OIDC Provider
      │
Validate Signature
      │
      ▼
AWS STS
      │
AssumeRoleWithWebIdentity
      │
      ▼
Temporary AWS Credentials
      │
      ▼
Amazon EKS
```

---

# Security Benefits

Traditional Method

```text
GitHub Secrets

AWS_ACCESS_KEY_ID

AWS_SECRET_ACCESS_KEY
```

Problems

❌ Permanent Credentials

❌ Secret Rotation

❌ Credentials Can Leak

---

OIDC Method

```text
GitHub

↓

JWT Token

↓

AWS STS

↓

Temporary Credentials
```

Benefits

✅ No Access Keys

✅ Temporary Credentials

✅ Automatic Rotation

✅ Least Privilege

✅ AWS Recommended

---

# Common Errors

## AccessDenied

Possible Causes

* Incorrect Trust Policy

* Repository name mismatch

* Branch mismatch

---

## Unauthorized

Cause

IAM Role not mapped in

```text
aws-auth ConfigMap
```

---

## kubectl get nodes Fails

Check

```bash
kubectl config current-context
```

Verify kubeconfig is updated correctly.

---

## InvalidIdentityToken

Usually caused by:

* Incorrect OIDC Provider URL

* Wrong Audience

* Trust Policy Condition mismatch

---

# Production Best Practices

✅ Restrict IAM Role to a specific repository

✅ Restrict to specific branches

✅ Avoid using `system:masters`

✅ Use Kubernetes RBAC

✅ Enable CloudTrail

✅ Enable EKS Control Plane Logging

✅ Rotate IAM policies regularly

✅ Use Helm for deployments

✅ Store manifests in Git

✅ Use Argo CD or Flux for GitOps

---

# Cleanup

Delete resources when finished.

```bash
eksctl delete cluster \
--region ap-south-1 \
--name demo
```

---

# 🎯 What You Learned

✅ GitHub OIDC Authentication

✅ AWS IAM OIDC Provider

✅ JWT Token Validation

✅ STS AssumeRoleWithWebIdentity

✅ Temporary AWS Credentials

✅ Amazon EKS Authentication

✅ Kubernetes RBAC Authorization

✅ GitHub Actions CI/CD

✅ Passwordless AWS Authentication

---

# 📚 References

* AWS IAM OIDC Provider
* Amazon EKS Documentation
* GitHub Actions Documentation
* AWS STS Documentation

---

# ⭐ If this repository helped you...

If you found this project useful:

⭐ Star this repository

🍴 Fork it

📢 Share it with others

🎥 Subscribe to **CloudOpspro** for more production-grade AWS & DevOps tutorials.

Happy Learning! 🚀



