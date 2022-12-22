# VSCodeでAWSの開発環境を整える
## 環境のセットアップ
参考：
* AWS Toolkit for Visual Studio Codeのインストール  
  https://docs.aws.amazon.com/ja_jp/toolkit-for-vscode/latest/userguide/setup-toolkit.html#setup-prereq
* アクセスキーの取得  
  https://docs.aws.amazon.com/toolkit-for-vscode/latest/userguide/obtain-credentials.html  
* AWS Toolkit for Visual Studio Codeを使ってＡＷＳへ接続 
  https://docs.aws.amazon.com/ja_jp/toolkit-for-vscode/latest/userguide/connect.html 
* Lambdaの開発
  https://docs.aws.amazon.com/ja_jp/toolkit-for-vscode/latest/userguide/remote-lambda.html#remote-lambda-prereq

手順としては
1. 事前にAWS管理画面からアクセスキーを取得（IAMからユーザを選択して、対象ユーザの管理画面から取得可能）
2. VSCodeにAWS Toolkit for Visual Studio Codeをインストール。インストールされると左端のバーにＡＷＳのアイコンが表示されるようになる
3. AWSに接続するための認証プロファイルを設定し、AWSへ接続  
   AWSに接続しようとするとどのプロファイルを使用するか選択するように求められるため、このプロファイルを使って、アクセスする環境の切り替えができそう
