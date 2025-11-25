# ECR脆弱性スキャンとCodePipeline デモ資材

このディレクトリには、Day07の課題「ECR脆弱性スキャンとCodePipelineで安全な自動デプロイを構築しよう」で使用する資材が含まれています。

## 📁 ファイル構成

- **buildspec.yml**: CodeBuildで使用するビルド仕様ファイル
- **Dockerfile**: ✅ 安全なイメージパターン（脆弱性スキャン合格想定）
- **Dockerfile.vulnerable**: ❌ 脆弱性を含むイメージパターン（スキャン失敗想定）
- **.dockerignore**: Dockerビルド時に除外するファイルの設定
- **demo-comparison.md**: 成功・失敗パターンの詳細比較表
- **VERIFICATION-REPORT.md**: AWS公式ドキュメントに基づく検証レポート
- **README.md**: この説明ファイル

## 🚀 使い方

### 1. ECRリポジトリの作成

```bash
aws ecr create-repository \
    --repository-name demo-app \
    --image-scanning-configuration scanOnPush=true \
    --region ap-northeast-1
```

### 2. パターン選択

このディレクトリには2種類のDockerfileがあります：

#### ✅ パターンA：安全なイメージ（スキャン合格）

最新の安全なベースイメージ（nginx:1.27-alpine）を使用します。

```bash
# 通常のDockerfileを使用
docker build --platform linux/amd64 -t demo-app:safe -f Dockerfile .
```

#### ❌ パターンB：脆弱性のあるイメージ（スキャン失敗）

古いベースイメージ（nginx:1.19-alpine）を使用し、意図的に脆弱性を含めます。

```bash
# 脆弱性のあるDockerfileを使用
docker build --platform linux/amd64 -t demo-app:vulnerable -f Dockerfile.vulnerable .
```

⚠️ **注意**: Dockerfile.vulnerableは学習・デモ目的でのみ使用してください。

### 3. ローカルでのテスト（任意）

```bash
# 安全なイメージのテスト
docker run -p 8080:80 demo-app:safe

# 脆弱性のあるイメージのテスト
docker run -p 8081:80 demo-app:vulnerable

# ブラウザで http://localhost:8080 または http://localhost:8081 にアクセス
```

### 4. ECRへの手動プッシュ（任意）

#### ✅ 安全なイメージをプッシュ（スキャン合格を確認）

```bash
# AWSアカウントIDを取得
AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# ECRにログイン
aws ecr get-login-password --region ap-northeast-1 | \
    docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com

# イメージにタグ付け
docker tag demo-app:safe ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/demo-app:safe

# ECRにプッシュ
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/demo-app:safe
```

#### ❌ 脆弱性のあるイメージをプッシュ（スキャン失敗を確認）

```bash
# イメージにタグ付け
docker tag demo-app:vulnerable ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/demo-app:vulnerable

# ECRにプッシュ
docker push ${AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/demo-app:vulnerable
```

### 5. スキャン結果の確認

#### ✅ 安全なイメージのスキャン結果

```bash
# スキャン完了まで少し待つ
sleep 30

# スキャン結果を取得（安全なイメージ）
aws ecr describe-image-scan-findings \
    --repository-name demo-app \
    --image-id imageTag=safe \
    --region ap-northeast-1

# 期待される結果: CRITICALやHIGHの脆弱性が0件、またはごく少数
```

#### ❌ 脆弱性のあるイメージのスキャン結果

```bash
# スキャン結果を取得（脆弱性のあるイメージ）
aws ecr describe-image-scan-findings \
    --repository-name demo-app \
    --image-id imageTag=vulnerable \
    --region ap-northeast-1

# 期待される結果: CRITICALまたはHIGHレベルの脆弱性が複数検出される
```

### 6. CodePipelineでの動作確認

buildspec.ymlは使用するDockerfileに応じて動作が変わります：

**✅ 成功パターン（Dockerfileを使用）:**
```bash
# buildspec.ymlはデフォルトでDockerfileを使用
# → ビルド成功 → スキャン合格 → デプロイ実行
```

**❌ 失敗パターン（Dockerfile.vulnerableを使用）:**

buildspec.ymlの32行目を以下のように変更：
```yaml
# 変更前
- docker build --platform linux/amd64 -t $IMAGE_REPO_NAME:$IMAGE_TAG .

# 変更後（脆弱性パターンをテストする場合）
- docker build --platform linux/amd64 -t $IMAGE_REPO_NAME:$IMAGE_TAG -f Dockerfile.vulnerable .
```

この場合の動作：
```bash
# → ビルド成功 → スキャン実行 → CRITICAL検出 → ビルド失敗 → デプロイ中止
```

## 🔧 buildspec.yml の主な機能

1. **ECRへのログイン**: 自動的にECRに認証
2. **Dockerイメージのビルド**: Dockerfileからイメージをビルド
3. **ECRへのプッシュ**: ビルドしたイメージをECRにプッシュ
4. **脆弱性スキャン結果のチェック**: 
   - CRITICALレベルの脆弱性を検出した場合、ビルドを失敗させる
   - 安全なイメージのみデプロイが進むようにする
5. **ECS用の成果物出力**: `imagedefinitions.json` を生成

## ⚙️ カスタマイズポイント

### リージョンの変更

```yaml
env:
  variables:
    AWS_DEFAULT_REGION: "ap-northeast-1"  # 任意のリージョンに変更
```

### リポジトリ名の変更

```yaml
env:
  variables:
    IMAGE_REPO_NAME: "demo-app"  # 任意の名前に変更
```

### HIGHレベルの脆弱性も検出する場合

`buildspec.yml`の以下のコメントアウトを解除：

```yaml
if [ "$HIGH_COUNT" -gt 0 ]; then
  echo "⚠️  HIGH レベルの脆弱性が検出されました！確認が必要です。"
  exit 1
fi
```

## 📊 CodePipelineとの統合

このbuildspec.ymlは、CodePipelineのBuildステージで使用されます：

1. **Source（ソース）**: GitHubまたはCodeCommitからコードを取得
2. **Build（ビルド）**: このbuildspec.ymlを使用してビルド・スキャン
3. **Deploy（デプロイ）**: スキャンに合格したイメージをECSにデプロイ

## 🧹 リソースの削除

```bash
# ECRリポジトリの削除（イメージも全て削除）
aws ecr delete-repository \
    --repository-name demo-app \
    --force \
    --region ap-northeast-1
```

## 🧪 テストシナリオ

### シナリオ1：安全なイメージのデプロイ（成功）
1. `Dockerfile`を使用してビルド
2. ECRにプッシュ → 自動スキャン実行
3. スキャン結果：脆弱性なし（またはCRITICAL = 0）
4. buildspec.ymlのチェック：合格
5. 結果：✅ パイプライン成功、ECSへデプロイ

### シナリオ2：脆弱性のあるイメージのデプロイ（失敗）
1. `Dockerfile.vulnerable`を使用してビルド
2. ECRにプッシュ → 自動スキャン実行
3. スキャン結果：CRITICALレベルの脆弱性を検出
4. buildspec.ymlのチェック：失敗（exit 1）
5. 結果：❌ パイプライン失敗、デプロイ中止

## 📝 注意事項

### スキャン待機時間について
- スキャン完了には通常30秒〜数分かかります
- buildspec.ymlでは簡易的に`sleep 30`を使用していますが、大きなイメージの場合は不足する可能性があります
- より確実にするには、スキャン状態をポーリングするループ処理を実装することをおすすめします

### セキュリティについて
- CRITICALレベルの脆弱性が検出されると、デプロイは自動的にブロックされます
- `Dockerfile.vulnerable`は**学習・デモ目的でのみ使用**してください。本番環境では使用しないでください
- CodeBuildの特権モードはDockerビルドに必要ですが、セキュリティリスクがあることに注意してください

### スキャンタイプについて
- このハンズオンでは **Basic scanning (AWS Native)** を使用しています
- より高度な機能が必要な場合は、**Enhanced scanning** (Amazon Inspector統合) も検討してください
- 詳細: [Scan images for software vulnerabilities in Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)

## 🔗 参考リンク

- [Amazon ECR イメージスキャン](https://docs.aws.amazon.com/AmazonECR/latest/userguide/image-scanning.html)
- [AWS CodeBuild buildspec.yml リファレンス](https://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html)

