# aws-iam-setup
This repository has information to scope IAM roles for any cloud. 


1. Cluster Operator Role (Admin)
Purpose

Human/administrator access + bootstrapping the cluster + running kubectl + initial CI/CD if needed.

Trust Policy

✔ Allow assumption by IAM users or IdP (AzureAD, Okta, GitHub OIDC, AWS SSO)
✔ Require MFA (good practice)

Permissions

Best practice: avoid broad AWS-managed policies when possible.
AWS managed options:

AmazonEKSClusterPolicy

AmazonEKSServicePolicy

(optional but often needed) AmazonEKSAdminPolicy

(optional) AmazonEC2FullAccess (only required for nodegroup provisioning)

BUT be careful:
AmazonEKSAdminPolicy does NOT include EC2 permissions needed for creating managed node groups.
So if Terraform/GitHub Actions is using this role to provision nodegroups, add:

ec2:Describe*
ec2:CreateTags
autoscaling:*
iam:PassRole (on NodeInstanceRole only)

Mapping

Mapped to system:masters inside aws-auth.

✔ Correction:

system:masters gives FULL Kubernetes admin.
Use sparingly — only the cluster creator and 1–2 operator roles should have this.

2. Node Group Role (Worker Nodes)
Purpose

IAM role attached to EC2 GPU worker nodes (managed or self-managed).

Trust Policy

✔ ec2.amazonaws.com

Permissions

Correct:

AmazonEKSWorkerNodePolicy

AmazonEKS_CNI_Policy (missing from your list — must include this!)

AmazonEC2ContainerRegistryReadOnly

AmazonSSMManagedInstanceCore

Mapping

Mapped to system:node via the NodeInstanceRole ARN.

Important Fix:

❗You must attach AmazonEKS_CNI_Policy or Pod networking will fail
(CNI cannot manage ENIs, IP addresses, or attach network interfaces).

3. IRSA (IAM Roles for Service Accounts)
Purpose

Fine-grained IAM for pods: Prometheus, ALB controller, Velero, GPU inference jobs, S3, CloudWatch, etc.

Trust Policy

✔ The correct OIDC identity provider for your cluster
(format looks valid)

Permissions

✔ Scoped per workload (S3, CloudWatch, ECR, Route53, dynamodb, sqs, etc.)

Mapping

✔ Not included in aws-auth
✔ IRSA handled by OIDC + annotation:

eks.amazonaws.com/role-arn: arn:aws:iam::<account>:role/EKSIRSA_S3Reader

Important Fix:

Any component that touches AWS APIs requires IRSA, not node roles.
This includes:

AWS Load Balancer Controller

Cluster Autoscaler

EBS CSI Driver

Monitoring stacks that send to CloudWatch

These are mandatory IRSA roles — not optional.

4. CI/CD Role (Terraform / GitHub Actions / Jenkins)
Purpose

CI/CD automation to deploy to EKS.

Trust Policy

✔ GitHub OIDC provider
or Jenkins IAM user
or AWS SSO automation principal

Permissions

Minimum for deploying apps (safe):

eks:DescribeCluster
eks:ListClusters
eks:DescribeNodegroup
eks:AccessKubernetesApi


If provisioning infra (Terraform):

cloudformation:*
eks:*
ec2:*
autoscaling:*
iam:PassRole   # restricted to node role only

Mapping

✔ Can map to:

system:masters (risky)

OR a custom RBAC group allowing only namespace-level access
(best practice in regulated environments)

🎯 Clean Naming Convention (Correct & Recommended)
EKSOperatorRole            # human admin
EKSNodeRole                # EC2 nodes
EKSIRSA_<workload>         # IRSA roles
EKSCI_CDDeployRole         # GitHub Actions / Terraform
EKSClusterAutoscalerRole   # required
EKSALBControllerRole       # required
EKSEBSCSIDriverRole        # required

Fix: You MUST include the mandatory controller roles (IRSA)

These are not optional:

ALB Controller

Cluster Autoscaler

EBS CSI Driver

Node Termination Handler (if using Spot GPUs)

Your list mentions “IRSA roles for workloads” but does not highlight that these core controllers MUST have them.

⚡ Final Architecture Summary (Corrected)

You need 2 roles to bootstrap:

1. EKSOperatorRole

Bootstrap + initial cluster admin.

2. EKSNodeRole

Worker nodes.

Then, once the cluster is live, create IRSA roles:

Required IRSA roles you were missing:

✔ EKSIRSA_ALBController
✔ EKSIRSA_EBSCSIDriver
✔ EKSIRSA_ClusterAutoscaler
✔ EKSIRSA_NodeTerminationHandler (critical for GPU Spot!)

These are essential for a real GPU production cluster.
