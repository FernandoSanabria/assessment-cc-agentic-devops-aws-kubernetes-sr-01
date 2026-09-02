# Infrastructure

CloudFormation, deployable with only the AWS CLI. Terraform is not a
prerequisite on this machine and adding one would make the setup less
reproducible, not more.

Region is `us-west-2` throughout. `ProjectName` defaults to `fsl-rdicidr` and
names every resource.

## Provisioning order

The OIDC stack imports exports from the platform stack, so the order matters.

```bash
# 1. Network, ALB, ECR, ECS cluster, task roles, service (zero tasks)
aws cloudformation deploy \
  --template-file infra/platform.yaml \
  --stack-name fsl-rdicidr-platform \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-west-2

# 2. GitHub OIDC provider and the deploy role
aws cloudformation deploy \
  --template-file infra/github-oidc.yaml \
  --stack-name fsl-rdicidr-github-oidc \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-west-2

# 3. Hand the role ARN to GitHub. This is the only repository secret,
#    and it is an ARN, not a credential.
gh secret set AWS_ROLE_ARN --body "$(aws cloudformation describe-stacks \
  --stack-name fsl-rdicidr-github-oidc --region us-west-2 \
  --query 'Stacks[0].Outputs[?OutputKey==`DeployRoleArn`].OutputValue' \
  --output text)"
```

The first rollout is then the CD Pipeline's job — pushing to `main` builds the
image, pushes it to ECR, registers a task definition revision and scales the
service from 0 to 2.

## Two things to know

**The service is created with `DesiredCount: 0`.** The initial task definition
names an image tag that does not exist yet, so any non-zero count deadlocks
stack creation — ECS cannot pull, the service never stabilises, CloudFormation
waits on it. A stack *update* resets the count to 0, so **re-run the CD
Pipeline after changing `platform.yaml`**.

**The OIDC trust policy accepts two subject formats.** GitHub issues subjects
carrying immutable numeric ids (`owner@<id>/repo@<id>`) for this repository
rather than the documented `owner/repo`. Both are matched, both scoped to this
one repository. If you point these templates at a different repository, update
`GitHubOwnerId` and `GitHubRepoId` — read them from a token claim, or from
`gh api repos/<owner>/<repo> --jq '{owner: .owner.id, repo: .id}'`.

## Teardown

```bash
aws cloudformation delete-stack --stack-name fsl-rdicidr-github-oidc --region us-west-2
# ECR must be empty before its stack will delete
aws ecr batch-delete-image --repository-name fsl-rdicidr --region us-west-2 \
  --image-ids "$(aws ecr list-images --repository-name fsl-rdicidr \
    --region us-west-2 --query 'imageIds[*]' --output json)"
aws cloudformation delete-stack --stack-name fsl-rdicidr-platform --region us-west-2
```

The ALB bills hourly whether or not it serves traffic. Tear down when the
challenge recording is finished.
