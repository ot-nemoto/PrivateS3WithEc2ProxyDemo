# PrivateS3WithEc2ProxyDemo

## 概要

## 構成

## デプロイ

**Properties**

|Name|Type|Default|Description|
|--|--|--|--|
|AMIId|String|ami-0ff21806645c5e492|インスタンスのマシンイメージID|
|InstanceType|String|t2.micro|インスタンスタイプ|
|KeyName|AWS::EC2::KeyPair::KeyName|*require*|キーペア名|
|SourceCidrIp|String|0.0.0.0/0|デモサイトへの接続許可するCIDR|

```sh
aws cloudformation create-stack \
    --stack-name private-s3-with-ec2-proxy-demo \
    --capabilities CAPABILITY_IAM \
    --timeout-in-minutes 10 \
    --parameters ParameterKey=KeyName,ParameterValue=KEY_NAME \
    --template-body file://template.yaml
```

※KeyNameは自身の環境で作成済みのKeyNameを指定（未作成の場合は要作成）

## デモサイトデプロイ

```sh
WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name private-s3-with-ec2-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
    --output text)
echo ${WEBSITE_BUCKET}
  # (e.g.)
  # connect-to-s3-website-with-proxy-de-websitebucket-vx9g7huixqrp

aws s3 cp --content-type text/html index.html s3://${WEBSITE_BUCKET}
  # (e.g.)
  # upload: ./index.html to s3://connect-to-s3-website-with-proxy-de-websitebucket-vx9g7huixqrp/index.html
```

## 使い方

- PublicInstanceへはSSMの**Managed Instances**から接続可能
- PrivateInstanceへはPublicInstanceログイン後、環境構築時に指定したキーペアで接続可能。

  鍵作成とPrivateInstanceへのログイン

  ```sh
  cat <<EOT > key.pem
  -----BEGIN RSA PRIVATE KEY-----
  MIIEpAIBAAKCAQEAxtTC+BRf2xeuiuH4vEBzl1cfqTSzJXhquzY2LYWNNpX0kKzmYW+fSc4vgzkm
  ...
  AMjGg3t/Ml8Cw8uarpXwJLJnsNooX65OMfeEkFYlVl3yiCty1xZMxi8bdS6ZH+B9PphRLw==
  -----END RSA PRIVATE KEY-----
  EOT

  chmod 600 key.pem

  ssh -i key.pem ec2-user@PRIVATE_INSTANCE_PRIVATE_IP
  ```

  **PRIVATE_INSTANCE_PRIVATE_IP**は、環境作成したStackのOutputs **PrivateInstancePrivateIp** から確認

- VPNエンドポイントへのURLはStackのOutputs **WebsiteUrl**、**WebSiteSecureUrl** から確認
  - これらのURLはパブリックサブネットのルーティングでは設定せず、プライベートサブネットからのみ設定しているので、プライベートサブネットからのみ接続可能
  - プライベートサブネットのEC2インスタンスでは、VPNエンドポイントへリダイレクトするようnginxでリバースプロキシを立ち上げている

    **リバースプロキシ**のIPアドレス（プライベートIPアドレス）確認方法

    ```sh
    # ECSクラスタ名／サービス名を取得
    CLUSTER_NAME=$(aws cloudformation describe-stacks \
        --stack-name private-s3-with-ecs-proxy-demo \
        --query 'Stacks[].Outputs[?OutputKey==`EcsClusterName`].OutputValue' \
        --output text)
    SERVICE_NAME=$(aws cloudformation describe-stacks \
        --stack-name private-s3-with-ecs-proxy-demo \
        --query 'Stacks[].Outputs[?OutputKey==`EcsServiceName`].OutputValue' \
        --output text)
    echo ${CLUSTER_NAME}; echo ${SERVICE_NAME}
      # (e.g.)
      # private-s3-with-ecs-proxy-demo
      # private-s3-with-ecs-proxy-demo-EcsService-1QFVRXLQF7Z8F

    # ECSタスク名を取得
    TASK_NAME=$(aws ecs list-tasks \
        --cluster ${CLUSTER_NAME} \
        --service-name ${SERVICE_NAME} \
        --desired-status RUNNING \
        --query 'taskArns' \
        --output text | cut -d/ -f2)
    echo ${TASK_NAME}
      # (e.g.)
      # c7c313e6-0a46-4dee-a0c4-e1939d0b9a23

    # Reverse ProxyのIPプライベートIPアドレスを取得
    REVERS_PROXY_IPv4=$(aws ecs describe-tasks \
        --cluster ${CLUSTER_NAME} \
        --tasks ${TASK_NAME} \
        --query 'tasks[].containers[].networkInterfaces[].privateIpv4Address' \
        --output text)
    echo ${REVERS_PROXY_IPv4}
      # (e.g.)
      # 10.38.128.28

    # S3 Bucket名を取得
    WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
        --stack-name private-s3-with-ecs-proxy-demo \
        --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
        --output text)
    echo ${WEBSITE_BUCKET}
      # (e.g.)
      # private-s3-with-ecs-proxy-demo-websitebucket-1k95wltf29ucf

    # Reverse Proxy URL
    echo "http://${REVERS_PROXY_IPv4}/${WEBSITE_BUCKET}/index.html"
      # (e.g.)
      # http://10.38.128.28/private-s3-with-ecs-proxy-demo-websitebucket-1k95wltf29ucf/index.html
    ```

## クリーンアップ

```sh
WEBSITE_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name private-s3-with-ec2-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteBucket`].OutputValue' \
    --output text)
aws s3 rm s3://${WEBSITE_BUCKET} --recursive

LOGGING_BUCKET=$(aws cloudformation describe-stacks \
    --stack-name private-s3-with-ec2-proxy-demo \
    --query 'Stacks[].Outputs[?OutputKey==`WebsiteLoggingBucket`].OutputValue' \
    --output text)
aws s3 rm s3://${LOGGING_BUCKET} --recursive

aws cloudformation delete-stack \
    --stack-name private-s3-with-ec2-proxy-demo
```
