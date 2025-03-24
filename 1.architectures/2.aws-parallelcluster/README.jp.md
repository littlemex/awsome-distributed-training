# AWS ParallelCluster を使用した分散トレーニング リファレンスアーキテクチャ <!-- omit in toc -->

## 1. アーキテクチャ

AWS ParallelCluster のクラスターは、共通のコンポーネントを共有しています：ヘッドノード、コンピュートノード（通常は P または Trn EC2 インスタンスファミリー）、および1つまたは複数の共有ファイルシステム（FSx for Lustre）です。以下では、アーキテクチャ自体とそのデプロイ方法について説明します。その後、これらのテンプレートの重要な要素（または潜在的なミスを避けるために知っておくべきこと）について説明します。

開始するには、このリポジトリをクローンし、このディレクトリに移動します：

```bash
git clone -b geniac https://github.com/aws-samples/awsome-distributed-training.git
cd awsome-distributed-training/1.architectures/2.aws-parallelcluster
```

## 2. 前提条件

クラスターをデプロイする前に、AWS ParallelCluster（PCluster）CLIがインストールされており、後でヘッドノード用のEC2キーペアを生成していることを確認しましょう。PCがインストールされていてキーペアが生成されている場合は、このセクションをスキップして[クラスターのデプロイセクション](#3-クラスターのデプロイ)に進んでください。

### 2.1. EC2キーペアの作成（必要な場合）

EC2キーペアを使用すると、sshまたは[AWS Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html)を通じてクラスターのヘッドノードに接続できます。ここではSSHについて説明します。

[AWSコンソール](https://console.aws.amazon.com/ec2/home?#KeyPairs:)で公開鍵を確認できます。また、SSHディレクトリ（LinuxまたはOSXを使用している場合は`~/.ssh`）で秘密鍵を確認することもできます。

使用できるキーペアがない場合は、以下のコマンドで作成します（[このドキュメント](https://docs.aws.amazon.com/parallelcluster/latest/ug/set-up-keypair.html)を参照）。

```bash
#!/bin/bash

cd ~/.ssh
# AWS CLIを使用してキーペアを作成し、秘密鍵（.pemファイル）を取得
aws ec2 create-key-pair --key-name ${KEYPAIR_NAME} \
                        --query KeyMaterial \
                        --key-type ed25519 \
                        --region $AWS_REGION \
                        --output text > ${KEYPAIR_NAME}.pem

# 上記のコマンドは、現在のディレクトリに秘密鍵も生成します。
# sshクライアントがこの秘密鍵を使用してssh接続を開くことを許可するために、
# 現在のユーザーのみにアクセス権を変更する必要があります。
sudo chmod 600 ${KEYPAIR_NAME}.pem
```

### 2.2. AWS ParallelCluster CLIのインストール

以下のスクリプトを実行して、Python仮想環境にAWS ParallelCluster CLIをインストールし、この環境にアクセスします。

```bash
export VIRTUAL_ENV_PATH=~/pcluster_env # パスは好みに応じて変更可能
export AWS_REGION=ap-northeast-1
export KEYPAIR_NAME=${KEYPAIR_NAME} # 自分のキーペアを使用
```

```bash
#!/bin/bash

# pipと仮想環境モジュールを更新
python3 -m pip install --upgrade pip
python3 -m pip install --user --upgrade virtualenv
python3 -m virtualenv ${VIRTUAL_ENV_PATH} # 仮想環境を作成
source ${VIRTUAL_ENV_PATH}/bin/activate # 環境を有効化
pip3 install awscli # AWS CLIをインストール
pip3 install aws-parallelcluster==3.12.0 # AWS ParallelClusterをインストール
```

[ドキュメント](https://docs.aws.amazon.com/parallelcluster/latest/ug/commands-v3.html)でAWS ParallelClusterコマンドの一覧を確認できます。

### 2.3 コンピュートリソースの確認

進める前に以下の情報が必要です：

* アカウントのODCR（通常はP5またはTrn1）。以下を確認してください：
    * キャパシティのAZ `AZ`
    * CRのインスタンス数 `ODCR_ID`
* （オプションですが推奨）データS3バケットの名前。このバケットは、6ヶ月間のクラスター運用を通じてすべてのデータ/モデルチェックポイントを永続化するために使用されます。デプロイメントについては[Cloudformationテンプレート](https://github.com/aws-samples/awsome-distributed-training/blob/main/1.architectures/0.s3/0.private-bucket.yaml)を参照してください。バケット名は`DATA_BUCKET_NAME`として参照されます。

S3バケットをデプロイするには、このリンクをクリックしてください：

[<kbd> <br> 1-Click Deploy 🚀 <br> </kbd>](https://ap-northeast-1.console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/quickcreate?templateUrl=https://awsome-distributed-training.s3.amazonaws.com/templates/0.private-bucket.yaml&stackName=cluster-data-bucket)

### 2.4 parallelcluster-prerequisitesのデプロイ

このセクションでは、カスタム[_Amazon Virtual Private Cloud_](https://aws.amazon.com/vpc/)（Amazon VPC）ネットワークとセキュリティグループ、およびFSx for Lustreなどのサポートサービスを、`parallelcluster-prerequisites.yaml`というCloudFormationテンプレートを使用してデプロイします。このテンプレートはリージョンに依存せず、ワークロードを実行するために必要なネットワークアーキテクチャを持つVPCを作成できます。

リソースをデプロイするには、以下の手順に従ってください：

1. このリンクをクリックしてCloudFormationにデプロイします：

[<kbd> <br> 1-Click Deploy 🚀 <br> </kbd>](https://ap-northeast-1.console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/quickcreate?templateUrl=https://awsome-distributed-training.s3.amazonaws.com/templates/parallelcluster-prerequisites.yaml&stackName=parallelcluster-prerequisites)

CloudFormationスタックはFSx for Lustre Persistent_2デプロイメントタイプを使用します。Persistent_1デプロイメントタイプを使用する場合は、以下のリンクを使用してください：

[<kbd> <br> 1-Click Deploy 🚀 <br> </kbd>](https://ap-northeast-1.console.aws.amazon.com/cloudformation/home?region=ap-northeast-1#/stacks/quickcreate?templateUrl=https://awsome-distributed-training.s3.amazonaws.com/templates/parallelcluster-prerequisites-p1.yaml&stackName=parallelcluster-prerequisites)

リンクを開き、コンピュートリソースがあるリージョンとアベイラビリティゾーンを指定する必要があります。「サブネットのアベイラビリティゾーン設定」に入力し、スタックを作成します。

![parallelcluster-prerequisites-cfn](../../0.docs/parallelcluster-prerequisites-cfn.png)

### 2.5 Lustreストレージとデータリポジトリアソシエーション（DRA）の設定

このステップでは、S3バケットとFSx Lustreファイルシステムの間に[データリポジトリアソシエーション（DRA）](https://docs.aws.amazon.com/fsx/latest/LustreGuide/create-dra-linked-data-repo.html)を作成します。

```bash
export AWS_REGION=ap-northeast-1
export STACK_ID_VPC=parallelcluster-prerequisites 
export FSX_ID=`aws cloudformation describe-stacks \
    --stack-name ${STACK_ID_VPC} \
    --query 'Stacks[0].Outputs[?OutputKey==\`FSxLustreFilesystemId\`].OutputValue' \
    --region ${AWS_REGION} \
    --output text`
```

注意：`DATA_BUCKET_NAME`は[ステップ0：リソース情報の確認](https://quip-amazon.com/wDrEAxaBEI3A#temp:C:fdV996b34e8ad4e4dc3ac2ef128b)で作成したバケットです。
以下のようにDRAを作成します：

```bash
aws fsx create-data-repository-association \
    --file-system-id ${FSX_ID} \
    --file-system-path "/data" \
    --data-repository-path s3://${DATA_BUCKET_NAME} \
    --s3 AutoImportPolicy='{Events=[NEW,CHANGED,DELETED]},AutoExportPolicy={Events=[NEW,CHANGED,DELETED]}' \
    --batch-import-meta-data-on-create \
    --region ${AWS_REGION}
```

以下のような出力が表示されます：

```json
{
    "Association": {
        "AssociationId": "dra-0295ef8c2a0e78886",
        "ResourceARN": "arn:aws:fsx:ap-northeast-1:483026362307:association/fs-0160ebe1881498442/dra-0295ef8c2a0e78886",
        "FileSystemId": "fs-0160ebe1881498442",
        "Lifecycle": "CREATING",
        "FileSystemPath": "/data",
        "DataRepositoryPath": "s3://genica-cluster-data-483026362307",
        "BatchImportMetaDataOnCreate": true,
        "ImportedFileChunkSize": 1024,
        "S3": {
            "AutoImportPolicy": {
                "Events": [
                    "NEW",
                    "CHANGED",
                    "DELETED"
                ]
            },
            "AutoExportPolicy": {
                "Events": [
                    "NEW",
                    "CHANGED",
                    "DELETED"
                ]
            }
        },
        "Tags": [],
        "CreationTime": "2024-10-22T09:06:57.151000+09:00"
    }
}
```

DRA作成のステータスは以下のように確認できます：

```bash
aws fsx describe-data-repository-associations \
    --filters "Name=file-system-id,Values=${FSX_ID}" --query "Associations[0].Lifecycle" --output text
    --region ${AWS_REGION}
```

出力が`AVAILABLE`になるまで待ちます。AWSコンソールでもDRAのステータスを確認できます。

## 3. クラスターのデプロイ

クラスターを作成するには、[deployment-guides](./deployment-guides)を参照してください。このディレクトリには、以下のパターンのクラスターデプロイメント手順があります：

* [基本クラスターのデプロイ](./deployment-guides/vanilla-pcluster.md)
* [オブザーバビリティスタック付きのPclusterデプロイ](./deployment-guides/vanilla-pcluster.md)

## 4. AWS ParallelClusterの解剖学

![AWS ParallelCluster diagram](../../0.docs/parallelcluster-arch-diagram.png)

### 4.1. コンピュート

コンピュートは以下のように構成されています：

- **ヘッドノード**: ユーザーがジョブを投入するためのログインおよびコントローラーノード。[m5.8xlarge](https://aws.amazon.com/ec2/instance-types/m5/)に設定されています。
- **compute-gpu**: MLトレーニングジョブを実行するためのキュー（またはパーティション）。インスタンスは[p4de.24xlarge](https://aws.amazon.com/ec2/instance-types/p4/)または[trn1.32xlarge](https://aws.amazon.com/ec2/instance-types/trn1/)で、特にLLMや大規模モデルのトレーニングに推奨されます。キューのデフォルトインスタンス数は*4*に設定されており、必要に応じて変更できます。
- **inference-gpu**: 推論ワークロードを実行するためのオプションのキューで、[g5.12xlarge](https://aws.amazon.com/ec2/instance-types/m5/)を使用します。

### 4.2. オンデマンドキャパシティリザベーション（ODCR）

オンデマンドキャパシティリザベーション（ODCR）は、EC2インスタンスを起動・実行することなくキャパシティを予約するためのツールです。ODCRは、`p4d.24xlarge`、`p4de.24xlarge`、`p5.48xlarge`などのキャパシティ制約のあるインスタンスを起動するための**実質的に唯一の方法**です。さらに、これらのインスタンスタイプのCRは通常、ユーザーではなく*AWSによって作成*されます。これは、クラスターのネットワーキング（[セクション4.3](#43-ネットワークefa-elastic-fabric-adapter)）を正しく設定する方法に影響を与えます。

AWS ParallelClusterは、クラスターの設定ファイルで[CapacityReservationId](https://docs.aws.amazon.com/parallelcluster/latest/ug/Scheduling-v3.html#yaml-Scheduling-SlurmQueues-CapacityReservationTarget)を指定することをサポートしています。キャパシティリザベーションを使用する場合は、設定ファイルの`PLACEHOLDER_CAPACITY_RESERVATION_ID`変数をIDに置き換えます（例：`cr-12356790abcd`）。以下のようになります：

```yaml
CapacityReservationTarget:
    CapacityReservationId: cr-12356790abcd
```

複数のODCRがある場合は、それらを[*キャパシティリザベーショングループ*](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-cr-group.html)にグループ化できます。これにより、クラスターの**同じキュー**の一部として複数のODCRからインスタンスを起動できます。

1. まずグループを作成します。これにより、`arn:aws:resource-groups:us-east-2:123456789012:group/MyCRGroup`のようなグループARNが返されます。これは後で使用します。

```bash
aws resource-groups create-group --name MyCRGroup --configuration '{"Type":"AWS::EC2::CapacityReservationPool"}' '{"Type":"AWS::ResourceGroups::Generic", "Parameters": [{"Name": "allowed-resource-types", "Values": ["AWS::EC2::CapacityReservation"]}]}'
```

2. 次に、キャパシティリザベーションをそのグループに追加します：

```bash
aws resource-groups group-resources --group MyCRGroup --resource-arns arn:aws:ec2:sa-east-1:123456789012:capacity-reservation/cr-1234567890abcdef1 arn:aws:ec2:sa-east-1:123456789012:capacity-reservation/cr-54321abcdef567890
```

3. その後、グループをクラスターの設定に以下のように追加します：

```yaml
CapacityReservationTarget:
    CapacityReservationResourceGroupArn: arn:aws:resource-groups:us-east-2:123456789012:group/MyCRGroup
```

### 4.3. ネットワーク：EFA（Elastic Fabric Adapter）

アプリケーションは、分散トレーニング中の強化されたネットワーキングのために[Elastic Fabric Adapter（EFA）](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)を使用します。最適なネットワークレイテンシーを実現するために、インスタンスは`PlacementGroup`フラグを使用するか、ターゲットとなる[オンデマンドキャパシティリザベーション（ODCR）](#42-オンデマンドキャパシティリザベーションodcr)を指定してプレイスメントグループに配置する必要があります。

`p4`または`p5`のターゲットODCRは通常、ユーザーによって作成されないことに注意することが重要です。代わりに、AWSがプレイスメントグループが割り当てられたCRを作成し、そのCRをユーザーに提供（つまり、共有）します。ユーザーは、`p4`または`p5`インスタンスを起動するためにCRを使用する前に、CRを受け入れる必要があります（例：AWSコンソールを通じて）。

AWS支援のターゲットODCRを使用する場合、AWS ParallelClusterの`PlacementGroup`設定を無効にすることを強く推奨します。そうしないと、このプレイスメントグループオプションがODCRに割り当てられたプレイスメントグループと競合し、インスタンス起動の失敗（*Insufficient Capacity Error*（*ICE*））を引き起こす可能性があります。

プレイスメントグループは分散トレーニングにのみ関連し、推論には関係ありません。

AWS支援のCRを持つ`p4`または`p5`の両方の場合、経験則として以下のパラメータを`false`に設定します：

```yaml
PlacementGroup:
  Enabled: false
```

### 4.4. ストレージ

ストレージには3つの種類があります：

- **ローカル**: ヘッドノードとコンピュートノードには`/`にマウントされた200GiBのEBSボリュームがあります。さらに、ヘッドノードには`/apps`にマウントされた200GiBのEBSボリュームがあります。コンピュートノードにはRAID0でストライプ化されたNVMeドライブがあり、`/local_scratch`としてマウントされています。
- **ファイルネットワークストレージ**: ヘッドノードは`/home`と`/apps`をNFSを通じてクラスター全体で共有します。これらのディレクトリはクラスター内のすべてのインスタンスに自動的にマウントされ、同じパスでアクセスできます。`/home`は通常のホームディレクトリで、`/apps`はアプリケーションや共有ファイルを保存できる共有ディレクトリです。データ集約型タスクにはどちらも使用しないでください。
- **高性能ファイルシステム**: [FSx for Lustre](https://docs.aws.amazon.com/fsx/latest/LustreGuide/what-is.html)ファイルシステムは、クラスターのすべてのノードから`/fsx`でアクセスできます。ここにデータセットを保存します。このファイルシステムは4.8TiBにサイズ設定され、1.2GB/sの集約スループットを提供します。サービスの[ドキュメント](https://docs.aws.amazon.com/fsx/latest/LustreGuide/performance.html)に従って、設定ファイルでそのサイズとプロビジョニングされたTBあたりのスループットを変更できます。

### 4.5. アプリケーションとライブラリのインストール

カスタムイメージまたはポストインストールスクリプトを使用して、アプリケーションスタックをインストールできます。

- **カスタムイメージ**: イメージはクラスターを作成する前に事前にビルドする必要があります。ドライバー、カーネルモジュール、または定期的に使用され、ほとんど更新されないライブラリに推奨されます。このオプションは再現性を確保するために推奨されます。カスタムイメージは以下のように使用できます：

```yaml
Image:
  Os: alinux2 #システムタイプ
  CustomAmi: PLACEHOLDER_CUSTOM_AMI_ID #カスタムイメージAMI IDに置き換え
```

カスタムイメージを使用しない場合は、`CustomAmi`フィールドを削除します。

- **ポストインストールスクリプト**: これらのスクリプトはインスタンスの起動時（ヘッド+コンピュート）に実行されます。このオプションはクイックテストに推奨され、インスタンスの起動時間が増加します。ヘッドノードとコンピュートノードのポストインストールスクリプトは`CustomActions`を通じて実行できます。`distributed-training-p4de_postinstall_scripts.yaml`は、このリポジトリのポストインストールスクリプトを使用してコンテナサポートを有効にします。

### 4.6. トラブルシューティング

顧客が直面する一般的な問題は、ポストインストールスクリプトの問題や設定の誤りによるキャパシティへのアクセスの問題です。これは、ParallelClusterがクラスターのデプロイに失敗する原因となる`HeadNodeWaitCondition`として現れる可能性があります。

解決するには、クラスターのロググループでCloudWatchのログを確認するか、さらなるトラブルシューティングのためにリソースを維持するために`--rollback-on-failure false`オプションを使用します。

## ヒントとコツ

### 2.2 クラスターへの接続

[AWS Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html)を介して簡単にクラスターにログインするために、`easy-ssh.sh`スクリプトを含めています。`ml-cluster`がクラスターの名前であると仮定して、以下のように実行できます：

```bash
./easy-ssh.sh ml-cluster
```

このスクリプトには以下の前提条件が必要です：
* JQ: `brew install jq`
* aws cli
* `pcluster` cli
* [Session Manager Plugin](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)

スクリプトを実行すると、以下のような出力が表示されます：

```
Instance Id: i-0096542c11ccb02b5
Os: ubuntu2004
User: ubuntu
Add the following to your ~/.ssh/config to easily connect:

cat <<EOF >> ~/.ssh/config
Host ml-cluster
  User ubuntu
  ProxyCommand sh -c "aws ssm start-session --target i-0095542c11ccb02b5 --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
EOF

Add your ssh keypair and then you can do:

$ ssh ml-cluster

Connecting to ml-cluster...

Starting session with SessionId: ...
root@ip-10-0-24-126:~#
```

1. 公開鍵を`~/.ssh/authorized_keys`ファイルに追加します。

2. 出力からの行をターミナルに貼り付けます。これにより`~/.ssh/config`に追加されます。

```
cat <<EOF >> ~/.ssh/config
Host ml-cluster
  User ubuntu
  ProxyCommand sh -c "aws ssm start-session --target i-0095542c11ccb02b5 --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
EOF
```

3. `ml-cluster`がクラスターの名前であると仮定して、以下のようにsshで接続できます：

```
ssh ml-cluster
```

## 5. 付録

この記事の執筆時点（v3.8.0）では、Parallel Clusterは自動的にSlurmパーティションのソケット数とソケットあたりのコア数を正しく設定しません。複数のGPUを使用したMLトレーニングでは、NCCLがトポロジー検出を行うため、パフォーマンスに悪影響はありません。ただし、シングルGPUトレーニングや推論、またはまれにSlurmのアフィニティ/バインディング機能をサポートする場合には影響がある可能性があります。

[ここ](https://github.com/aws/aws-parallelcluster/issues/5797)の例を使用すると、`p5.48xlarge`は2つのソケットしか持つべきではありませんが、Slurmはパーティションがそれ以上のソケットを持つことを示しています。

```bash
$ ssh p5-st-p5-1 /opt/slurm/sbin/slurmd -C
NodeName=p5-st-p5-1 CPUs=192 Boards=1 SocketsPerBoard=2 CoresPerSocket=48 ThreadsPerCore=2 RealMemory=2047961 UpTime=2-00:08:02

$ sinfo -o '%9P %4c %8z %8X %8Y %8Z'
PARTITION CPUS S:C:T    SOCKETS  CORES    THREADS
p5*       192  192:1:1  192      1        1
```

パーティション設定を修正する場合は、クラスター設定ファイル（例：`distributed-training-*.yaml`ファイル）を編集し、以下のような新しいエントリ`Scheduling` / `SlurmQueues` / `ComputeResources` / `CustomSlurmSettings`を追加します。

```yaml
Scheduling:
  SlurmQueues:
    - Name: compute-gpu
      ...
      ComputeResources:
        - Name: distributed-ml
          InstanceType: p5.48xlarge
          ...
          CustomSlurmSettings:
            Sockets: 2
            CoresPerSocket: 48
```

各インスタンスタイプには独自の`Sockets`と`CoresPerSocket`の値があります。以下は分散トレーニングで一般的に使用されるインスタンスタイプの値です。

| インスタンスタイプ | `Sockets` | `CoresPerSocket` |
| ------------- | --------- | ---------------- |
| p5.48xlarge   | 2         | 48               |
| p4de.24xlarge | 2         | 24               |
| p4d.24xlarge  | 2         | 24               |

他のインスタンスタイプについては、インスタンスを実行して値を確認する必要があります。

## 実装の重要なポイント

### DRAの作成とS3連携

```bash
aws fsx create-data-repository-association \
    --file-system-id ${FSX_ID} \
    --file-system-path "/data" \
    --data-repository-path s3://${DATA_BUCKET_NAME} \
    --s3 AutoImportPolicy='{Events=[NEW,CHANGED,DELETED]},AutoExportPolicy={Events=[NEW,CHANGED,DELETED]}' \
    --batch-import-meta-data-on-create \
    --region ${AWS_REGION}
```

このコマンドの重要なポイント：
- `AutoImportPolicy`と`AutoExportPolicy`の`Events`配列に`NEW,CHANGED,DELETED`を指定することで、S3バケットとFSxファイルシステム間で双方向の自動同期が有効になります
- `batch-import-meta-data-on-create`フラグにより、作成時に既存のS3オブジェクトのメタデータが一括インポートされます
- ファイルシステムパスとデータリポジトリパスのマッピングにより、S3の内容が指定したディレクトリに自動的にマウントされ、透過的にアクセスできるようになります

### クラスター設定ファイルの生成と構造

```bash
./create_config.sh
```

このスクリプトの実行により、以下のような処理が行われます：

1. CloudFormationスタックから必要な情報（VPC ID、サブネットID、セキュリティグループID、FSx IDなど）を取得
2. 環境変数を設定し、テンプレートファイルの変数を置換
3. 最終的なクラスター設定ファイルを生成

生成される設定ファイルの重要な構造：

```yaml
Region: ${AWS_REGION}
Image:
  Os: alinux2
HeadNode:
  InstanceType: m5.8xlarge
  Networking:
    SubnetId: ${HEAD_SUBNET_ID}
  Ssh:
    KeyName: ${KEYPAIR_NAME}
Scheduling:
  Scheduler: slurm
  SlurmQueues:
    - Name: compute-gpu
      ComputeResources:
        - Name: distributed-ml
          InstanceType: p4de.24xlarge
          MinCount: 0
          MaxCount: 4
          Efa:
            Enabled: true
          CapacityReservationTarget:
            CapacityReservationId: ${CAPACITY_RESERVATION_ID}
```

この設定ファイルの重要なポイント：
- `Efa.Enabled: true` - 高性能なネットワーキングのためのEFAを有効化
- `CapacityReservationTarget` - 特定のODCRを指定して確実にキャパシティを確保
- `MinCount: 0` - 動的スケーリングのため、使用していない時はノードを0にスケールダウン

### クラスター作成コマンド

```bash
pcluster create-cluster --cluster-name ml-cluster \
  --cluster-configuration templates/cluster-vanilla.yaml
```

このコマンドの実装ポイント：
- AWS ParallelClusterは内部的にCloudFormationを使用してリソースをプロビジョニング
- クラスター作成プロセスでは以下のステップが実行されます：
  1. ヘッドノードのEC2インスタンスを起動
  2. 必要なIAMロールとポリシーを作成
  3. Slurmスケジューラーを設定
  4. 共有ストレージをマウント
  5. コンピュートノードのAuto Scaling Groupを設定
  6. カスタムアクション（ポストインストールスクリプト）を実行

### ポストインストールスクリプトの実装

```yaml
CustomActions:
  OnNodeConfigured:
    Script: https://raw.githubusercontent.com/aws-samples/aws-parallelcluster-post-install-scripts/main/scripts/install-container-deps.sh
    Args:
      - enroot
      - pyxis
```

このセクションの重要なポイント：
- `OnNodeConfigured` - ノードの設定が完了した後に実行されるスクリプト
- `install-container-deps.sh` - コンテナ依存関係をインストールするスクリプト
  - Enroot - コンテナランタイム（Dockerの代替）
  - Pyxis - SlurmとEnrootの統合プラグイン

スクリプトの実装では以下の処理が行われます：
1. システムパッケージの更新
2. 必要な依存関係のインストール
3. Enrootのビルドとインストール
4. Pyxisのビルドとインストール
5. Slurmの設定ファイルの更新
6. サービスの再起動

### EFAの設定と最適化

EFA（Elastic Fabric Adapter）は分散トレーニングのパフォーマンスに重要な役割を果たします。設定ファイルでは以下のように指定されます：

```yaml
ComputeResources:
  - Name: distributed-ml
    InstanceType: p4de.24xlarge
    Efa:
      Enabled: true
    Networking:
      PlacementGroup:
        Enabled: false
```

EFAの実装ポイント：
- EFAドライバーはAMIに事前インストールされているか、ポストインストールスクリプトでインストールされる
- `PlacementGroup.Enabled: false` - ODCRを使用する場合、プレイスメントグループの競合を避けるために無効化
- EFAはOSレベルで`/opt/amazon/efa`にインストールされ、libfabricとMPIライブラリを提供
- NCCLライブラリはEFAを自動的に検出し、最適な通信パスを選択

### Slurmジョブスクリプトの実装例

```bash
#!/bin/bash
#SBATCH --job-name=distributed-training
#SBATCH --nodes=4
#SBATCH --ntasks-per-node=8
#SBATCH --gpus-per-node=8
#SBATCH --exclusive
#SBATCH --partition=compute-gpu

# 環境変数の設定
export NCCL_DEBUG=INFO
export NCCL_IB_DISABLE=0
export NCCL_IB_GID_INDEX=3
export NCCL_SOCKET_IFNAME=^lo,docker
export NCCL_DEBUG_SUBSYS=ALL

# 分散トレーニングコマンドの実行
srun --mpi=pmix_v3 \
     --container-image=nvcr.io/nvidia/pytorch:23.09-py3 \
     --container-mounts=/fsx:/fsx \
     python -m torch.distributed.run \
     --nproc_per_node=8 \
     --nnodes=4 \
     --node_rank=$SLURM_PROCID \
     --master_addr=$(scontrol show hostnames $SLURM_JOB_NODELIST | head -n 1) \
     --master_port=6000 \
     /fsx/train.py
```

このジョブスクリプトの重要なポイント：
- `--nodes=4 --ntasks-per-node=8 --gpus-per-node=8` - リソース割り当ての指定
- `NCCL_*` 環境変数 - NCCLライブラリの最適化設定
- `srun --mpi=pmix_v3` - PMIxを使用したMPI実行
- `--container-image` - Pyxisを使用したコンテナ実行
- `torch.distributed.run` - PyTorchの分散トレーニングランチャー
