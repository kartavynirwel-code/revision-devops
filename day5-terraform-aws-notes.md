# Day 5 — Terraform + AWS (EKS, IRSA) Revision Notes

## 1. EKS via Terraform — Core Resources

### The Building Blocks
```hcl
# 1. VPC (usually via terraform-aws-modules/vpc)
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  # subnets, NAT gateway, tags required by EKS (kubernetes.io/cluster/<name> = shared)
}

# 2. EKS Cluster
resource "aws_eks_cluster" "main" {
  name     = "my-cluster"
  role_arn = aws_iam_role.eks_cluster_role.arn

  vpc_config {
    subnet_ids = module.vpc.private_subnets
  }
}

# 3. IAM Role for the Cluster control plane itself
resource "aws_iam_role" "eks_cluster_role" {
  name = "eks-cluster-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "eks.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}

# 4. Node Group (worker nodes)
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.name
  node_group_name = "main-nodes"
  node_role_arn   = aws_iam_role.eks_node_role.arn
  subnet_ids      = module.vpc.private_subnets

  scaling_config {
    desired_size = 2
    min_size     = 1
    max_size     = 4
  }
}

# 5. IAM Role for worker NODES (different from cluster role above)
resource "aws_iam_role" "eks_node_role" {
  name = "eks-node-role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action = "sts:AssumeRole"
    }]
  })
}
```

### Interview Point — Two Separate IAM Roles
- **Cluster role**: Lets the EKS control plane itself call AWS APIs (manage ENIs, load balancers, etc.) — trusted by `eks.amazonaws.com`.
- **Node role**: Lets EC2 worker nodes join the cluster and pull images (needs `AmazonEKSWorkerNodePolicy`, `AmazonEKS_CNI_Policy`, `AmazonEC2ContainerRegistryReadOnly`) — trusted by `ec2.amazonaws.com`.
- These are commonly confused — know that **cluster role ≠ node role ≠ pod role (IRSA)**, three different trust relationships.

## 2. IRSA — Full Chain (recall this out loud, step by step)

**IRSA = IAM Roles for Service Accounts.** The goal: give a specific Pod AWS permissions **without** using the node's IAM role (which would over-privilege every pod on that node) and without hardcoding AWS access keys.

### The Chain
```
1. EKS Cluster has an OIDC Provider
   → EKS automatically exposes an OIDC issuer URL for the cluster.
   → You register this as an IAM OIDC Identity Provider in AWS
     (resource: aws_iam_openid_connect_provider)

2. Create an IAM Role with a Trust Policy scoped to OIDC
   → Trust policy says: "trust tokens issued by THIS cluster's OIDC provider,
     but ONLY if the token's subject (sub claim) matches a specific
     Kubernetes namespace + service account name"

   Example trust policy condition:
   "Condition": {
     "StringEquals": {
       "<oidc-provider>:sub": "system:serviceaccount:my-namespace:my-service-account"
     }
   }

3. Attach a permissions policy to that IAM Role
   → e.g., S3 read access, DynamoDB access — whatever the pod actually needs

4. Create a Kubernetes ServiceAccount with an annotation
   apiVersion: v1
   kind: ServiceAccount
   metadata:
     name: my-service-account
     namespace: my-namespace
     annotations:
       eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/my-irsa-role

5. Pod uses that ServiceAccount
   spec:
     serviceAccountName: my-service-account

6. Runtime: AssumeRoleWithWebIdentity
   → EKS's Pod Identity webhook injects a projected service account token
     (JWT) + AWS_ROLE_ARN + AWS_WEB_IDENTITY_TOKEN_FILE env vars into the pod
   → AWS SDK inside the pod automatically calls
     sts:AssumeRoleWithWebIdentity using that JWT
   → STS validates the JWT against the OIDC provider, checks the trust
     policy's `sub` condition matches, and if valid, returns temporary
     AWS credentials scoped to the IAM role's permissions
```

### One-Line Summary (interview-ready)
> "IRSA lets a specific Kubernetes ServiceAccount assume an IAM role via OIDC federation — the pod gets short-lived, scoped AWS credentials through `AssumeRoleWithWebIdentity`, instead of inheriting the node's IAM role or using static access keys."

### Why It Matters (security angle)
- **Least privilege at the pod level**, not the node level — two pods on the same node can have completely different AWS permissions.
- **No static credentials** anywhere — tokens are short-lived and auto-rotated by the webhook.
- Terraform-wise, this typically involves: `aws_iam_openid_connect_provider`, `aws_iam_role` (with `assume_role_policy` referencing the OIDC provider), `aws_iam_role_policy_attachment`, and a Kubernetes `service_account` resource (via the Kubernetes/Helm provider) with the annotation.

---

## Quick Self-Test (do this without looking)
1. Name the three distinct IAM roles/trust relationships involved in a typical EKS + IRSA setup, and who assumes each.
2. Walk through the IRSA chain from "OIDC provider registered in AWS" to "pod has AWS credentials" — every step.
3. Why is IRSA more secure than just giving the EC2 node role broad S3/DynamoDB permissions?
4. What Kubernetes object needs the `eks.amazonaws.com/role-arn` annotation, and what does it link together?
