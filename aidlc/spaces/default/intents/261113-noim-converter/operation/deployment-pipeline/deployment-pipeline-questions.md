# Deployment Pipeline Questions

この段階で新しく聞くことはありませんでした。決める必要のあることは、すでに次の回答と前提で決まっています。

| 確認する内容 | 決まっていること | 根拠 |
|---|---|---|
| デプロイ方式 | Recreate(置き換え) | 常時のサービスではなく、EC2が1台 |
| 環境の昇格(dev → staging → prod) | 環境は1つだけ。昇格なし | `infrastructure-specification.md` |
| 本番の承認フロー | 本番なし。dev は実行者本人が確認(y/n) | PoC |
| ロールバックの手順 | `rollback-runbook.md` | `infra/README.md` の方針に沿う |
| 機能フラグ | 使わない | 常時のサービスではない |
| デプロイを誰が実行するか | 人間(CloudShell)。Kiro は実行しない | あなたの指示 |
| 配置の方法 | CDK のアセット | Infrastructure Design の Q3 |

決めたいことが他にあれば、承認のときに伝えてください。
