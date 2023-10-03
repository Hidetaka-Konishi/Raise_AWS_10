# CloudFormationテンプレートの中で扱うAWSリソースのパスワードをSystems Manager Parameter Storeで管理する手順。
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
`!Ref 論理ID`と指定することで論理IDに書かれている情報を得ることができる。

### 動的なリソース名をつける方法
```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: "S3"

Parameters:
  RandomName:
    Type: String

Resources:
    S3Bucket:
        Type: "AWS::S3::Bucket"
        Properties:
            BucketName: !Sub ${RandomName}-s3buket
            BucketEncryption: 
```
`Parameters:`、`RandomName:`、`Type: String`と記述することで、以下の写真のようにマネジメントコンソールからスタックを作成する際に`RandomName`とタイトルと文字列型で入力する欄が現れる。`BucketName: !Sub ${RandomName}-s3buket`の`${RandomName}-s3buket`の部分はPythonでいう`f"{RandomName}-s3buket"`と同じようなもの。

![スクリーンショット 2023-10-01 113024](https://github.com/Hidetaka-Konishi/Raise_AWS_10/assets/142459457/d473831a-4db8-4c10-ac39-1e9db382a2d6)

### 動的な値を他のテンプレートでパラメータストアから参照する方法

まず、定義する側のテンプレートで以下を記述する。
```yaml
VPCIDParameter:
    Type: "AWS::SSM::Parameter"
    Properties: 
      Name: !Sub "/${RandomName}/VPC-ID"
      Type: "String"
      Value: !Ref EC2VPC
```
`Name`の値はパラメータストアに保存する文字列で、上記で解説した「動的なリソース名をつける方法」を使って`RandomName`の部分を動的な値にしている。`Value`は参照する側のテンプレートで必要になる情報が書かれている論理IDを指定している。

次に、参照する側のテンプレートで以下を記述する。
```yaml
!Sub "{{resolve:ssm:/${VPCName}/VPC-ID}}"
```
`/${VPCName}/VPC-ID`の部分は定義する側のテンプレートの`/${RandomName}/VPC-ID`と同じになるようにする必要があるので、Parametersを使って定義した側と同じ文字列を指定する必要がある。

## VPC
### AWS::EC2::VPCDHCPOptionsAssociationとAWS::EC2::DHCPOptions
`AWS::EC2::VPCDHCPOptionsAssociation`の`DhcpOptionsId`は`!Ref EC2DHCPOptions`のように`AWS::EC2::DHCPOptions`の論理IDを指定しなければならない。

## EC2
### AWS::EC2::Instance
`BlockDeviceMappings`の`SnapshotId`に既存のスナップショットIDを使用しないのであれば記述する必要はない。

### AWS::EC2::NetworkInterfaceAttachment
`AWS::EC2::InstanceでNetworkInterfaces`を設定していれば記述する必要はない。

`AWS::EC2::NetworkInterfaceAttachment`はEC2インスタンスにENIをアタッチする際に使用される。

`NetworkInterfaceId`はアタッチするENIのID。ここで指定するENIは動的に変化するので、`!Ref EC2NetworkInterface`のようにして参照させる必要がある。

### AWS::EC2::Volume
`AWS::EC2::Instance`の`BlockDeviceMappings`で`Ebs`の`SnapshotId`を指定した場合は、`AWS::EC2::Volume`で`SnapshotId`を指定する必要はない。

### AWS::EC2::NetworkInterface
`AWS::EC2::Instance`で`NetworkInterfaces`を設定していれば記述する必要はない。

### AWS::EC2::VolumeとAWS::EC2::VolumeAttachment
追加のEBSが必要でなければ記述する必要はない。

### AWS::EC2::SecurityGroup
#### GroupName
作成済みのセキュリティーグループ名を記述することはできない。

## RDS
### AWS::RDS::DBInstance
#### DeletionPolicy,UpdateReplacePolicy
以下の写真のように`DeletionPolicy`と`UpdateReplacePolicy`のキーを作成して、`Retain`, `Delete`, `Snapshot`のどれか一つを値として設定する。

![スクリーンショット 2023-09-29 191446](https://github.com/Hidetaka-Konishi/Raise_AWS_10/assets/142459457/171fcc0d-0930-4866-82c7-edba67dcba32)

[DeletionPolicy] 

スタックが削除される際にリソースをどのようにするかを定義する。

Delete：スタックが削除されたときにリソースも削除する。

Retain：スタックが削除されたときにリソースを保持する。手動で削除するまで保持される。

Snapshot：RDSやEBSのスタック削除時にリソースも削除されるが同時にスナップショットを作成する。

[UpdateReplacePolicy]

スタックのリソースが更新されたときにリソースををどのようにするかを定義する。

Delete：更新前のリソースが削除される。

Retain：更新前のリソースが保管される。手動で削除しない限り保管される。

Snapshot：更新前のRDSやEBSのスナップショットを作成する。

#### MasterUserPassword
RDSのパスワードを記述する。ただ、そのまま記述することはできないので上記の「CloudFormationテンプレートの中で扱うAWSリソースのパスワードをSystems Manager Parameter Storeで管理する手順」に沿ってパスワードを暗号化して管理する必要がある。

#### KmsKeyId
`arn:aws:kms~`の`aws`の部分を削除して、`${AWS::Partition}`に書き換える必要がある。これにより、異なるリージョンでもリソースを利用することができる。

### AWS::RDS::DBSubnetGroup
#### DBSubnetGroupName
既に作成済みのDBサブネットグループは記述することができない。

# CloudFormationでスタックを作成する手順
1. マネジメントコンソールから「CloudFormation」と検索し、選択します。
2. 「スタックの作成」をクリックする。
3. 「テンプレートの指定」の項目では、「テンプレートファイルのアップロード」を選択し、「ファイルの選択」をクリックしてテンプレートになるコードのファイルを選択する。
4. 一番下までスクロールして、「次へ」をクリックする。
5. 「スタック名」に名前をつけて、「次へ」→「次へ」→「送信」をクリックする。
