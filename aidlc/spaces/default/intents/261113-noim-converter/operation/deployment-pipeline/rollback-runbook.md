# Rollback Runbook: NOIM変換ツール

実行するのは人間です。作業は CloudShell(東京リージョン)と AWS コンソールで行います。連絡先・エスカレーション先は、個人の PoC のため、デプロイした本人です。

## 判断の目安

| 状況 | 対応 |
|---|---|
| `cdk deploy` が途中で失敗した | **何もしなくてよい。** CloudFormation が自動で、もとの状態に戻す(初回なら作成したものを消す)。原因を見て、直してから `deploy` をやり直す |
| デプロイは成功したが、`sudo noim-run` が失敗する | 下の「A. コードを前の版に戻す」 |
| 作った環境が不要になった・やり直したい | 下の「B. すべて消す」 |
| EC2 に接続できない | 下の「C. 接続できないとき」 |

## A. コードを前の版に戻す

1. Git で、動いていた版に戻す(`git checkout <前のコミット>`、または前の zip を使う)。
2. `infra/` で `. .venv/bin/activate` してから、`npx aws-cdk@2 diff` で差分を確認する。
3. `npx aws-cdk@2 deploy`。EC2 が前の版のコードで作り直される。
4. `sudo noim-run` で確認する。

## B. すべて消す

必要な出力を S3 の `output/` からダウンロードしてから実行する。

```bash
cd ~/noim/infra
. .venv/bin/activate
npx aws-cdk@2 destroy
```

S3 の中身も含めて、作ったものがすべて消える。`cdk bootstrap` で作られたもの(アセット用バケットなど)は消えない。それも消す場合は、CloudFormation コンソールで `CDKToolkit` スタックを削除する(他の CDK プロジェクトがあるなら削除しない)。

## C. 接続できないとき

| 確認すること | 対応 |
|---|---|
| 起動から数分経っているか | 数分待って、もう一度接続する |
| EC2 の状態 | 「実行中」で、ステータスチェックが合格か。失敗なら、A の手順で作り直す |
| 起動時の配置の記録 | 接続できる場合は `sudo cat /var/log/cloud-init-output.log` で、エラーを確認する |
| 権限 | EC2 のロールに `AmazonSSMManagedInstanceCore` が付いているか(CloudFormation の `InstanceRole`) |

## 復旧後の確認

デプロイ後のスモークテスト(`deployment-strategy.md`)を、もう一度行う。

## 事後の振り返り

失敗が続いた場合は、原因・直した内容・防ぐ方法を、短くメモに残す(置き場所は自由。形式は問わない)。
