# aws-rds-cfn
CloudFormationを用いた既存VPCへのRDS（MySQL）自動構築テンプレート

## 概要（何を作ったか・目的）
CloudFormationを用いて、既存のVPCおよびサブネット上に
RDS（MySQL）を自動構築するテンプレートを作成しました。

IaC（Infrastructure as Code）の基礎理解と、
RDS構築時のセキュリティ設計（PubliclyAccessible、IP制限など）を
学習することを目的としています。

## 構成 / アーキテクチャ

![構成図](images/architecture.png)

## 検証時の構成
※ 本テンプレートは学習目的のため、  
検証時にはRDSをPublic Subnetに配置し、`PubliclyAccessible` を `true` に設定することで、  
外部クライアント（A5M2）からの接続確認を行いました。

EC2は本テンプレートでは作成していませんが、  
本番構成を想定した接続元として構成図に含めています。

本番運用を想定した構成としては、  
RDSをPrivate Subnetに配置し、  
VPC内部のEC2またはSSM経由で接続することを想定しています。

RDSはMulti-AZ構成としており、障害発生時には自動的に
スタンバイAZへフェイルオーバーされます。

## 使用技術

### AWS
- Amazon VPC  
  - 既存VPCを利用
  - Public / Private Subnet を想定した設計
- Amazon RDS (MySQL)
  - MySQL 8.0
  - Multi-AZ 構成（フェイルオーバー対応）
  - DB Subnet Group を利用した AZ 分散
- Amazon EC2（※本テンプレートでは未作成）
  - 本番想定の接続元として設計図に記載
- AWS Secrets Manager
  - RDS のマスターユーザー名 / パスワードを管理
- AWS CloudFormation
  - インフラをコードとして管理（IaC）

### ネットワーク
- Security Group
  - RDS への接続元 IP を制限（検証時は特定 IP のみ許可）

### 設計・その他
- draw.io
  - 構成図（アーキテクチャ図）の作成
- GitHub
  - テンプレートおよび README の管理

## CloudFormation構成の説明

本テンプレートでは、CloudFormation を用いて RDS（MySQL）の構築を自動化しています。

DB の認証情報は AWS Secrets Manager により管理し、テンプレート内に認証情報を平文で記載しない構成としています。

また、DB Subnet Group に複数 AZ のサブネットを指定し、MultiAZ: true を設定することで、高可用性を考慮した RDS 構成を実現しています。

接続制御は Security Group により行い、検証時と本番運用時で切り替え可能なよう、接続元 IP や Public アクセスの可否を Parameters として定義しています。
これにより、検証用途と本番用途でテンプレートを使い分けることなく、パラメータ変更のみで柔軟に構成を切り替え可能な設計としています

## 工夫・学習したポイント

・Parameters を活用することで、異なる環境でも再利用可能な CloudFormation テンプレート設計を意識しました。

・接続元 IP、PubliclyAccessible、DBInstanceClass を Parameters 化し、  
  検証用途と本番用途でテンプレートを分けることなく、パラメータ変更のみで構成を切り替え可能な設計としました。

・Multi-AZ を有効化し、障害発生時にスタンバイインスタンスへ自動フェイルオーバーされる高可用性構成を実装しました。

・DB の認証情報は AWS Secrets Manager で管理し、CloudFormation テンプレート内にパスワードを平文で記載しないことで、セキュリティを考慮した設計としました。

・既存 VPC を利用し、サブネットや Internet Gateway との関係を意識して構築することで、AWS ネットワーク構成への理解を深めました。

・PubliclyAccessible の挙動や Public / Private Subnet の役割を整理することで、本番環境を想定したネットワークおよびセキュリティ設計の考え方を学習しました。

・Outputs を定義し、作成した RDS のエンドポイントや Secret ARN を即座に参照できるようにし、運用や他サービス連携を考慮した設計としました。

・DeletionProtection や DeleteAutomatedBackups を明示的に設定し、環境削除時の挙動を制御できるようにしました。

## 開発中に直面した課題と解決策

### 課題①：RDS に外部クライアントから接続できなかった

#### 原因
・CloudFormation テンプレート内で DB Subnet Group に明示的なサブネット指定を行っていなかったため、Internet Gateway が関連付けられていないサブネットが選択され、通信がタイムアウトしていた。

#### 解決策
・Internet Gateway が関連付けられたサブネットを作成し、DB Subnet Group に対象サブネットを明示的に指定することで、外部クライアントからの接続を可能にした。

### 課題②：環境ごとにテンプレート修正が必要だった

#### 原因
・IP アドレス、DB インスタンスタイプ、PubliclyAccessible の設定を YAML 内に直接記述していたため、  
　検証環境と本番環境で構成を切り替える際に、テンプレート自体の修正が必要だった。

#### 解決策
・Parameters を活用して各設定値を外部化し、  
　テンプレートを変更せずにパラメータ変更のみで環境ごとの構成切り替えが可能な設計に改善した。

## 今後の改善 / 本番想定での構成

・RDSを完全にプライベートサブネットに配置し、アプリケーションサーバー（EC2）または踏み台サーバー経由でのみ接続可能な構成へ変更する。

・RDS への接続を IP アドレス制限ではなく、EC2のSecurity Groupを参照する方式へ変更し、本番環境を想定したアクセス制御へ改善する。

