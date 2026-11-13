# Environment Provisioning Questions

この段階で新しく聞くことはありませんでした。環境の内容は、Infrastructure Design の質問(リージョンとEC2の構成、接続方法、ツールの配置方法、入出力の受け渡し)への回答ですべて決まっています。

決定事項は `aidlc/spaces/default/intents/261113-noim-converter/construction/infrastructure-design/infrastructure-design-questions.md` を参照してください。

また、この段階の通常の確認事項は、環境を実際に作った後の確認(VPC、セキュリティグループ、シークレットの注入、アカウント間の接続)です。環境の作成は人間が行うため、作成後に `infra/README.md` の手順5と `validation-report.md` のスモークテストで確認します。シークレットとアカウント間の接続は、この構成では使いません。
