# Infrastructure Design Questions

NOIM変換ツールをAWS上のEC2に置くための構成を決めます。CDK(Python)のコードは私が作り、`cdk deploy` はあなたが実行します。
前提(質問しないもの): ネットワークは既定のVPCを使い、EC2へのインバウンド通信は開けません(外向きのみ)。DB、キュー、ロードバランサー、冗長化は作りません(PoC)。

## Q1. [AWSプラットフォームエンジニア] リージョンとEC2の構成

EC2は1台だけ作ります。変換ツールは標準ライブラリだけで動き、実データは数百行なので、小さいインスタンスで足ります。

A. 東京リージョン(ap-northeast-1)、Amazon Linux 2023、t3.micro (Recommended)
B. 東京リージョン(ap-northeast-1)、Amazon Linux 2023、t3.small(入力が大きくなりそうな場合)
C. 東京以外のリージョン(「Other」にリージョン名を書いてください)
X. Other (please specify)

[Answer]: A

## Q2. [セキュリティ担当] EC2への接続方法

EC2に入って変換を実行するための方法です。

A. SSM Session Manager(ポートを開けず、キーペアも不要。IAMの権限で接続する) (Recommended)
B. SSH(キーペアを使い、あなたのIPアドレスからだけ許可する。IPアドレスを「Other」に書いてください)
X. Other (please specify)

[Answer]: A

## Q3. [リリース担当] 変換ツールをEC2へ置く方法

`cdk deploy` のときに、変換ツール(`noim_converter.py` と `noim/`)をEC2へ配置します。

A. CDKのアセットとして一緒にアップロードする(コードがS3に置かれ、EC2が起動時に取得して `/opt/noim` に展開する。リポジトリの公開やURLは不要) (Recommended)
B. EC2がGitリポジトリをcloneする(リポジトリのURLが必要。非公開リポジトリは別途認証が必要)
X. Other (please specify)

[Answer]: A

## Q4. [運用担当] 入力CSVと出力CSVの受け渡し

入力ファイル(3種のCSV)をEC2に渡し、出力(`resourceInfo_*.csv`)を受け取る方法です。

A. S3バケットを1つ作る(`input/` と `output/` の置き場。EC2に読み書きの権限を付ける。あなたがアップロードし、EC2上で取得して実行する。バケットはスタック削除時に中身ごと削除する設定にする) (Recommended)
B. S3は使わず、EC2のディスクに直接置く(SSM Session Managerで接続して、コピー&ペーストなどで置く。ファイルが大きいと手間がかかる)
X. Other (please specify)

[Answer]: A
