# CD Configuration: NOIM変換ツール

自動のパイプライン(CodePipeline、GitHub Actions など)は作りません。デプロイは、人間がAWS CloudShellからCDKで行います(`infra/README.md`)。CI の設定もありません(計画が CI Pipeline を実行しないため)。

## 入力資料

| 資料 | 状態 |
|---|---|
| `infrastructure-specification.md`、`cicd-pipeline.md`(Infrastructure Design) | あり。これに従う |
| CI の設定(`ci-config.md`、`quality-gates.md`) | なし(CI Pipeline は計画外)。テストの合格(Build and Test)を、デプロイ前の条件にする |

## デプロイの流れ(人間が実行)

| 順 | 内容 | 実行場所 | 次に進む条件 |
|---|---|---|---|
| 1 | 単体テスト `python3 -m unittest discover -s tests -t .` | 手元(または CloudShell) | 42件すべて成功 |
| 2 | コードを CloudShell に置く | CloudShell | `infra/` が見える |
| 3 | 依存関係のインストール | CloudShell | `pip install -r requirements.txt` が成功 |
| 4 | `npx aws-cdk@2 bootstrap`(アカウントとリージョンごとに初回のみ) | CloudShell | 完了 |
| 5 | `npx aws-cdk@2 synth` | CloudShell | エラーなし |
| 6 | `npx aws-cdk@2 diff` | CloudShell | 変更内容を人が確認 |
| 7 | `npx aws-cdk@2 deploy` | CloudShell | `CREATE_COMPLETE` |
| 8 | スモークテスト(`sudo noim-run`) | EC2コンソールのセッションマネージャー | `output/` に8ファイル |

## 環境の昇格(promotion)

環境は1つ(dev)だけです。昇格する先はありません。環境を増やす場合は、そのときに指示してください。

## 承認

| 対象 | 承認者 | 方法 |
|---|---|---|
| dev へのデプロイ | デプロイを実行する人 | `diff` の内容を見て、`deploy` の確認(y/n)に `y` |
| 本番 | 該当なし | 本番環境は作らない |

## 機能フラグ

使いません。

## 認証情報とシークレット

| 項目 | 方針 |
|---|---|
| デプロイする人 | AWSコンソールにログインしたユーザーの権限で CloudShell を使う。アクセスキーは作らない |
| EC2 | IAM ロールで S3 と SSM の権限を得る。キーは置かない |
| リポジトリ・S3 | 認証情報を置かない |
