# Config Controller ハンズオン

## 0. 前提条件
- Google Cloud 上にプロジェクトが作成してある
- プロジェクトのオーナー権限をもつアカウントで Cloud Shell にログインしている

## 1. Config Controller クラスタ作成
本手順は代表者 1 名のみ実行してください  

### 1-1. API 有効化
Cloud Shell 上で以下コマンドを実行し、本手順で利用する API (Config Controller, GKE, Resource Manager) を有効化します
```bash
gcloud services enable krmapihosting.googleapis.com \
    container.googleapis.com \
    cloudresourcemanager.googleapis.com
```

### 1-2. Config Controller クラスタの作成
任意の名前で Config Controller クラスタを作成します (完成まで 十数分程度時間がかかります)
```bash
export CONFIG_CONTROLLER_NAME=<Config Controller クラスタ名>

gcloud alpha anthos config controller create ${CONFIG_CONTROLLER_NAME} \
    --location=us-central1
```

コマンド実行完了後、以下コマンドを実行し想定通りクラスタが作成されていることを確認します
```bash
gcloud alpha anthos config controller list --location=us-central1
```

### 1-3. Config Controller に対して権限を付与
以下コマンドを実行し、Config Controller に対して必要な権限を付与します
```bash
export PROJECT_ID=$(gcloud config get-value project)
export SA_EMAIL="$(kubectl get ConfigConnectorContext -n config-control \
    -o jsonpath='{.items[0].spec.googleServiceAccount}' 2> /dev/null)"

gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
    --member "serviceAccount:${SA_EMAIL}" \
    --role "roles/owner" \
    --condition=None \
    --project "${PROJECT_ID}"
```

## 2. Config Controller によるリソースの作成
この手順以降は参加者各自で進めてください

### 2-1. 作業準備
ハンズオン用リポジトリをクローンします
```bash
# 自分の ID を入力
export USERNAME=<自分の ID>

# ハンズオン用リポジトリのクローン
git clone https://github.com/kkuchima/config-controller-handson.git

# 作業用ディレクトリの作成 & 移動
cd config-controller-handson
```

### 2-2. リソースの作成
Config Controller を利用し Cloud Storage バケットを作成します
```bash
# 東京 (asia-northeast1) にバケットを作成するマニフェストの作成
cat << EOF > tokyo-bucket.yaml
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  annotations:
    cnrm.cloud.google.com/force-destroy: "false"
  name: ${USERNAME}-tokyo
  namespace: config-control
spec:
  location: asia-northeast1
EOF

# マニフェストの Apply
kubectl apply -f tokyo-bucket.yaml
```

### 2-3. 作成した Cloud Storage バケットの確認
https://cloud.google.com/storage/browser?project=${PROJECT_ID} にアクセスし、手順 2-2 で作成したバケットが存在していることを確認します

### 2-4. 作成した Cloud Storage バケットの削除
作成したバケットを削除します  
2-3 と同様に Cloud Console 上からもバケットが削除されていることを確認します
```bash
kubectl delete -f tokyo-bucket.yaml
```

## 3. Policy Controller を使ったガードレールの作成（ロケーションの制約）

### 3-1. 制約テンプレートの適用
Cloud Storage バケットのロケーションを制限する制約テンプレートを適用します
```bash
kubectl apply -f constraint/gcpstoragelocationconstraintv1.yaml
```

### 3-2. 制約の作成
Cloud Storage バケットのロケーションを制限する制約を作成します
```bash
# 制約の中身を確認（asia-northeast1 / 2 のみバケットの作成を許可）
cat constraint/gcs-tokyo-osaka-only.yaml

# 制約の作成
kubectl apply -f constraint/gcs-tokyo-osaka-only.yaml
```

### 3-3. 制約の動作確認
動作確認のため、大阪とシンガポールにバケットを作成します
```bash
# 大阪 (asia-northeast2) にバケットを作成するマニフェストの作成
cat << EOF > osaka-bucket.yaml
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  annotations:
    cnrm.cloud.google.com/force-destroy: "false"
  name: ${USERNAME}-osaka
  namespace: config-control
spec:
  location: asia-northeast2
EOF

# シンガポール (asia-southeast1) にバケットを作成するマニフェストの作成
cat << EOF > singapore-bucket.yaml
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  annotations:
    cnrm.cloud.google.com/force-destroy: "false"
  name: ${USERNAME}-singapore
  namespace: config-control
spec:
  location: asia-southeast1
EOF

# 大阪にバケットを作成（成功）
kubectl apply -f osaka-bucket.yaml

# シンガポールにバケットを作成（失敗）
kubectl apply -f singapore-bucket.yaml
```

### 3-4. 作成した Cloud Storage バケットの確認
https://cloud.google.com/storage/browser?project=${PROJECT_ID} にアクセスし、手順 3-3 で作成したバケットが存在していることを確認します

## 4. Policy Controller を使ったガードレールの作成（バージョニングの要求）

### 4-1. 制約テンプレートの適用
Cloud Storage バケットのバージョニング設定を要求する制約テンプレートを適用します
```bash
# 制約テンプレートの中身を確認
cat constraint/gcpstorageversioningconstraintv1.yaml

# 制約テンプレートの適用
kubectl apply -f constraint/gcpstorageversioningconstraintv1.yaml
```

### 4-2. 制約の作成
Cloud Storage バケットのバージョニングを要求する制約を作成します
```bash
# 制約の中身を確認（パラメータの指定なし）
cat constraint/gcs-versioning-required.yaml

# 制約の作成
kubectl apply -f constraint/gcs-versioning-required.yaml
```

### 4-3. バケットの作成（バージョニングなし）
東京リージョンに Cloud Storage バケットを作成します（バージョニング定義なしなので失敗）
```bash
kubectl apply -f tokyo-bucket.yaml
```

### 4-4. バージョニングを有効化したバケットの作成
バージョニングを有効化したバケットを東京リージョンに作成します
```bash
cat << EOF > tokyo-bucket.yaml
apiVersion: storage.cnrm.cloud.google.com/v1beta1
kind: StorageBucket
metadata:
  annotations:
    cnrm.cloud.google.com/force-destroy: "false"
  name: ${USERNAME}-tokyo
  namespace: config-control
spec:
  location: asia-northeast1
  versioning:
    enabled: true
EOF

# マニフェストの Apply
kubectl apply -f tokyo-bucket.yaml
```

### 4-5. 作成した Cloud Storage バケットの確認
https://cloud.google.com/storage/browser?project=${PROJECT_ID} にアクセスし、手順 4-4 で作成したバケットが存在していることを確認します

## 5. ハンズオン環境のクリーンアップ
ハンズオンが完了したら本環境で利用したプロジェクトを削除します