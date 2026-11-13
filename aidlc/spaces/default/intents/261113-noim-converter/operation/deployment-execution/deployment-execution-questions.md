# Deployment Execution Questions

AWS上での `cdk deploy` と動作確認は、人間(あなた)が実行します。実行手順は `infra/README.md` です。実行した結果を教えてください。結果の記録(deployment-log、smoke-test-results、health-check-report)は、あなたの回答をもとに作ります。

事前確認(データベースの移行、依存サービス、デプロイの時間帯)は、この構成では該当しません(DBなし、依存サービスなし、時間帯の制限なし)。

## Q1. [リリース担当] デプロイ(cdk deploy)の結果

A. 成功した(スタックが `CREATE_COMPLETE`、`BucketName` と `InstanceId` が表示された)
B. 途中で失敗した(「Other」にエラーメッセージを書いてください)
C. まだ実行していない
X. Other (please specify)

[Answer]: A

## Q2. [リリース担当] 動作確認(sudo noim-run)の結果

A. 「完了」と表示され、S3の `output/` に `resourceInfo_*.csv` の8ファイルができた
B. エラーが出た(「Other」にエラーメッセージを書いてください)
C. まだ実行していない
X. Other (please specify)

[Answer]: A
