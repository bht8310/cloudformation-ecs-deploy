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

### dns の設定追加
```shell
aws cloudformation create-stack --stack-name ecs-task --region ap-northeast-2 --template-body file://./05_dns.yaml --parameters ParameterKey=Domain,ParameterValue={自分で管理しているドメイン}
```
- 上記設定後、ドメインのdns serverをAWSに設定が必要です。
- 設定完了後、www.{ドメイン名}で構築したサーバへアクセスできます。

### code commit の作成
```shell
aws cloudformation create-stack --stack-name code --region ap-northeast-2 --template-body file://./06_code.yaml
```

### buildの作成
```shell
aws cloudformation create-stack --stack-name build --region ap-northeast-2 --template-body file://./07_build.yaml --capabilities CAPABILITY_NAMED_IAM
```

### deployの作成
```shell
aws cloudformation create-stack --stack-name deploy --region ap-northeast-2 --template-body file://./08_deploy.yaml --capabilities CAPABILITY_NAMED_IAM
```

### deployment groupの作成
- CloudFormationでFargateのBlue/Green deploy用の設定追加がわからなかったため、cliで追加

```shell
aws deploy create-deployment-group --application-name {デプロイアプリケーション名} --deployment-group-name {デプロイグループ名} --service-role-arn {deploy.yamlで作成した出力RoleCodeDeployArnの値} --region ap-northeast-2 --cli-input-json '{
        "loadBalancerInfo": {
            "targetGroupPairInfoList": [
                {
                    "prodTrafficRoute": {
                        "listenerArns": [
                            "{ALBのリスナーのARN}"
                        ]
                    }, 
                    "targetGroups": [
                        {
                            "name": "{ターゲットグループ名}"
                        }, 
                        {
                            "name": "{ターゲットグループ名2}"
                        }
                    ]
                }
            ]
        }, 
        "blueGreenDeploymentConfiguration": {
            "terminateBlueInstancesOnDeploymentSuccess": {
                "action": "TERMINATE", 
                "terminationWaitTimeInMinutes": 5
            }, 
            "deploymentReadyOption": {
                "actionOnTimeout": "CONTINUE_DEPLOYMENT", 
                "waitTimeInMinutes": 0
            }
        }, 
        "deploymentConfigName": "CodeDeployDefault.ECSAllAtOnce", 
        "alarmConfiguration": {
            "ignorePollAlarmFailure": false, 
            "alarms": [], 
            "enabled": false
        }, 
        "ecsServices": [
            {
                "clusterName": "{ECSのクラスタ名}",
                "serviceName": "{ECSのWebのサービス名}"
            }
        ],
        "autoRollbackConfiguration": {
            "enabled": true, 
            "events": [
                "DEPLOYMENT_FAILURE"
            ]
        }, 
        "deploymentStyle": {
            "deploymentType": "BLUE_GREEN", 
            "deploymentOption": "WITH_TRAFFIC_CONTROL"
        },
        "triggerConfigurations": []
}'
```

### pipelineの作成
```shell
aws cloudformation create-stack --stack-name pipeline --region ap-northeast-2 --template-body file://./09_pipeline.yaml --capabilities CAPABILITY_NAMED_IAM
```

### Pipelineの実行
- これでCodeCommitのMasterに変更があると、Pipelineが実行されデプロイまで実行されます。