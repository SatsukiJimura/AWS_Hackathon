# CI/CD Pipeline: NOIM変換ツール

自動のパイプライン(CodePipeline、GitHub Actions など)は作らない。デプロイは人間が手元からCDKで行う(Q3 と、運用上の前提)。計画も CI Pipeline を実行しない。

## 手順(人間が実行)

1. テストを実行する: `python3 -m unittest discover -s tests -t .`(すべて成功することを確認)
2. CDK の準備(初回のみ): `infra/` で依存関係をインストールし、`cdk bootstrap`(アカウントとリージョンごとに1回)
3. 差分の確認: `cdk synth` と `cdk diff`
4. デプロイ: `cdk deploy`。出力される S3 バケット名、インスタンスID、接続コマンドを控える
5. 動作確認: S3 の `input/` に3種のCSVを置き、SSM で接続して `noim-run` を実行、`output/` に8ファイルができることを確認
6. 後片付け: 使い終わったら `cdk destroy`

## Stage → Gate

| Stage | 内容 | Gate(次に進む条件) |
|---|---|---|
| テスト | 単体テスト 42 件 | すべて成功 |
| synth | CloudFormation テンプレートの生成 | エラーなし |
| diff | 変更内容の確認 | 人が内容を確認 |
| deploy | リソースの作成と、コードの配置 | `CREATE_COMPLETE` |
| 動作確認 | `noim-run` の実行 | 終了ステータス 0、`output/` に 8 ファイル |

## Deployment strategy

| Item | Choice |
|---|---|
| 方式 | Recreate(置き換え)。コードを変えて `cdk deploy` すると、EC2 が新しく作られて古いものが消える |
| 理由 | 常時サービスではなく、人が実行するバッチのため。停止しても影響がない |

## Rollback

| Case | 手順 |
|---|---|
| デプロイが失敗した | CloudFormation が自動でロールバックする。原因を確認して再度 `cdk deploy` |
| コードを戻したい | 前の版に戻して `cdk deploy`(EC2 が前の版のコードで置き換わる) |
| すべて消したい | `cdk destroy`(S3 の中身も消える。必要な出力は先にダウンロードする) |

## Environment promotion

なし(環境は1つだけ)。

## Secrets management

| Item | Choice |
|---|---|
| シークレット | 使わない。EC2 の権限は IAM ロールで与え、アクセスキーは置かない |
| デプロイする人の認証情報 | 人間の手元の AWS 認証情報を使う。リポジトリや S3 には置かない |
