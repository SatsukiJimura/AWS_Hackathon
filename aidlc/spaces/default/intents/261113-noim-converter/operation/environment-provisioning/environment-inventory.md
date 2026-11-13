# Environment Inventory: NOIM変換ツール(dev)

作成日時: 2026-11-13T05:16:58Z(UTC)

**状態: 未作成(コードのみ)。** 環境の作成(`cdk deploy`)は人間が実行します。このファイルは、デプロイしたときに作られるものの一覧です。実行手順は `infra/README.md`。

## 環境

| 環境 | リージョン | アカウント | スタック | 状態 |
|---|---|---|---|---|
| dev(1つだけ) | ap-northeast-1(東京) | 実行する人の認証情報で決まる | `NoimConverterStack` | 未作成 |

## 作られるリソース(`infra/noim_stack.py`)

| 論理ID | 種類 | 設定 | 備考 |
|---|---|---|---|
| Vpc | VPC | max_azs=1、パブリックサブネット1つ(/24)、NATなし | インターネットゲートウェイは自動で作られる |
| InstanceSg | セキュリティグループ | インバウンドなし、アウトバウンドは TCP 443 のみ | |
| DataBucket | S3 バケット | SSE-S3、パブリックアクセス全遮断、TLS必須、削除時に中身ごと削除 | `input/` と `output/` |
| ToolAsset | CDK アセット(S3) | `noim_converter.py` と `noim/` のみ(除外一覧は `ASSET_EXCLUDES`) | CDK のブートストラップのバケットに置かれる |
| InstanceRole | IAM ロール | `AmazonSSMManagedInstanceCore`、`input/*` の読み取り、`output/*` への書き込み、アセットの読み取り | ワイルドカードの全許可は無し |
| Instance | EC2 | t3.micro、AL2023、IMDSv2必須、ルート gp3 8 GiB 暗号化、ユーザーデータで配置 | コード変更で作り直し |
| StatusCheckFailedAlarm | CloudWatch アラーム | `StatusCheckFailed` が 1分×5回 連続で 0超 | 通知先なし |

## 出力値(デプロイ後に表示)

| 出力 | 内容 |
|---|---|
| BucketName | 入力・出力用の S3 バケット名 |
| InstanceId | EC2 のインスタンスID(コンソールのセッションマネージャーで接続) |

## タグ

全リソースに `Project=noim-converter`、`Environment=dev`。

## シークレットとパラメータ

使いません(アクセスキーは置かず、IAM ロールで権限を与える)。

## 検証済みか

`validation-report.md` を参照。
