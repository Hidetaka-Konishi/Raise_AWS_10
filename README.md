# CloudFormationテンプレートの中で扱うAWSリソースのパスワードをSystems Manager Parameter Storeで管理する手順。
1. マネジメントコンソールの検索欄に「Systems Manager」と検索して選択します。
2. 左のサイドバーの「パラメータストア」→「パラメータの作成」をクリックします。
3. 「名前」で名前を設定し、「タイプ」で「安全な文字列」を選択します。「値」で管理してほしいAWSリソースのパスワードを入力します。
4. 一番下までスクロールして、「パラメータを作成」をクリックします。
5. MasterUserPassword: '{{resolve:ssm-secure:[パラメータの名前]:1}}'のようにテンプレートに記載します。

# CloudFormationテンプレートの作成時に注意すること
## EC2
以下の写真のようにDeletionPolicyとUpdateReplacePolicyのキーを作成して、Retain,Delete,Snapshotのどれか一つを値として設定する。

・ Retainはスタックが削除されたときにリソースを保持

・ Deleteはスタックが削除されたときにリソースも削除

・ SnapshotはAmazon RDSとAmazon EBSボリュームのみに適用されてリソースが削除されたときにスナップショットを作成

![スクリーンショット 2023-09-29 191446](https://github.com/Hidetaka-Konishi/Raise_AWS_10/assets/142459457/171fcc0d-0930-4866-82c7-edba67dcba32)


# CloudFormationでスタックを作成する手順
1. マネジメントコンソールから「CloudFormation」と検索し、選択します。
2. 「スタックの作成」をクリックする。
3. 「テンプレートの指定」の項目では、「テンプレートファイルのアップロード」を選択し、「ファイルの選択」をクリックしてテンプレートになるコードのファイルを選択する。
4. 一番下までスクロールして、「次へ」をクリックする。
5. 「スタック名」に名前をつけて、「次へ」→「次へ」→「送信」をクリックする。
