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



