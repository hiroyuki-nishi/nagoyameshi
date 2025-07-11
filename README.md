# Laravel Nagoyameshi アプリケーション

このリポジトリは、Terraformで構築されたAWSインフラ（ECS Fargate, ALB, RDS など）にデプロイされる Laravel アプリケーションです。

---

## 🚀 デプロイ方法（Terraform連携）

このプロジェクトは、Terraform構成で作成される CodePipeline により、自動的にデプロイされます。

---

### ✅ 必須手順（Terraform側）

Terraformプロジェクト側の `terraform.tfvars` に、このリポジトリの情報を記入してください：

```hcl
github_owner       = "your-github-username"
github_repo        = "laravel-nagoyameshi"
github_oauth_token = "ghp_XXXXXXXXXXXXXXXXXXXX"
github_branch      = "main"
Terraform を init → plan → apply で実行すると、AWS CodePipeline が作成され、
このリポジトリが自動的にデプロイ対象として取り込まれます。
```

## 🐳 ローカル環境での動作確認
ローカルでの動作確認は以下のコマンドで行えます：

```bash
docker compose up --build
```

これにより、Laravel アプリケーションが .env を使って Docker コンテナ上で起動されます。

※ .env ファイルを事前に作成してください（例：.env.example をコピーして編集）

## 🚀 AWS本番デプロイ用のビルドについて
AWS ECS Fargate への本番デプロイでは、以下のDockerfileを使用します：

```bash
Dockerfile.deploy
```

この Dockerfile.deploy は CodeBuild によって使用され、
ECR イメージをビルド → ECS による Laravel アプリの本番環境起動が行われます。

## 📦 プロジェクト構成
Laravel 10.x

MySQL（Amazon RDS）

SSM Parameter Store による環境変数管理（.env は本番環境では不要）

デプロイは GitHub → CodePipeline → CodeBuild → ECR → ECS の流れ

## 📝 注意事項
本番環境の .env は使用されず、SSM によって環境変数が注入されます

ブランチ名は terraform.tfvars で指定した github_branch に合わせてください

push → CodePipeline → デプロイ の自動化を前提としています


---------------------
# ローカルでの構築
- 1. ECRにログイン
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin xxxx.dkr.ecr.ap-northeast-1.amazonaws.com

- 2. Dockerビルド
```
docker build -t laravel-nagoyameshi . -f Dockerfile.deployment

ARM版
docker build --platform linux/amd64 -t laravel-nagoyameshi . -f Dockerfile.deployment
```

- 3. ECRにPUSH
```
docker tag laravel-nagoyameshi:latest xxxxx.dkr.ecr.ap-northeast-1.amazonaws.com/laravel-nagoyameshi:latest
docker push xxxx.dkr.ecr.ap-northeast-1.amazonaws.com/laravel-nagoyameshi:latest
```
  

# ✅ ECS enable-execute-command有効化
aws ecs update-service \
    --cluster larabel-cluster \
    --service prod-laravel-task-service \
    --enable-execute-command
    
    An error occurred (InvalidParameterException) when calling the UpdateService operation: The service couldn't be updated because a valid taskRoleArn is not being used. Specify a valid task role in your task definition and try again.
    
# 404の原因調査
ECS Fargate でデプロイ後、/ へのアクセスが 404 になる場合、主に以下の原因が考えられます。

### 1. ドキュメントルートの設定ミス
Laravel の public ディレクトリが正しくドキュメントルートになっていない場合、/ で 404 になります。
Dockerfile.deployment では、下記のように public をドキュメントルートに変更しています。

```
RUN sed -i 's|/var/www/html|/var/www/html/public|g' /etc/apache2/sites-available/000-default.conf
```

[確認ポイント]
public ディレクトリ配下に index.php が存在しているか
```
/var/www/html/public ではなく /var/www/html/laravel-nagoyameshi/public になっていないか
```

### 2. COPY コマンドのパスミス
Dockerfile で Laravel のソースを /var/www/html/laravel-nagoyameshi にコピーしていますが、
DocumentRoot を /var/www/html/public にしているため、パスがズレている可能性があります。

```
COPY ./laravel-nagoyameshi /var/www/html/laravel-nagoyameshi
WORKDIR /var/www/html/laravel-nagoyameshi
...
RUN sed -i 's|/var/www/html|/var/www/html/public|g' /etc/apache2/sites-available/000-default.conf
```
現状:
この場合、/var/www/html/public には何もないため 404 になります。

### 3. 解決方法
A. Dockerfile の修正案
Laravel の public ディレクトリが /var/www/html/public になるようにコピー先を調整してください。

例1: ルートにコピーする場合
```
COPY ./laravel-nagoyameshi /var/www/html
WORKDIR /var/www/html
```

例2: サブディレクトリを考慮する場合
DocumentRoot を /var/www/html/laravel-nagoyameshi/public に変更する。
```
RUN sed -i 's|/var/www/html|/var/www/html/laravel-nagoyameshi/public|g' /etc/apache2/sites-available/000-default.conf
```

### 4. 調査方法
ECS タスクで bash などで入って、/var/www/html/public や /var/www/html/laravel-nagoyameshi/public に index.php があるか確認
Apache のエラーログ（/var/log/apache2/error.log）を確認
curl コマンドで localhost:80 にアクセスし、レスポンスを確認

### 5. まとめ
Dockerfile の COPY と DocumentRoot の整合性を確認
public ディレクトリに index.php が存在するか確認
必要に応じて Dockerfile を修正し、再ビルド・再デプロイ
もし、どちらのパターンで修正したいか（ルートにコピー or サブディレクトリのまま）ご希望があれば教えてください。
また、ECS タスク内でのディレクトリ構成を確認できる場合は、その情報もいただけるとより正確にアドバイスできます。
ECSタスクなどでFargate内に入って調査する方法について、ステップバイステップで教えてください。

-------

## 1. ECSタスクのコンテナに入る方法（ECS Exec）
AWS ECS Fargate では、ECS Exec 機能を使ってコンテナ内にシェルで入ることができます。
前提条件
- ECS Exec が有効化されていること（タスク定義・サービスで設定済み）
- AWS CLI v2 がインストールされていること
- 対象タスクのクラスター名・タスクID・コンテナ名が分かること
- IAM権限（ssmmessages, ecs:ExecuteCommand など）があること

### ステップ1: ECS Exec の有効化（初回のみ）
- タスク定義で enableExecuteCommand を true にする
- サービスやタスクを再デプロイ

### ステップ2: ECS Exec でコンテナに入る
```
aws ecs execute-command \
  --cluster larabel-cluster \
  --task 4f4444e488c941e8b0771465de74bfee \
  --container laravel-nagoyameshi \
  --interactive \
  --command "/bin/sh"
```

### ステップ3: コンテナ内での調査
入れたら、以下のコマンドでディレクトリ構成やファイルの有無を確認します。
```
# 現状
pwd
# 結果
/var/www/html/laravel-nagoyameshi

# ルート直下の確認
ls -l /var/www/html

# publicディレクトリの中身
ls -l /var/www/html/public
ls -l /var/www/html/laravel-nagoyameshi/public

# Apacheのログ確認
cat /var/log/apache2/error.log
cat /var/log/apache2/access.log

# index.php の有無
find /var/www/html -name index.php
```

### ステップ5: 調査結果を元に原因を特定
publicディレクトリに index.php が無い場合 → DockerfileのCOPYやDocumentRoot設定を見直す
ログにエラーが出ている場合 → エラー内容を確認

参考: ECS Exec のドキュメント
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/ecs-exec.html


### 環境変数の確認
```
printenv | grep APP_
printenv | grep DB_
```