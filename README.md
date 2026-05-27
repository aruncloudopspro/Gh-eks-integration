# Gh-eks-integration \
This repo contains Github actions and EKS integration guide: \

**Architecture:**
<img width="1610" height="964" alt="image" src="https://github.com/user-attachments/assets/a32df857-d01e-4da6-88a2-0aae5ea6efa8" />

**Flow:** \
Developer pushes code to GitHub \
GitHub Actions workflow starts \
GitHub generates OIDC JWT token \
AWS IAM validates token via OIDC Provider \
GitHub assumes IAM Role using sts:AssumeRoleWithWebIdentity \
Temporary AWS credentials are issued \
Workflow accesses EKS cluster \
Deployments happen using kubectl/helm \

**Prerequisites:** \
AWS account \
Existing GitHub repository \
Existing Amazon Web Services EKS cluster \
kubectl knowledge \
IAM permissions \

**Step 1 — Create EKS Cluster:** \
I am using eksctl to create eks cluster for time being. \
**1) eksctl installation : (Ubuntu )** \
Note: In Redhat this installation is not working \

  Install Eksctl tar file first, \
   curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/ \
   eksctl_$(uname -s)_amd64.tar.gz"     | tar xz -C /tmp \
  Move eksctl binary to bin location \
  mv eksctl /usr/local/bin \
  eksctl --version \
**eksctl installation :(Amazon Linux)** \
1. Here, we’re using the curl command to download the most recent version of the eksctl package into a tar file, \
which we then untar under the tmp directory with the following command.

1) curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/ \
eksctl_$(uname -s)_amd64.tar.gz"  | tar xz -C /tmp \
Installing and untar the eksctl \

2. Data from the untar eksctl package is being transferred to the /usr/local/bin directory. \
sudo mv /tmp/eksctl /usr/local/bin \
eksctl version \

**Command to create EKS cluster:** \
--> eksctl create cluster --name demo --region <region-name> --version <k8s_version> --nodegroup-name <nodegroup-name> \
-- node-type <ec2-type> --nodes <node-count> \
Ex: eksctl create cluster --name demo --region ap-south-1 --version 1.34 --nodegroup-name linux-nodes --node-type t2.micro \
--nodes 2

==> Command to delete cluster: Once your work done, don't forget to delete the resources. \
 --> eksctl delete cluster --region=ap-south-1 --name=demo

**Step 2 — Create GitHub OIDC Provider in AWS**: \
 AWS must trust GitHub's identity provider. \
 Go to: AWS Console → IAM → Identity Providers → Add Provider. \
 
 Provider details: \
 Provider Type ---- OpenID Connect (OIDC is an identity layer on top of OAuth 2.0 and GitHub becomes an Identity Provider \
 Provider URL  ---- https://token.actions.githubusercontent.com (This represents who is issuing the identity token?) \
 Audience      ---- sts.amazonaws.com (Audience means who is this token intended for?) \

 What Happens Internally?
  GitHub generates a JWT token like: \
    { \
  "iss": "https://token.actions.githubusercontent.com", \
  "sub": "repo:arun/demo:ref:refs/heads/main", \
  "aud": "sts.amazonaws.com"
 }

 We can use aws-cli to configure this, \
 aws iam create-open-id-connect-provider --url https://token.actions.githubusercontent.com \
 --client-id-list sts.amazonaws.com --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1

**Step 3 — Create IAM Policy for EKS Access**: \
**Step 4 — Create IAM Role for GitHub Actions**: \
**Step 5 — Map IAM Role to EKS RBAC**: \
  AWS IAM authentication alone is NOT enough. \
  EKS must authorize this IAM role inside Kubernetes. \
  kubectl edit configmap aws-auth -n kube-system and add below lines \
  
  mapRoles: | 
  - rolearn: arn:aws:iam::<ACCOUNT_ID>:role/GitHubActionsEKSRole \
    username: github-actions 
    groups: 
      - system:masters

**Note: system:masters gives admin access. For production, use limited RBAC roles instead.**

**Step 6 — Configure GitHub Actions Workflow**:
Workflow:
  name: Deploy to EKS

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
      - name: Checkout
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
            --name demo-eks

      - name: Verify Cluster Access
        run: kubectl get nodes

      - name: Deploy Application
        run: |
          kubectl apply -f deployment.yaml

**Step 7 — Push Code**
  git add . \
  git commit -m "eks oidc setup" \
  git push origin main

  GitHub Actions will: \
    Generate OIDC token \
    Assume IAM role \
    Connect to EKS \
    Deploy application .

--> To play with kubernetes install kubectl client and execute some commands.
  

