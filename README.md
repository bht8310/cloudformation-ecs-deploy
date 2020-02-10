# cloudformation-ecs-deploy

CloudFormationを利用して、
VPC,SubnetからFargate、CodeDeployを構築し、CI/CD環境まで整えるサンプルです。

## 作成するリソース
- VPC
- Subnet
- SecurityGroup
- Role
- ALB
- ECR
- ECS Task / ECS Service (Fargate)
- Route 53 Hosted Zone
- CodeCommit
- CodeDeploy
- Code Pipeline
- S3

## 実行手順

### networkの作成
```shell
aws cloudformation create-stack --region ap-northeast-2 --stack-name network --template-body file://./01_network.yaml
```

### albの作成
```shell
aws cloudformation create-stack --region ap-northeast-2 --stack-name alb --template-body file://./02_alb.yaml
```

### ecrの作成
```shell
aws cloudformation create-stack --region ap-northeast-2 --stack-name ecr --template-body file://./03_ecr.yaml
```