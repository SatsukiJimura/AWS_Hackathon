# Infrastructure Specification: NOIM変換ツール(EC2)

根拠はすべて `infrastructure-design-questions.md` の回答(Q1〜Q4)と、`code-summary.md` の「標準ライブラリのみ・ファイルの読み書きだけ」。NFR設計の成果物はない(express)ため、推奨構成(Well-Architected の最小限)を採った。実際のコードは次の段階(Environment Provisioning)で作る。

## Deployment

| Facet | Choice | Rationale |
|---|---|---|
| Compute model | EC2 を1台(Amazon Linux 2023、t3.micro、東京 ap-northeast-1) | Q1 の回答。実データは数百行で小さくて足りる。変換は人が実行するバッチで、常時のサーバープロセスはない |
| Python | AL2023 標準の `python3`(3.9) | コードは Python 3.8 で動く構文にとどめているため、追加インストール不要 |
| Networking | 専用 VPC(パブリックサブネット1つ、NAT なし)に配置。インバウンドは全て閉じる。アウトバウンドは 443 のみ許可 | Q2 の SSM 接続は、EC2 から AWS へ出る通信だけで成り立つ。NAT、VPC エンドポイントは作らない(PoC。固定費を避ける)。**変更点:** 当初案は既定 VPC だったが、既定 VPC が無いアカウントでも動くようにし、`cdk synth` が認証情報なしでも通るよう、専用 VPC に変更した(コストは変わらない) |
| Access | SSM Session Manager のみ。SSH、キーペア、インバウンドなし | Q2 の回答。IAM の権限で接続する |
| Storage | ルート EBS(gp3、8 GiB、暗号化)。変換用の作業ディレクトリは `/var/noim/` | データはS3に置くので、EC2のディスクは作業領域と一時保管だけ |
| Metadata | IMDSv2 を必須にする | 標準的なセキュリティ設定 |
| Tool delivery | `noim_converter.py` と `noim/` だけを CDK のアセットとして S3 にアップロードし、EC2 が起動時に取得して `/opt/noim/` に展開 | Q3 の回答。アセットから、tests、設計書、`入力ファイル/`、`aidlc/`、`.kiro/` は除く |
| Data exchange | S3 バケットを1つ(`input/` と `output/`) | Q4 の回答。人がアップロード、EC2 が取得して実行、結果を `output/` に戻す |
| Run command | 起動時に `/usr/local/bin/noim-run` を作る。実行すると、`input/` を `/var/noim/input/` に同期 → 変換 → `/var/noim/output/` を `output/` に同期 | 人が1コマンドで実行できるようにする。ツール本体(`noim_converter.py`)は変更しない |
| Environments | 1つだけ(dev 相当)。環境名はCDKのコンテキストで付けられるが、本番相当は作らない | PoC。環境の追加は、必要になった時点で指示する |
| IaC approach | AWS CDK v2(Python)。`infra/` に置き、スタックは1つ(`NoimConverterStack`) | プロジェクトの言語に合わせた。リソースが少なく、スタックを分ける利点がない |
| Execution | `cdk bootstrap` と `cdk deploy` は人間が実行する。Kiro は実行しない | 運用上の前提 |
| Update | コードを変えて `cdk deploy` すると、EC2 は置き換わる(`user_data_causes_replacement`) | 起動時の配置を確実にやり直すため。作業ディレクトリのデータは S3 に残る |
| Resource sizing | t3.micro(2 vCPU、1 GiB)。不足したら t3.small に変更(Q1 の B) | 実データの規模で足りる |
| Tags | `Project=noim-converter`、`Environment=dev` を全リソースに付ける | コストの把握 |

## Infrastructure Services

| Service | Role | Configuration | Notes |
|---|---|---|---|
| Amazon EC2 | 変換の実行環境 | t3.micro、AL2023、パブリックサブネット、パブリック IP あり(インバウンドなし)、IMDSv2 必須 | 停止しても EBS の料金は続く。使わないときは `cdk destroy` かインスタンスの停止 |
| IAM ロール(インスタンスプロファイル) | EC2 の権限 | `AmazonSSMManagedInstanceCore`(SSM 接続用)。S3 バケットの `input/*` を読み、`output/*` に書く。CDK アセットのバケットから、このアセットだけを読む | ワイルドカードの `*:*` は使わない。CDK の `grant_*` を使う |
| セキュリティグループ | 通信の制御 | インバウンドなし、アウトバウンドは TCP 443 のみ | DNS(VPC リゾルバー)はセキュリティグループの対象外のため追加不要 |
| Amazon S3(データ用) | 入力 CSV と出力 CSV の受け渡し | SSE-S3 で暗号化、パブリックアクセスを全て遮断、TLS 必須、バージョニングなし。スタック削除時は中身ごと削除(`removal_policy=DESTROY`、`auto_delete_objects=True`) | 「PoCで後片付けしやすい」ことを優先。重要なデータは置かない。`input/` に機微データを置く場合は、削除の設定を見直す |
| Amazon S3(CDK アセット) | 変換ツールのコードの置き場 | CDK のブートストラップが作るバケット | `cdk bootstrap` が必要(アカウントとリージョンごとに1回) |
| AWS Systems Manager | EC2 への接続 | Session Manager。接続のログは任意(今回は設定しない) | セッションの記録が必要なら、別途指示 |

## Shared Infrastructure

該当なし(Unit は1つだけで、共有するリソースはない)。

## 前提と未確認

| Item | Content |
|---|---|
| 前提 | 実行するユーザーの権限で、`cdk bootstrap` と `cdk deploy`(IAM、EC2、S3、CloudFormation の作成)ができること |
| 概算コスト | 東京の On-Demand で、t3.micro 約 $10/月、EBS 8 GiB 約 $1/月、パブリック IPv4 約 $4/月。合計は 約 $15/月(24時間動かした場合)。S3 は数円程度。単価は目安で、変わることがある。使わないときは `cdk destroy` |
| 未確認 | EC2 のユーザーデータが、AL2023 の `python3`(3.9)で `noim_converter.py` を実行できること。実機で確認する(Deployment Execution で、人が `noim-run` を実行して確認) |
| 未確認 | CDK アセットの除外設定(コードだけを含める)が、意図どおり動くこと。`cdk synth` の結果で確認する |
