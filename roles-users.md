# IAM Setup: Roles and Users Explained

1. Cluster Operator Role (EKSOperatorRole)
WHY:

You need a dedicated role for cluster administration.

Avoids using personal IAM users directly — cleaner, more secure, easier to rotate.

This role is what you’ll assume when running kubectl or Terraform against the cluster.

HOW:

Trust policy allows assumption by your IAM user(s) (e.g., nancy-devops) or a federated identity provider.

Attach policies like AmazonEKSClusterPolicy and AmazonEKSServicePolicy so it can manage clusters and nodegroups.

Map it in aws-auth.yaml to system:masters so Kubernetes recognizes it as an admin.

Result: You log in as nancy-devops, assume EKSOperatorRole, and Kubernetes sees you as a cluster admin.

2. Node Group Role (EKSNodeRole)
WHY:

Worker nodes (EC2 instances) need IAM permissions to pull images, join the cluster, and talk to AWS services.

Without this, nodes can’t register with EKS or fetch containers from ECR.

HOW:

Trust policy allows assumption by EC2 (ec2.amazonaws.com).

Attach policies:

AmazonEKSWorkerNodePolicy → lets nodes join the cluster.

AmazonEC2ContainerRegistryReadOnly → lets nodes pull images from ECR.

AmazonSSMManagedInstanceCore → lets you connect via Session Manager.

Map it in aws-auth.yaml to system:node.

Result: Every EC2 node in your GPU node group automatically assumes this role and registers with the cluster.

3. Service Account Roles (IRSA)
WHY:

Pods need fine‑grained IAM permissions (e.g., Prometheus scraping CloudWatch, GPU jobs writing to S3).

You don’t want to give all pods node‑level permissions.

HOW:

Use the cluster’s OIDC provider (https://oidc.eks.us-east-1.amazonaws.com/id/...).

Create IAM roles with trust policies tied to specific Kubernetes service accounts.

Annotate the service account with the role ARN.

Result: Only the pod bound to that service account can assume the role, keeping permissions tight.

4. CI/CD Role
WHY:

Pipelines (GitHub Actions, Jenkins) need to deploy manifests to EKS.

You don’t want to embed long‑lived access keys in CI/CD.

HOW:

Use GitHub OIDC or IAM user trust.

Attach policies like eks:DescribeCluster, eks:UpdateClusterConfig, eks:CreateNodegroup.

Map it in aws-auth.yaml to system:masters or a custom RBAC group.

Result: Pipelines can authenticate securely without static secrets.

🛠 Users vs Roles
IAM Users:

Long‑lived identities (like nancy-devops).

Best for humans logging into AWS console or CLI.

Should have minimal permissions — mainly used to assume roles.

IAM Roles:

Temporary, scoped identities.

Best for workloads (EKS nodes, pods, pipelines).

Permissions are attached via policies, assumed when needed.

👉 Best practice: Keep IAM users lean, and push all real permissions into roles. Users only exist to assume roles.

✅ Bootstrap Flow (WHY/HOW Together)
Cluster creator IAM user (WHY: only identity trusted by default) → apply aws-auth.yaml (HOW: maps EKSOperatorRole + EKSNodeRole).

Switch kubeconfig to EKSOperatorRole (WHY: clean separation, HOW: aws eks update-kubeconfig --role-arn ...).

Nodes join with EKSNodeRole (WHY: EC2 trust, HOW: mapped to system:node).

Add IRSA roles later for workloads (WHY: least privilege, HOW: OIDC trust + service account annotation).

Add CI/CD role if pipelines need cluster access (WHY: automation, HOW: OIDC trust + RBAC mapping).

⚡ Bottom line:

Users exist to authenticate humans.

Roles exist to scope permissions for cluster operators, nodes, pods, and pipelines.

The aws-auth.yaml is the bridge between IAM roles and Kubernetes RBAC.
