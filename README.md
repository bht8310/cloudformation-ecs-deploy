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
### ルートテーブルの設定
```
aws ec2 describe-route-tables --region ap-northeast-2
aws ec2 replace-route-table-association --association-id {メインのassciation id} --route-table-id {作成したルートテーブル} --region ap-northeast-2
```

### albの作成
```shell
aws cloudformation create-stack --region ap-northeast-2 --stack-name alb --template-body file://./02_alb.yaml
```

### ecrの作成
```shell
aws cloudformation create-stack --region ap-northeast-2 --stack-name ecr --template-body file://./03_ecr.yaml
```

## docker imageの作成 / push
- リポジトリの確認
```shell
aws ecr describe-repositories
```
- docker image(web)の作成
```shell
docker build -t {リポジトリURI}:latest -f docker-build/web/Dockerfile . 
```
- docker image(app)の作成
```shell
docker build -t {リポジトリURI}:latest -f docker-build/app/Dockerfile . 
```

```
- ecrにログイン
```shell
 $(aws ecr get-login --no-include-email --region {AWSリージョン名})
```
- docker push(web/app)
```
docker push {リポジトリURI}:latest
```

### ecs task / service の作成
```shell
aws cloudformation create-stack --stack-name ecs-task --region ap-northeast-2 --template-body file://./04_ecs_task.yaml --capabilities CAPABILITY_NAMED_IAM
```