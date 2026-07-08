# GitHub Actions VS. ChatGPT
## #信用するなChatGPT #それはそれとして助かる

命令：GitHubActions を使って、自動化をしたい。 developブランチに他ブランチからマージを行ったときをトリガーにして、 python はpytest実行、Next.jsはNext.jsに対応するテスト実行、pass後、 event-appコンテナとevent-app-webコンテナをECRにプッシュしたい。プッシュするレジストリは 両方event-appレジストリ。タグは、event-appの方はevent-up:backend-<gitのcommitログの番号>、event-app-webの方はevent-up:frontend-<gitのcommitログの番号>です。更にその後、EKSにデプロイしたいです。その流れ実現するworkflow.yamlを作成してください。

解答：はい。この構成であれば、一般的なCI/CDパイプラインになります。

```text
feature/*
    │
    ▼
Pull Request
    │
    ▼
developへマージ
    │
    ▼
GitHub Actions
    │
    ├── Checkout
    ├── Python Test (pytest)
    ├── Next.js Test
    ├── ECR Login
    ├── Backend Image Build
    ├── Frontend Image Build
    ├── Backend Push
    ├── Frontend Push
    ├── kubectl設定(EKS)
    ├── DeploymentのImage更新
    └── Rollout確認
```

この構成を前提として、以下の `workflow.yaml` を作成しました。

```yaml
name: CI/CD to Amazon EKS

on:
  pull_request:
    branches:
      - develop
    types:
      - closed

permissions:
  id-token: write
  contents: read

env:
  AWS_REGION: ap-northeast-1
  ECR_REPOSITORY: event-app
  CLUSTER_NAME: event-up-cluster
  K8S_NAMESPACE: default

jobs:
  ci-cd:
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest

    steps:

      ############################################################
      # Checkout
      ############################################################
      - name: Checkout
        uses: actions/checkout@v4

      ############################################################
      # Commit SHA
      ############################################################
      - name: Get Commit SHA
        run: |
          echo "SHORT_SHA=$(git rev-parse --short HEAD)" >> $GITHUB_ENV

      ############################################################
      # AWS Login (OIDC)
      ############################################################
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_GITHUB_ROLE }}
          aws-region: ${{ env.AWS_REGION }}

      ############################################################
      # Login ECR
      ############################################################
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      ############################################################
      # Python Test
      ############################################################
      - name: Setup Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.13"

      - name: Install Backend Dependencies
        run: |
          cd backend
          pip install -r requirements.txt

      - name: Run pytest
        run: |
          cd backend
          pytest

      ############################################################
      # Next.js Test
      ############################################################
      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: "22"

      - name: Install Frontend Dependencies
        run: |
          cd frontend
          npm ci

      - name: Run Next.js Test
        run: |
          cd frontend
          npm test

      ############################################################
      # Backend Build
      ############################################################
      - name: Build Backend Image
        run: |
          docker build \
            -t event-app:backend-${SHORT_SHA} \
            ./backend

      ############################################################
      # Frontend Build
      ############################################################
      - name: Build Frontend Image
        run: |
          docker build \
            -t event-app:frontend-${SHORT_SHA} \
            ./frontend

      ############################################################
      # Tag Backend
      ############################################################
      - name: Tag Backend
        run: |
          docker tag \
          event-app:backend-${SHORT_SHA} \
          ${{ steps.login-ecr.outputs.registry }}/${ECR_REPOSITORY}:backend-${SHORT_SHA}

      ############################################################
      # Tag Frontend
      ############################################################
      - name: Tag Frontend
        run: |
          docker tag \
          event-app:frontend-${SHORT_SHA} \
          ${{ steps.login-ecr.outputs.registry }}/${ECR_REPOSITORY}:frontend-${SHORT_SHA}

      ############################################################
      # Push Backend
      ############################################################
      - name: Push Backend
        run: |
          docker push \
          ${{ steps.login-ecr.outputs.registry }}/${ECR_REPOSITORY}:backend-${SHORT_SHA}

      ############################################################
      # Push Frontend
      ############################################################
      - name: Push Frontend
        run: |
          docker push \
          ${{ steps.login-ecr.outputs.registry }}/${ECR_REPOSITORY}:frontend-${SHORT_SHA}

      ############################################################
      # kubectl
      ############################################################
      - name: Install kubectl
        uses: azure/setup-kubectl@v4

      ############################################################
      # Configure kubeconfig
      ############################################################
      - name: Configure kubeconfig
        run: |
          aws eks update-kubeconfig \
            --region ${AWS_REGION} \
            --name ${CLUSTER_NAME}

      ############################################################
      # Update Backend Deployment
      ############################################################
      - name: Update Backend Image
        run: |
          kubectl set image deployment/event-app \
          event-app=${{ steps.login-ecr.outputs.registry }}/${ECR_REPOSITORY}:backend-${SHORT_SHA} \
          -n ${K8S_NAMESPACE}

      ############################################################
      # Update Frontend Deployment
      ############################################################
      - name: Update Frontend Image
        run: |
          kubectl set image deployment/event-app-web \
          event-app-web=${{ steps.login-ecr.outputs.registry }}/${ECR_REPOSITORY}:frontend-${SHORT_SHA} \
          -n ${K8S_NAMESPACE}

      ############################################################
      # Rollout Check
      ############################################################
      - name: Wait Backend Rollout
        run: |
          kubectl rollout status deployment/event-app \
          -n ${K8S_NAMESPACE}

      - name: Wait Frontend Rollout
        run: |
          kubectl rollout status deployment/event-app-web \
          -n ${K8S_NAMESPACE}
```

## ただし、いくつか実環境に合わせて調整が必要です

このワークフローは流れとしては正しいですが、以下は実際の環境に合わせて変更する必要があります。

| 項目                 | あなたの値に変更するもの                |
| ------------------ | --------------------------- |
| `backend/`         | Pythonアプリのディレクトリ名           |
| `frontend/`        | Next.jsアプリのディレクトリ名          |
| `event-up-cluster` | EKSクラスター名                   |
| `event-app`        | BackendのDeployment名         |
| `event-app-web`    | FrontendのDeployment名        |
| `event-app`        | ECRリポジトリ名（今回は共通）            |
| `AWS_GITHUB_ROLE`  | GitHub Actionsから引き受けるIAMロール |

### さらに改善できる点

実運用では、この1つのジョブにすべてを詰め込むよりも、

1. **test**（pytest・Next.jsテスト）
2. **build-and-push**（Dockerビルド・ECRプッシュ）
3. **deploy**（EKSデプロイ）

の3ジョブに分け、`needs` を使って依存関係を設定する構成がおすすめです。失敗箇所が分かりやすくなり、将来的にステージング・本番へのデプロイや承認フローも追加しやすくなります。

# 質問その2
質問：これって、pytestなどで失敗したときに、pull request,mergeってキャンセルされますか？

解答：結論から言うと、**今のworkflowではキャンセルされません。**

理由は、このWorkflowのトリガーが

```yaml
on:
  pull_request:
    branches:
      - develop
    types:
      - closed
```

だからです。

`types: closed` は**「Pull Requestが閉じられた後」**に実行されます。そして

```yaml
if: github.event.pull_request.merged == true
```

によって、「マージが完了した後」にのみWorkflowが開始されます。

つまり時系列は

```text
feature
    │
    ▼
Pull Request作成
    │
    ▼
レビュー
    │
    ▼
Merge ← ここでdevelopへ反映される
    │
    ▼
GitHub Actions開始
    │
    ├── pytest失敗
    └── npm test失敗
```

となるため、**テストが失敗してもdevelopには既にマージ済み**です。

---

## 一般的な運用

通常は次のように**2つのWorkflow**に分けます。

```text
① Pull Request作成・更新
        │
        ▼
pytest
Next.js Test
        │
        ├── PASS
        │      ↓
        │   Merge可能
        │
        └── FAIL
               ↓
        Merge不可
```

↓

```text
② developへMerge
        │
        ▼
Docker Build
        │
        ▼
ECR Push
        │
        ▼
EKS Deploy
```

これがGitHubでも最もよく採用されるCI/CDの構成です。

---

## Workflow①（CI）

トリガーは

```yaml
on:
  pull_request:
    branches:
      - develop
```

ここでは

* pytest
* Next.js test

だけ実行します。

失敗すると

```text
❌ Checks failed
```

になります。

さらにGitHubの**Branch Protection Rule**で

> **Require status checks to pass before merging**

を有効にすると、

```text
pytest ❌
```

の状態では**Mergeボタンが押せなくなります**。

---

## Workflow②（CD）

こちらは

```yaml
on:
  pull_request:
    branches:
      - develop
    types:
      - closed
```

または

```yaml
on:
  push:
    branches:
      - develop
```

をトリガーにして、

* Docker Build
* ECR Push
* EKS Deploy

だけを担当させます。

---

## 私のおすすめ構成

あなたのプロジェクト（Python + Next.js + ECR + EKS）なら、次のように分けるのがおすすめです。

```text
feature
    │
    ▼
Pull Request
    │
    ▼
GitHub Actions (CI)
    ├── pytest
    └── npm test
    │
    ├── PASS
    │      ↓
    │   Merge可能
    │
    └── FAIL
           ↓
      Merge不可
              │
              ▼
developへMerge
              │
              ▼
GitHub Actions (CD)
    ├── Docker Build
    ├── ECR Push
    └── EKS Deploy
```

この構成には次の利点があります。

* **品質保証**：テストに通らないコードは`develop`に入らない。
* **責務の分離**：CI（テスト）とCD（デプロイ）が独立し、トラブル時の切り分けが容易。
* **運用性**：将来的にステージング環境や本番環境へのデプロイフローを追加しやすい。

この構成が、実際の業務でも広く採用されているベストプラクティスです。
