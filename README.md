### SNS 設計図

### サービス概要
- 投稿には最大閲覧数を設定でき、上限を超えると投稿はマスキングされる
- 投稿した痕跡（日時・閲覧数など）はプロフィールやタイムラインに残る
- ユーザー登録・ログインはメールアドレス + ワンタイムパスワード（OTP）方式
- 本サービスはログイン状態の保持やパスワードの保存を行わない

### 認証仕様（セキュリティポリシー）
- メールアドレスのみで認証
- 入力されたメールアドレスに**ワンタイムパスワード（OTP）**を送信
- ユーザーはそのOTPを使ってログイン（OTP有効時間：5分程度）
- セッションやクッキーでログイン状態は維持せず、基本は「その都度認証」
- セキュリティの責任を極力サービス側に持たせない設計方針

### データベース構成（ER図付き）
#### テーブル：User
| フィールド名  | 型       | 説明                 |
| ------------ | -------- | -------------------- |
| id           | String   | ユーザーID（メールアドレス） |
| created_at   | Date     | 登録日時             |

#### テーブル：Post
| フィールド名     | 型        | 説明                     |
| --------------- | --------- | ------------------------ |
| id              | UUID      | 投稿ID（一意）           |
| author_id       | String    | 投稿者のID（User.id）    |
| content         | Text      | 投稿内容                 |
| image_id        | String?   | 添付画像ID（オプション） |
| max_views       | Integer   | 最大閲覧数               |
| current_views   | Integer   | 現在の閲覧数             |
| created_at      | DateTime  | 投稿日時                 |
| expired         | Boolean   | 閲覧数上限に達したかどうか（計算式で可） |

#### テーブル：OTP (One Time Password)
| フィールド名  | 型        | 説明                     |
| ------------ | --------- | ------------------------ |
| email        | String    | 対象メールアドレス       |
| otp_code     | String    | 発行されたOTP（6桁など） |
| expires_at   | DateTime  | 有効期限（例：5分）      |
| used         | Boolean   | 使用済みかどうか         |

### 投稿マスキング仕様
- 投稿の閲覧数が max_views に達すると：
  - 投稿本文と画像は非表示
  - 投稿者名、投稿日、現在の閲覧数などの「メタ情報」は表示
  - 投稿者本人のタイムラインには履歴として残る

### 今後の拡張候補
- 画像アップロード機能（Flask staticまたはS3）
- 投稿削除 / 通報機能
- WebSocketでのリアルタイム更新
- QRコード投稿 / 限定公開

# 全体構成図

## アプリ機能構成
User（メールアドレス）ログイン
   ↓
Frontend (React)
   ↓
Backend (Flask REST API)
   ├── 投稿作成
   ├── 投稿取得・閲覧数更新
   ├── OTP発行・認証
   └── 画像アップロード（予定）
   ↓
PostgreSQL / SQLite（最小構成はSQLite）

## ☁️ Kubernetes & CI/CD 構成図
[ GitHub ]
   └─ Push
       ↓
[ GitHub Actions ]
   └─ Build & Push Docker image
       ↓
[ Docker Hub（or private registry）]

[ Argo CD ]
   └─ Watch Git Repo for Manifests
       ↓
[ Kubernetes Cluster (on mini PC) ]
   ├── Pod: frontend (React + Nginx)
   ├── Pod: backend (Flask + Gunicorn)
   ├── Pod: postgres or PV-mounted SQLite
   └── Service / Ingress / PVC

## インフラ構成（mini PC 上）
層	構成
ホストOS	Ubuntu Server / Proxmox VE
仮想基盤	KVM（libvirt）or Proxmox
仮想マシン	k8s-master（2vCPU / 2GB）
k8s-node1（2vCPU / 4GB）
K8s構築方法	Kubespray（最小構成）
GitOps	Argo CD
CI/CD	GitHub Actions + Docker Hub

## kubernetes Namespace & Pod 設計案
Namespace	Pod名	機能
sns-app	frontend	React SPA + Nginx
sns-app	backend	Flask REST API + Gunicorn
sns-app	postgres or sqlite	DBサーバー
argocd	ArgoCD一式	GitOps管理ツール

# ディレクトリ構成（Git）

sns-project/
├── backend/
│   ├── app.py
│   ├── Dockerfile
│   └── ...
├── frontend/
│   ├── src/
│   ├── Dockerfile
│   └── ...
├── k8s-manifests/
│   ├── frontend.yaml
│   ├── backend.yaml
│   ├── ingress.yaml
│   └── ...
├── README.md
└── .github/
    └── workflows/
        └── deploy.yml（CI/CD設定
        
## 検討課題
 備考
開発初期は SQLite + Local PV でOK

投稿画像は static/images に保存してもよい（本番はS3推奨）

OTPの送信には SendGrid / Mailgun / SMTP など使用可能

Argo CD の GUI は NodePort or Ingress でアクセス可能
