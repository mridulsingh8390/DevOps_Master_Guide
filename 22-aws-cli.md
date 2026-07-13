# AWS CLI + DevOps Core Services Complete Guide
## IAM, EC2, S3, ECR, ECS/EKS Basics for DevOps Engineers

> This file is practical and command-focused.
> It gives a strong AWS DevOps base from CLI perspective.

---

## 1) Why This Toolset Matters

If you work in cloud DevOps, AWS CLI is essential for:
- automation scripts
- CI/CD deployments
- infrastructure operations
- debugging and incident response

Core services to know first:
- IAM (identity/access)
- EC2 (compute)
- S3 (storage/artifacts)
- ECR (container registry)
- ECS/EKS (container orchestration)

---

## 2) Install AWS CLI v2 (Linux)

```bash
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

---

## 3) Initial Configuration

```bash
aws configure
```

Prompts:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `ap-south-1`, `us-east-1`)
- Output format (`json`)

Check caller identity:
```bash
aws sts get-caller-identity
```

> Best practice: prefer temporary credentials/role assumption over long-lived keys.

---

## 4) IAM Essentials

## 4.1 List users/roles
```bash
aws iam list-users
aws iam list-roles
```

## 4.2 Create policy from file
`readonly-s3-policy.json`
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket","s3:GetObject"],
      "Resource": ["arn:aws:s3:::my-bucket","arn:aws:s3:::my-bucket/*"]
    }
  ]
}
```

Create:
```bash
aws iam create-policy \
  --policy-name ReadOnlyS3Policy \
  --policy-document file://readonly-s3-policy.json
```

## 4.3 Attach policy to role
```bash
aws iam attach-role-policy \
  --role-name MyAppRole \
  --policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/ReadOnlyS3Policy
```

---

## 5) EC2 Essentials

## 5.1 List instances
```bash
aws ec2 describe-instances --query "Reservations[].Instances[].{ID:InstanceId,State:State.Name,Type:InstanceType,AZ:Placement.AvailabilityZone}" --output table
```

## 5.2 Start/stop instance
```bash
aws ec2 start-instances --instance-ids i-xxxxxxxx
aws ec2 stop-instances --instance-ids i-xxxxxxxx
```

## 5.3 Create key pair
```bash
aws ec2 create-key-pair --key-name devops-key --query 'KeyMaterial' --output text > devops-key.pem
chmod 400 devops-key.pem
```

---

## 6) S3 Essentials

## 6.1 Create bucket
```bash
aws s3 mb s3://my-devops-artifacts-12345
```

## 6.2 Upload/download/list
```bash
aws s3 cp app.tar.gz s3://my-devops-artifacts-12345/
aws s3 ls s3://my-devops-artifacts-12345/
aws s3 cp s3://my-devops-artifacts-12345/app.tar.gz .
```

## 6.3 Sync directory
```bash
aws s3 sync ./dist s3://my-devops-artifacts-12345/dist
```

---

## 7) ECR (Container Registry)

## 7.1 Create repository
```bash
aws ecr create-repository --repository-name myapp
```

## 7.2 Login Docker to ECR
```bash
aws ecr get-login-password --region <region> | \
docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
```

## 7.3 Tag and push image
```bash
docker tag myapp:1.0 <account-id>.dkr.ecr.<region>.amazonaws.com/myapp:1.0
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/myapp:1.0
```

---

## 8) ECS Basics (if not using Kubernetes)

## 8.1 List clusters/services
```bash
aws ecs list-clusters
aws ecs list-services --cluster my-cluster
```

## 8.2 Update service (new task revision rollout)
```bash
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --force-new-deployment
```

---

## 9) EKS Basics (Kubernetes on AWS)

## 9.1 Update kubeconfig
```bash
aws eks update-kubeconfig --name my-eks-cluster --region <region>
kubectl get nodes
```

## 9.2 Describe cluster
```bash
aws eks describe-cluster --name my-eks-cluster --region <region>
```

---

## 10) CloudWatch Logs and Metrics Basics

## List log groups
```bash
aws logs describe-log-groups
```

## Tail logs
```bash
aws logs tail /aws/ecs/my-service --follow
```

## Put custom metric
```bash
aws cloudwatch put-metric-data \
  --namespace "DevOpsApp" \
  --metric-name "DeploySuccess" \
  --value 1
```

---

## 11) SSM (Session Manager) - SSH-less Ops

## Start session
```bash
aws ssm start-session --target i-xxxxxxxx
```

Why useful:
- No inbound SSH needed
- Better audit trail than raw SSH
- IAM-controlled access

---

## 12) Secrets and Parameters

## 12.1 SSM Parameter Store
```bash
aws ssm put-parameter --name /myapp/dev/db/password --type SecureString --value "StrongPass123!" --overwrite
aws ssm get-parameter --name /myapp/dev/db/password --with-decryption
```

## 12.2 Secrets Manager
```bash
aws secretsmanager create-secret --name myapp/dev/api-key --secret-string "abc123"
aws secretsmanager get-secret-value --secret-id myapp/dev/api-key
```

---

## 13) STS Assume Role (Cross-account / safer automation)

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::<target-account-id>:role/CICDDeployRole \
  --role-session-name ci-session
```

Use returned temporary credentials in pipeline environment.

---

## 14) JMESPath Queries (CLI Superpower)

Example: get instance IDs only
```bash
aws ec2 describe-instances --query "Reservations[].Instances[].InstanceId" --output text
```

Example: show stopped instances
```bash
aws ec2 describe-instances \
  --query "Reservations[].Instances[?State.Name=='stopped'].{ID:InstanceId,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table
```

---

## 15) Security Best Practices

1. Avoid root account usage  
2. Use IAM roles over static keys  
3. Enforce MFA  
4. Rotate credentials regularly  
5. Use least-privilege policies  
6. Store secrets in SSM/Secrets Manager  
7. Enable CloudTrail and GuardDuty  

---

## 16) Troubleshooting

## `AccessDenied`
- Missing IAM permissions
- Wrong role/account context
- SCP restrictions in AWS Organizations

Check identity:
```bash
aws sts get-caller-identity
```

## `ExpiredToken`
- Refresh temporary credentials
- Re-run SSO/assume-role flow

## Region mismatch
- resource exists in different region
```bash
aws configure get region
```

## ECR login failed
- wrong account/region in registry URL
- Docker daemon issues

---

## 17) Practice Tasks

1. Configure AWS CLI and validate identity  
2. Create S3 bucket and upload/download artifact  
3. Create ECR repo and push Docker image  
4. Fetch secret from SSM/Secrets Manager  
5. Connect to EC2 via SSM session manager  
6. Update ECS service or EKS kubeconfig  
7. Write one least-privilege IAM policy and attach to role  

---

## 18) Daily AWS CLI Cheat Sheet

```bash
aws sts get-caller-identity
aws ec2 describe-instances --output table
aws s3 ls
aws ecr describe-repositories
aws logs describe-log-groups
aws eks list-clusters
aws ecs list-clusters
```

---

## Final Notes

- AWS CLI is foundational for cloud automation.
- Build strong IAM habits early (least privilege + short-lived creds).
- Integrate CLI commands into Makefiles and CI pipelines for repeatable ops.