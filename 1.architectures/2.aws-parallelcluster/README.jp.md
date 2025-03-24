# AWS ParallelCluster を使用した分散トレーニング

このディレクトリには、AWS ParallelCluster を使用して分散機械学習トレーニングのためのクラスターをデプロイするためのリソースが含まれています。

## 概要

AWS ParallelCluster は、AWS クラウドでハイパフォーマンスコンピューティング（HPC）クラスターを簡単に作成、管理するためのAWSサポートのオープンソースクラスター管理ツールです。機械学習の分散トレーニングワークロードに最適化されたクラスターを迅速にデプロイできます。

## 主な特徴

- **Slurm スケジューラー**: ジョブスケジューリングとリソース管理のための業界標準のツール
- **EFA（Elastic Fabric Adapter）サポート**: 低レイテンシー、高スループットのネットワーキング
- **FSx for Lustre 統合**: 高性能な共有ファイルシステム
- **コンテナサポート**: Pyxis/Enroot を使用したコンテナ化されたワークロードの実行
- **モニタリングとオブザーバビリティ**: Prometheus/Grafana によるクラスターのモニタリング

## ディレクトリ構造

```
2.aws-parallelcluster/
├── deployment-guides/           # デプロイメントガイドとスクリプト
│   ├── post-install-scripts/    # クラスターノードの設定スクリプト
│   ├── templates/               # クラスター設定テンプレート
│   └── create_config.sh         # 設定生成スクリプト
├── templates/                   # CloudFormation テンプレート
│   ├── parallelcluster-prerequisites.yaml      # 前提条件リソース作成用テンプレート
│   └── parallelcluster-prerequisites-p1.yaml   # 代替バージョン
└── utils/                       # ユーティリティスクリプト
    ├── easy-ssh.sh              # SSM を介したヘッドノードへの簡単な接続
    └── pcluster-fetch-config.sh # クラスター設定の取得
```

## デプロイメントガイド

### 前提条件

1. AWS CLI がインストールされ、適切に設定されていること
2. AWS ParallelCluster CLI がインストールされていること
   ```bash
   pip install aws-parallelcluster
   ```
3. jq がインストールされていること
   ```bash
   # Amazon Linux/RHEL
   sudo yum install -y jq
   
   # Ubuntu
   sudo apt-get install -y jq
   
   # macOS
   brew install jq
   ```

### デプロイメントステップ

1. **前提条件スタックのデプロイ**

   まず、必要なネットワークリソース（VPC、サブネット）、セキュリティグループ、FSx for Lustre ファイルシステムを作成します：

   ```bash
   aws cloudformation create-stack \
     --stack-name parallelcluster-prerequisites \
     --template-body file://templates/parallelcluster-prerequisites.yaml \
     --parameters ParameterKey=PrimarySubnetAZ,ParameterValue=us-east-1a \
     --capabilities CAPABILITY_IAM
   ```

   注意: `PrimarySubnetAZ` パラメータを、使用するリージョンの有効なアベイラビリティゾーンに変更してください。

2. **クラスター設定の生成**

   スタックがデプロイされたら、クラスター設定を生成します：

   ```bash
   cd deployment-guides
   ./create_config.sh
   ```

   このスクリプトは CloudFormation スタックから必要な情報を取得し、環境変数を設定します。

3. **クラスターのデプロイ**

   基本的なクラスターをデプロイするには：

   ```bash
   # 基本クラスター
   pcluster create-cluster --cluster-name ml-cluster \
     --cluster-configuration templates/cluster-vanilla.yaml
   
   # モニタリング機能付きクラスター
   pcluster create-cluster --cluster-name ml-cluster-monitoring \
     --cluster-configuration templates/cluster-observability.yaml
   ```

4. **クラスターへの接続**

   クラスターのヘッドノードに接続するには、提供されている `easy-ssh.sh` スクリプトを使用します：

   ```bash
   cd ../utils
   ./easy-ssh.sh ml-cluster
   ```

## 高度な設定

### コンテナサポート（Pyxis/Enroot）

クラスターテンプレートには、Pyxis/Enroot を使用したコンテナ化されたワークロードの実行をサポートするための設定が含まれています。これにより、Docker コンテナを直接 Slurm ジョブとして実行できます。

### モニタリングとオブザーバビリティ

`cluster-observability.yaml` テンプレートを使用すると、Prometheus と Grafana によるモニタリングが設定されます。これには以下が含まれます：

- ノードエクスポーター: システムメトリクスの収集
- DCGM エクスポーター: GPU メトリクスの収集
- Slurm エクスポーター: Slurm クラスターメトリクス
- EFA ノードエクスポーター: EFA 関連メトリクス

### カスタマイズ

クラスター設定をカスタマイズするには、`templates` ディレクトリ内のテンプレートを編集します。主な設定オプションには以下が含まれます：

- インスタンスタイプの変更
- ノード数の調整
- ストレージ設定の変更
- カスタムポストインストールスクリプトの追加

## 技術的詳細

### ネットワーキング

テンプレートは以下を作成します：
- パブリックサブネットとプライベートサブネットを持つ VPC
- インターネットゲートウェイと NAT ゲートウェイ
- S3 エンドポイント（オプション）
- EFA 通信用のセキュリティグループ

### ストレージ

- **FSx for Lustre**: 高性能な並列ファイルシステム（デフォルト 1.2TB、カスタマイズ可能）
- **FSx for OpenZFS**: ホームディレクトリ用（デフォルト 512GB）
- **インスタンスストレージ**: 各コンピュートノードに `/scratch` としてマウントされる NVMe ストレージ

### インスタンスタイプ

デフォルトでは、テンプレートは P5 インスタンス（p5.48xlarge）を使用するように設定されていますが、これは他の GPU インスタンスタイプ（P4d、P4de、G5 など）に変更できます。

## トラブルシューティング

1. **クラスター作成の失敗**
   - CloudWatch Logs でエラーを確認
   - `pcluster describe-cluster --cluster-name <cluster-name>` を実行

2. **ノードの起動問題**
   - Slurm ログを確認: `/var/log/slurmctld.log`
   - クラウドイニット ログを確認: `/var/log/cloud-init-output.log`

3. **ネットワーク接続の問題**
   - セキュリティグループの設定を確認
   - VPC エンドポイントが正しく設定されていることを確認

4. **設定の取得**
   - クラスターの現在の設定を取得するには：
     ```bash
     ./utils/pcluster-fetch-config.sh ml-cluster
     ```

## 専門用語解説

- **AWS ParallelCluster**: HPC クラスターを AWS にデプロイするためのクラスター管理ツール
- **Slurm**: ジョブスケジューリングとリソース管理のためのオープンソースツール
- **EFA (Elastic Fabric Adapter)**: AWS の高性能ネットワークインターフェース、低レイテンシーの通信に最適
- **FSx for Lustre**: 高性能コンピューティング向けの並列ファイルシステム
- **NCCL (NVIDIA Collective Communications Library)**: GPU 間の効率的な通信のためのライブラリ
- **Pyxis/Enroot**: Slurm でコンテナ化されたワークロードを実行するためのプラグイン/ツール
- **ODCR (On-Demand Capacity Reservation)**: 特定のインスタンスタイプのキャパシティを予約するための AWS 機能

## 参考リンク

- [AWS ParallelCluster 公式ドキュメント](https://docs.aws.amazon.com/parallelcluster/)
- [Slurm ワークロードマネージャー](https://slurm.schedmd.com/)
- [FSx for Lustre ドキュメント](https://docs.aws.amazon.com/fsx/latest/LustreGuide/)
- [EFA ドキュメント](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)
