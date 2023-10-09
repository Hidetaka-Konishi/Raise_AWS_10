# CloudFormationテンプレートの中で扱うAWSリソースのパスワードをパラメータストアで管理する手順。
1. マネジメントコンソールの検索欄に「Systems Manager」と検索して選択します。
2. 左のサイドバーの「パラメータストア」→「パラメータの作成」をクリックします。
3. 「名前」で名前を設定し、「タイプ」で「安全な文字列」を選択します。「値」で管理してほしいAWSリソースのパスワードを入力します。
4. 一番下までスクロールして、「パラメータを作成」をクリックします。
5. MasterUserPassword: '{{resolve:ssm-secure:[パラメータの名前]:1}}'のようにテンプレートに記載します。

# CloudFormationテンプレートの作成
## 基礎知識
### 論理ID
プログラミングでいう変数のようなもの。

### !Ref
リソースIDやパラメータの値を取得するために使われるもの。

### !Sub
動的に変化する文字列を扱う際に使うもの。例えば、`!Sub ${Prefix}-s3buket`や`!Sub "raise10-subnet-public2-${EC2Subnet2.AvailabilityZone}"`のようにして利用する。

### !GetAtt
特定のリソースの情報を取得するために使うもの。文法は`!GetAtt [論理ID].[リソース名]`となる。例えば、`!GetAtt EC2Subnet2.AvailabilityZone`とすることで`EC2Subnet2`の`AvailabilityZone`の情報を取得することができる。

### 動的なリソース名をつける方法
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "S3"

Parameters:
  Prefix:
    Type: String
    Description: Fill in the part that hits the prefix to give a unique name.

Resources:
    S3Bucket:
        Type: "AWS::S3::Bucket"
        Properties:
            BucketName: !Sub ${Prefix}-s3buket
            BucketEncryption: 
```
`Parameters:`、`Prefix:`、`Type: String`と記述することで、以下の写真のようにマネジメントコンソールからスタックを作成する際に`Prefix`というタイトルと文字列型で入力する欄が現れる。`Description`はこのパラメータが何を意味するのかを説明している。

![スクリーンショット 2023-10-08 115246](https://github.com/Hidetaka-Konishi/Raise_AWS_10/assets/142459457/843d1c9e-aa47-4b5b-8694-a9e4a6825cf1)

### 動的な値を他のテンプレートでパラメータストアから参照する方法

まず、定義する側のテンプレートで以下を記述する。
```yaml
VPCIDParameter:
    Type: "AWS::SSM::Parameter"
    Properties: 
      Name: !Sub "/${Prefix}/VPC-ID"
      Type: "String"
      Value: !Ref EC2VPC
```
`Name`の値はパラメータストアに保存する文字列で、上記で解説した「動的なリソース名をつける方法」を使って`Prefix`の部分を動的な値にしている。`Value`は参照する側のテンプレートで必要になる情報を取得している。

次に、参照する側のテンプレートで以下を記述する。
```yaml
!Sub "{{resolve:ssm:/${VPCPrefix}/VPC-ID}}"
```
`/${VPCPrefix}/VPC-ID`の部分は定義する側のテンプレートの`/${Prefix}/VPC-ID`と同じになるようにする必要があるので、Parametersを使って定義した側と同じ文字列を指定する必要がある。

## VPC
### AWS::EC2::VPC
#### Value
ここで記述した名前はVPC本体の名前になる。

### AWS::EC2::Subnet
#### MapPublicIpOnLaunch
EC2インスタンスがそのサブネットに配置される際に、パブリックIPアドレスを自動で割り当てるかを指定するもの。
####  AvailabilityZone
`!Sub "${AWS::Region}b"`の`AWS::Region`はスタックが作成されたリージョン名が入る。

### AWS::EC2::RouteTable
AWSが自動的に一つルートテーブルを作成するため、実際に作成されたVPCのルートテーブルの数はテンプレートより一つ多くなる。

### AWS::EC2::Route
特定のルートテーブルにルートを追加するために使用するもの。

### AWS::EC2::VPCGatewayAttachment
VPCにインターネットゲートウェイまたは仮想プライベートゲートウェイを関連づけるためのもの。

## EC2
### AWS::EC2::Instance
#### SubnetId
EC2インスタンスをVPCのどのサブネットに配置するのかを指定する。
#### EbsOptimized
「EBS最適化」のことであり、EC2インスタンスの情報が書かれたページの「ストレージ」で確認することができる。
#### SourceDestCheck
EC2インスタンスの情報が書かれたページの「ネットワークインターフェース」の「送信元/送信先チェック」のこと。
#### SnapshotId
既存のスナップショットIDを使用しないのであれば記述する必要はない。
#### Encrypted
「暗号化済み」のこと。
#### Value
EC2インスタンスの名前を指定する。
#### HibernationOptions
「停止 - 休止動作」のこと。
#### EnclaveOptions
「Enclaves のサポート」のこと。

### AWS::EC2::NetworkInterfaceAttachment
EC2インスタンスにENIをアタッチする際に使用されるもので、`AWS::EC2::Instance`で`NetworkInterfaces`を設定していれば記述する必要はない。
#### NetworkInterfaceId
アタッチするENIのID。ここで指定するENIは動的に変化するので、`!Ref EC2NetworkInterface`のようにして参照させる必要がある。

### AWS::EC2::Volume
#### SnapshotId
`AWS::EC2::Instance`の`BlockDeviceMappings`で`Ebs`の`SnapshotId`を指定した場合は指定する必要はない。

### AWS::EC2::NetworkInterface
`AWS::EC2::Instance`で`NetworkInterfaces`を設定していれば記述する必要はない。

### AWS::EC2::Volume
追加のEBSが必要でなければ記述する必要はない。

### AWS::EC2::VolumeAttachment
追加のEBSが必要でなければ記述する必要はない。

### AWS::EC2::SecurityGroup
#### GroupName
作成済みのセキュリティーグループ名を記述することはできない。
#### IpProtocol
-1はすべてのプロトコルを意味する。

## RDS
### AWS::RDS::DBInstance
#### DeletionPolicy
スタックが削除される際にリソースをどのようにするかを定義する。

Delete：スタックが削除されたときにリソースも削除する。

Retain：スタックが削除されたときにリソースを保持する。手動で削除するまで保持される。

Snapshot：RDSやEBSのスタック削除時にリソースも削除されるが同時にスナップショットを作成する。

#### UpdateReplacePolicy

スタックのリソースが更新されたときにリソースををどのようにするかを定義する。

Delete：更新前のリソースが削除される。

Retain：更新前のリソースが保管される。手動で削除しない限り保管される。

Snapshot：更新前のRDSやEBSのスナップショットを作成する。

#### AllocatedStorage
「ストレージ」のこと。
#### MasterUserPassword
RDSのパスワードを記述する。ただ、そのまま記述することはできないので上記の「CloudFormationテンプレートの中で扱うAWSリソースのパスワードをパラメータストアで管理する手順」に沿ってパスワードを暗号化して管理する必要がある。
#### BackupRetentionPeriod
「自動バックアップ」のことであり、指定する数字はバックアップされる日数を意味する。
#### PreferredMaintenanceWindow
RDSインスタンスの30分間のメンテナンスを行う時間帯を指定する。マネジメントコンソール上ではこの情報を確認することはできない。
#### KmsKeyId
`arn:aws:kms~`の`aws`の部分を削除して、`${AWS::Partition}`に書き換える必要がある。これにより、異なるリージョンでもリソースを利用することができる。
#### MonitoringInterval
RDSインスタンスの情報が書かれたページの「モニタリング」でデータが取得されていない場合は0となる。
#### CACertificateIdentifier
「認証機関」のこと。
#### SourceSecurityGroupOwnerId
セキュリティグループのIDはAWS全体で一意であるため、どのAWSアカウントでスタックをデプロイしても指定したセキュリティグループのIDを動的に使用できるようにするために`AWS::AccountId`と記述する。

### AWS::RDS::DBSubnetGroup
#### DBSubnetGroupName
既に作成済みのDBサブネットグループは記述することができない。

## ALB
### AWS::ElasticLoadBalancingV2::LoadBalancer
#### Subnets
スキームが`Internet-facing`に設定されている場合は、インターネットを利用して通信するのでここで指定するサブネットはパブリックサブネットになる。

### AWS::ElasticLoadBalancingV2::Listener
#### Type
`forward`は転送を意味する。

### AWS::ElasticLoadBalancingV2::TargetGroup
#### HealthCheckEnabled
マネジメントコンソール上では確認することができないが通常はtrueにすることが推奨されている。
#### stickiness.enabled
「維持設定」のこと。
#### stickiness.type
マネジメントコンソール上では確認することができない。ここでは、スティッキネスの種類を指定する。スティッキネスは連続する通信を同じサーバーに送信する機能。`lb_cookie`はALBが自動で生成するcookieを使用してスティッキネスを実現するということ。
#### stickiness.lb_cookie.duration_seconds
ALBが自動で生成するcookieを使用したスティッキネスの有効期限。ここで指定する数字は秒数を意味する。
#### stickiness.app_cookie.duration_seconds
アプリケーションが生成するcookieを使用したスティッキネスの有効期限。

## S3
### AWS::S3::Bucket
#### ServerSideEncryptionByDefault
「デフォルトの暗号化」のこと。
#### SSEAlgorithm
`AES256`を指定することでSSE-S3、`aws:kms`を指定することでSSE-KMSの暗号化タイプになる。

### AWS::S3::StorageLens
#### BucketLevel
情報を取得したいバケットを指定する。{}は何も設定されていないデフォルトの状態で、このデフォルトの状態はすべてのバケットに対して情報を取得できることを意味する。
#### IsEnabled
StorageLensを作成した後に、実際にそれを有効にするのかを指定する。

# CloudFormationでスタックを作成する手順
1. マネジメントコンソールから「CloudFormation」と検索し、選択します。
2. 「スタックの作成」をクリックする。
3. 「テンプレートの指定」の項目では、「テンプレートファイルのアップロード」を選択し、「ファイルの選択」をクリックしてテンプレートになるコードのファイルを選択する。
4. 一番下までスクロールして、「次へ」をクリックする。
5. 「スタック名」に名前をつけて、「次へ」→「次へ」→「送信」をクリックする。
