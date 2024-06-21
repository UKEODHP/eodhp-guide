# EODHP Development Guide

## Git Flow

To effectively share the dev cluster between developers when making updates to the `eodhp-argocd-deployment` repo we will use the a prescribed form of GitFlow, detailed below.

### Issue Development

1. Feature branches should be created using `git checkout -b feature/EODHP-XXX-my-feature`.
2. Make the required updates to the code base.
3. When ready to push to the dev cluster, merge into the `develop` branch. The dev cluster should normally already be pointing to the `develop` branch.
4. Changes will be pulled into the dev cluster using the GitOps deployment of ArgoCD.
5. Any merge conflicts will need to be resolved on the feature branch you are working on. This can be done by cherry picking the merge resolution back into your feature branch after merge has been resolved.
6. Repeat steps 2-5 until the issue is complete and successfully tested on the dev cluster.
7. At no point should you amend, rebase or in anyway alter the history of the `develop` branch as this can overwrite other developers' intermediate commits. Force pushes are disabled for the `develop` branch.

### Issue Closure

1. Create a Pull request for your issue. All PRs for completed issues should still be feature branch -> main.
    - Do not create the PR for feature branch -> develop.
    - Do not create the PR for develop -> main.
2. Once PR has been accepted:
    - Merge into main.
    - Delete the feature branch on the remote server (and locally).
3. Merge main -> develop. __Do not__ rebase develop on to main as this is potentially destructive to any intermediate commits other developers may have made to `develop`.

## Debugging EKS Nodes

Sometimes it is necessary to access the EKS node EC2 instances via SSH. Predominantly our nodes do not have public IP addresses, but we can connect via a VPC endpoint.

The VPC endpoint service is known as EC2 Instance Connect. The existing endpoints can be viewed through the AWS console by going to VPC > Endpoints (side panel under Virtual private cloud).

EC2 Instance Connect is deployed as part of the eodhp-deploy-supporting-infrastructure repo, and the security groups for each cluster are configured in the eodhp-deploy-infrastucture repo.

The AWS CLI is required to connect to node instances in a private subnet.

### SSH into a Node

You must first push your own public SSH key to the node.

```bash
aws ec2-instance-connect send-ssh-public-key \
  --region eu-west-2 \
  --availability-zone eu-west-2a \
  --instance-id i-a12b3c4e5f6g7h8i9 \
  --instance-os-user ec2-user \
  --ssh-public-key file://~/.ssh/id_rsa.pub
```

Set the region, availability zone, instance-id, os-user and ssh public key as required. The instance ID of the node can be found through the AWS EC2 console.

Once you have pushed your SSH key you can then connect via SSH using the AWS CLI.

```bash
aws ec2-instance-connect ssh --instance-id i-a12b3c4e5f6g7h8i9
```
