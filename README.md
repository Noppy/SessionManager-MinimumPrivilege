# SessionManager-MinimumPrivilege
Systems Manager - Session Managerを利用する場合の最小権限の確認



# 検証手順
## (1)事前設定
### (1)-(a) 作業環境の準備
下記を準備します。
* bashが利用可能な環境(LinuxやMacの環境)
* aws-cliのセットアップ
* AdministratorAccessポリシーが付与され実行可能な、aws-cliのProfileの設定

### (1)-(b) gitのclone
```shell
git clone https://github.com/Noppy/SessionManager-MinimumPrivilege.git
cd SessionManager-MinimumPrivilege
```

### (1)-(c) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE=<Fargaeを動かすComputeAccount用のプロファイルを指定>
export REGION=<デプロイ先のリージョン>

#プロファイルの動作テスト
aws --profile ${PROFILE} sts get-caller-identity

```

## (2)Network準備
### (2)-(a) VPC作成
```shell
# ComputeVPC
aws --profile ${PROFILE} --region ${REGION} \
    cloudformation create-stack \
        --stack-name SSMTestVPC \
        --template-body "file://./src/vpc-2az-4subnets.yaml" \
        --parameters "file://./src/SSmTestVpc.json" \
        --capabilities CAPABILITY_IAM ;
```
### (2)-(b) VPCエンドポイント作成
```shell
aws --profile ${PROFILE} --region ${REGION} \
    cloudformation create-stack \
        --stack-name SSMTestVpce \
        --template-body "file://./src/vpce.yaml"
```

## (3)インスタンス￥準備
```shell
KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。
```
```shell
#最新のAmazon Linux2のAMI IDを取得します。
AMIID=$(aws --profile ${PROFILE} --region ${REGION} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;

#Clientインスタンス作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AMIID}"'"
  },
  {
    "ParameterKey": "KeyName",
    "ParameterValue": "'"${KEYNAME}"'"
  }
]'

aws --profile ${PROFILE} --region ${REGION} \
    cloudformation create-stack \
        --stack-name SSMTestInstances \
        --template-body "file://./src/bastion_and_testinstance.yaml" \
        --parameters "${CFN_STACK_PARAMETERS}" \
        --capabilities CAPABILITY_IAM ;
```
