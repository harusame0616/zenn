---
title: "GitHub Actions で Cloud Run へデプロイする"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ['GCP', 'CloudRun', 'GitHubActions']
published: true
---

# GitHub Actions で Cloud Run へデプロイする

## 概要

GitHub Actions で Cloud Run に自動デプロイする設定を行ってみました。

備忘録として参考にしたサイトと、手順を残しておきます。

[https://github.com/harusame0616/cloud-run-github-actions](https://github.com/harusame0616/cloud-run-github-actions)

node.js + express で {message: 'hello world'} を返すだけのAPIサーバーをコンテナ化し、
mainにマージされるのを契機に GitHub Actions で Dockerイメージのビルドとデプロイ、Cloud Run へのデプロイを行います。

認証にJSONサービスアカウントキーを使用するのは非推奨となっているので、Workload Identityを使って認証を行っています。

--- 

## GCP設定

ほぼ以下のサイトの手順通りで、Cloud Run用の設定を追加しています。

[https://www.asobou.co.jp/blog/web/workload-identity](https://www.asobou.co.jp/blog/web/workload-identity)

### APIを有効にする

- IAM Service Account Credentials API
- ****Cloud Run Admin API****

```bash
> gcloud services enable \
		iamcredentials.googleapis.com \
		run.googleapis.com
```

### サービスアカウントを作成する

```bash
> gcloud iam service-accounts create ${SERVICE_ACCOUNT}
```

### Workload Identity Pool を作成する

```bash
gcloud iam workload-identity-pools create "${POOL_NAME}" \
  --location="global" 
```

### Workload Identity Provider を作成する

```bash
gcloud iam workload-identity-pools providers create-oidc "${PROVIDER_NAME}" \
  --location="global" \
  --workload-identity-pool="${POOL_NAME}" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.actor=assertion.actor,attribute.aud=assertion.aud" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

### サービス アカウントの権限借用の設定

```bash
POOL_ID=$(gcloud iam workload-identity-pools describe "${POOL_NAME}" \
    --location="global" \
    --format="value(name)")
gcloud iam service-accounts add-iam-policy-binding "${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/${POOL_ID}/attribute.repository/${GITHUB_REPO}" 
```

### サービスアカウントのロール設定

- Cloud Run 管理者
- Storage 管理者
- サービス アカウント ユーザー

```bash
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member="serviceAccount:${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
--role="roles/run.admin"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member="serviceAccount:${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
--role="roles/storage.admin"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
--member="serviceAccount:${SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com" \
--role="roles/iam.serviceAccountUser"
```

--- 

## GitHubの設定

GitHub-ActionsのYMLは以下をベースに作成しました。

[https://github.com/google-github-actions/setup-gcloud/tree/main/example-workflows/cloud-run](https://github.com/google-github-actions/setup-gcloud/tree/main/example-workflows/cloud-run)

```yaml
on:
  push:
    branches:
      - main

name: Build and Deploy to Cloud Run
env:
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}
  SERVICE_NAME: hello-world

jobs:
  deploy:
    runs-on: ubuntu-latest

    permissions:
      contents: 'read'
      id-token: 'write'

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - id: 'auth'
        name: 'Authenticate to Google Cloud'
        uses: 'google-github-actions/auth@v0.4.0'
        with:
          workload_identity_provider: ${{ secrets.WORKLOAD_IDENTITY_PROVIDER }}
          service_account: github-actions@${{ secrets.GCP_PROJECT_ID }}.iam.gserviceaccount.com

      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v0

      - name: Authorize Docker push
        run: gcloud auth configure-docker

      - name: Build Docker image
        run: docker build -t asia.gcr.io/$PROJECT_ID/$SERVICE_NAME:${{ github.sha }} .

      - name: Push Docker Image
        run: docker push asia.gcr.io/$PROJECT_ID/$SERVICE_NAME:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |-
          gcloud run deploy $SERVICE_NAME \
            --project=$PROJECT_ID \
            --image=asia.gcr.io/$PROJECT_ID/$SERVICE_NAME:${{ github.sha }} \
            --region=$REGION \
            --service-account=github-actions@$PROJECT_ID.iam.gserviceaccount.com \
            --allow-unauthenticated
```

GitHub Actionsのシークレット設定

- GCP_PROJECT_ID
    - GCPのプロジェクトID
- WORKLOAD_IDENTITY_PROVIDER
    
    ```bash
    echo $(gcloud iam workload-identity-pools providers describe ${PROVIDER_NAME} --location=global --workload-identity-pool=${POOL_NAME} --format="value(name)")
    ```


## 他参考

- https://cloud.google.com/blog/ja/products/identity-security/enabling-keyless-authentication-from-github-actions
- https://cloud.google.com/iam/docs/workload-identity-federation#impersonation
- https://zenn.dev/konnyaku256/articles/auto-deploy-with-cloudrun-and-githubactions
- https://zenn.dev/rince/scraps/4e3cbba78d2cd1
- https://christina04.hatenablog.com/entry/workload-identity-federation
- https://dev.classmethod.jp/articles/google-cloud-auth-with-workload-identity/