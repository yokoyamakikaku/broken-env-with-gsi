# GSIに関するエラーの再現

Amplifyでのデプロイ時にGSIに関するエラーが発生し、再デプロイを試みても成功しない状態になった。再度正常にデプロイできるようになることはなく、環境を作り直す必要が出た。

関連するイシューもいくつかあったが解決策が明示されていなかった。そのため、環境を再現する手順をまとめ原因の調査に利用する。

## エラーメッセージ

以下のようなエラーメッセージが出てから再デプロイできなくなった。

```
"Cannot update GSI's properties other than Provisioned Throughput and Contributor Insights Specification. You can create a new GSI with a different name."
```

## 関連するイシュー

- [Error: Cannot update GSI's properties other than Provisioned Throughput and Contributor Insights Specification. You can create a new GSI with a different name.](https://github.com/aws-amplify/amplify-category-api/issues/774)
- [Amplify push got error "Message: Resource is not in the state stackUpdateComplete"](https://github.com/aws-amplify/amplify-category-api/issues/92)


# 環境の再現

エラーの発生には複数条件があるが主要な部分は以下の通りある。

- 十分に多いテーブルとインデックス
- コンソールとローカルからの２回のデプロイ

テーブルとインデックスの数がある一定数を超えるとデプロイの時間が30分以上かかることがある。30分はAmplifyの初期設定でのデプロイのタイムアウトの時間である。
コンソールからのデプロイがタイムアウトした状態でローカルからデプロイを行おうとするとデプロイ中であると判断されデプロイに失敗する。
その後にコンソールから再デプロイを行おうとするとGSIに関するエラーが発生する。

## 開発環境

```
% node -v
v18.16.0
% npm -v
9.5.1
% amplify -v
12.1.1
```

## 依存パッケージのインストール

```
% npm i
```

## Amplify プロジェクトの作成

```
 % amplify init
Note: It is recommended to run this command from the root of your app directory
? Enter a name for the project brokenenvwithgsi
The following configuration will be applied:

Project information
| Name: brokenenvwithgsi
| Environment: dev
| Default editor: Visual Studio Code
| App type: javascript
| Javascript framework: react
| Source Directory Path: src
| Distribution Directory Path: build
| Build Command: npm run-script build
| Start Command: npm run-script start

? Initialize the project with the above configuration? Yes
Using default provider  awscloudformation
? Select the authentication method you want to use: AWS profile

For more information on AWS Profiles, see:
https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-profiles.html

? Please choose the profile you want to use yokoyama-t@mococo
Adding backend environment dev to AWS Amplify app: dl3tvujn21ots

Deployment completed.
Deploying root stack brokenenvwithgsi [ ---------------------------------------- ] 0/4
	amplify-brokenenvwithgsi-dev-… AWS::CloudFormation::Stack     CREATE_IN_PROGRESS             Mon Jul 10 2023 21:00:13…
	DeploymentBucket               AWS::S3::Bucket                CREATE_IN_PROGRESS             Mon Jul 10 2023 21:00:16…
	AuthRole                       AWS::IAM::Role                 CREATE_IN_PROGRESS             Mon Jul 10 2023 21:00:17…
	UnauthRole                     AWS::IAM::Role                 CREATE_IN_PROGRESS             Mon Jul 10 2023 21:00:17…

✔ Help improve Amplify CLI by sharing non sensitive configurations on failures (y/N) · yes
Deployment state saved successfully.
✔ Initialized provider successfully.
✅ Initialized your environment successfully.

Your project has been successfully initialized and connected to the cloud!

Some next steps:
"amplify status" will show you what you've added already and if it's locally configured or deployed
"amplify add <category>" will allow you to add features like user login or a backend API
"amplify push" will build all your local backend resources and provision it in the cloud
"amplify console" to open the Amplify Console and view your project status
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud

Pro tip:
Try "amplify add api" to create a backend API and then "amplify push" to deploy everything
```

### 認証の追加

```
% amplify add auth
Using service: Cognito, provided by: awscloudformation

 The current configured provider is Amazon Cognito.

 Do you want to use the default authentication and security configuration? Default configuration
 Warning: you will not be able to edit these selections.
 How do you want users to be able to sign in? Email
 Do you want to configure advanced settings? No, I am done.
✅ Successfully added auth resource [app-name] locally

✅ Some next steps:
"amplify push" will build all your local backend resources and provision it in the cloud
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud
```

### APIの追加

```
% amplify add auth
Using service: Cognito, provided by: awscloudformation

 The current configured provider is Amazon Cognito.

 Do you want to use the default authentication and security configuration? Default configuration
 Warning: you will not be able to edit these selections.
 How do you want users to be able to sign in? Email
 Do you want to configure advanced settings? No, I am done.
✅ Successfully added auth resource [app name] locally

✅ Some next steps:
"amplify push" will build all your local backend resources and provision it in the cloud
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud

broken-env-with-gsi % amplify push -y
broken-env-with-gsi % amplify add api
? Select from one of the below mentioned services: GraphQL
? Here is the GraphQL API that we will create. Select a setting to edit or continue Authorization modes: API key (default, expiration time: 7 days from now)
? Choose the default authorization type for the API Amazon Cognito User Pool
Use a Cognito user pool configured as a part of this project.
? Configure additional auth types? No
? Here is the GraphQL API that we will create. Select a setting to edit or continue Continue
? Choose a schema template: Blank Schema
✅ GraphQL schema compiled successfully.

Edit your schema at /path/to/project/amplify/backend/api/brokenenvwithgsi/schema.graphql or place .graphql files in a directory at /path/to/project/amplify/backend/api/brokenenvwithgsi/schema
✔ Do you want to edit the schema now? (Y/n) · yes
Edit the file in your editor: /path/to/project/amplify/backend/api/brokenenvwithgsi/schema.graphql
✅ Successfully added resource brokenenvwithgsi locally

✅ Some next steps:
"amplify push" will build all your local backend resources and provision it in the cloud
"amplify publish" will build all your local backend and frontend resources (if you have hosting category added) and provision it in the cloud
```

### スキーマのコピー

スキーマをコピーする

```
% cp ./schemas/schema.0.graphql ./amplify/backend/api/brokenenvwithgsi/schema.graphql
```

## CLIでのデプロイ

```
% amplify push -y
```

## リモートリポジトリへのプッシュ

CI/CDからのデプロイのためにリモートリポジトリを準備し、プッシュする。

```
% git add .
% git commit -m 'init amplify project'
% git add remote origin https://github.com/OWNER/REPOSITORY.git
% git push -u origin main
```

## リポジトリの接続

GitHub などのリポジトリホスティングサービスにリポジトリをアップロードして連携する。
リポジトリを接続して設定し、対象のブランチに変更があった時にデプロイするようにする。

リポジトリを接続にデプロイする。


##　データの作成

以下のような対話に返答しながらデータを作成する

```
% node scripts/create-data/index.mjs
? AWS Profile を選択してください ******
? リージョンを入力してください ap-northeast-1
? UserPool を選択してください ******
? User を選択してください ******(******) (******)
? APIを選択してください ******
? 作成するグループの数を入力してください 20
? 作成するユーザの数を入力してください 200
? 作成する本のカテゴリの数を入力してください 400
? 作成する本の数を入力してください 2000
? 作成する本のコメントの数を入力してください 10000
? 作成するメッセージの数を入力してください 1000
? 作成する予定の数を入力してください 1000
? 作成する記事分類の数を入力してください 20
? 作成する記事の数を入力してください 1000
? 作成する作業の数を入力してください 1000
```

## スキーマの更新と自動デプロイからのデプロイ

```
% cp ./schemas/schema.1.graphql ./amplify/backend/api/brokenenvwithgsi/schema.graphql
```

```
% git add .
% git commit -m 'update schema 0 to 1'
% git push
```

ここでタイムアウトが発生する

##　スキーマの更新とローカルからのデプロイ

スキーマを変更する

```
% cp ./schemas/schema.1.graphql ./amplify/backend/api/brokenenvwithgsi/schema.graphql
```

ローカルからデプロイする。デプロイステータスに関するエラーがでる。

```
% amplify push -y
```

###　deploy-state.json の削除

S3 から対応するバケットから delpoy-state.json を削除する

以降、デプロイステータスのエラーが出るたびにS3のバケットから削除する。

### 再度デプロイする

スキーマを変更せずにデプロイする。エラーが出る

```
% amplify push -y
```

## 別のスキーマをCI/CDからデプロイする

```
% cp ./schemas/schema.2.graphql ./amplify/backend/api/brokenenvwithgsi/schema.graphql
```

```
% git add .
% git commit -m 'update schema 1 to 2'
% git push
```

プッシュ後、デプロイが始まる。

## 別のスキーマでローカルからデプロイする

```
% cp ./schemas/schema.3.graphql ./amplify/backend/api/brokenenvwithgsi/schema.graphql
```

```
% amplify push -y
```
