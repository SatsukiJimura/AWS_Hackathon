# Deployment Log: NOIM変換ツール(dev)

記録日時: 2026-11-13T06:04:25Z(UTC)

**実行者:** 人間(あなた)。AWS CloudShell(東京リージョン)から CDK を実行した。Kiro は AWS に接続しておらず、`cdk` も実行していない。
**情報源:** 実行者の報告(`deployment-execution-questions.md` の Q1・Q2、およびチャットでの「手順書の ## 5 まで実行完了」)。コマンドの出力そのものは、この記録には取り込んでいない。

## 実行内容(`infra/README.md` の手順)

| 手順 | 内容 | 結果 | 備考 |
|---|---|---|---|
| 1〜2 | CloudShell を開き、`noim-deploy.zip` を展開 | 完了 | 配置先 `~/noim/` |
| 3 | 仮想環境の作成、`pip install -r requirements.txt`、`bootstrap` | 完了 | 実行時に `JSII_RUNTIME_PACKAGE_CACHE=disabled` が必要だった(下記) |
| 4 | `synth` → `diff` → `deploy` | 成功(Q1: 1) | スタック `NoimConverterStack` が `CREATE_COMPLETE`。`BucketName` と `InstanceId` が表示された |
| 5 | S3 の `input/` に CSV を置き、セッションマネージャーで接続して `sudo noim-run` | 成功(Q2: 1) | 「完了」と表示され、S3 の `output/` に `resourceInfo_*.csv` の 8 ファイルができた |

## 途中で起きたこと

| 出来事 | 原因 | 対応 |
|---|---|---|
| `synth` が `Cannot use the jsii package cache` で失敗 | CloudShell のホームディレクトリの権限が広い(mode 0777)ため、jsii がキャッシュの使用を拒否した | `export JSII_RUNTIME_PACKAGE_CACHE=disabled` を実行して解消。`infra/README.md` に追記済み |
| `Node 20 is end-of-life` の警告 | CloudShell の Node が 20 系 | 警告のみで、動作に影響なし |

## 記録しないもの

アカウント ID、ARN、`BucketName`・`InstanceId` の実際の値は、この記録に書いていない(実行者の手元の出力と、AWS コンソールで確認できる)。

## 現在の状態

| 項目 | 状態 |
|---|---|
| スタック `NoimConverterStack` | 作成済み(`CREATE_COMPLETE`) |
| EC2、S3 バケット、IAM ロール、アラーム | 稼働中 |
| 後片付け(`destroy`) | 未実行(出力のダウンロード後、全体の完了後に判断) |

## ロールバック

実施していない(必要が生じなかった)。手順は `operation/deployment-pipeline/rollback-runbook.md`。
