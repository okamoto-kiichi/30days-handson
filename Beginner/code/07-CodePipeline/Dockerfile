# ✅ 成功パターン：最新の安全なベースイメージを使用
# このDockerfileは脆弱性スキャンに合格することを想定しています

FROM --platform=linux/amd64 nginx:1.27-alpine

# メタデータ
LABEL maintainer="AWS Jr Champions"
LABEL description="ECR脆弱性スキャンとCodePipeline - 安全なイメージパターン"
LABEL version="1.0"
LABEL scan-status="safe"

# カスタムHTMLページを追加
RUN printf '<!DOCTYPE html>\n<html lang="ja">\n<head>\n    <meta charset="UTF-8">\n    <meta name="viewport" content="width=device-width, initial-scale=1.0">\n    <title>セキュアCI/CDデモ</title>\n    <style>\n        body { font-family: Arial, sans-serif; max-width: 800px; margin: 50px auto; padding: 20px; }\n        h1 { color: #232f3e; }\n        .status { padding: 15px; background: #d4edda; border-left: 4px solid #28a745; margin: 20px 0; }\n        .info { color: #666; }\n    </style>\n</head>\n<body>\n    <h1>🚀 セキュアCI/CDデモへようこそ！</h1>\n    <div class="status">\n        <strong>✅ ステータス：</strong>このイメージは脆弱性スキャンに合格しています\n    </div>\n    <p>このアプリケーションは、CodePipelineによって自動的にビルド、スキャン、デプロイされています。</p>\n    <div class="info">\n        <h3>使用技術：</h3>\n        <ul>\n            <li>Amazon ECR - コンテナレジストリ</li>\n            <li>Amazon Inspector - 脆弱性スキャン</li>\n            <li>AWS CodePipeline - CI/CDパイプライン</li>\n            <li>AWS CodeBuild - ビルドとテスト</li>\n        </ul>\n    </div>\n</body>\n</html>' > /usr/share/nginx/html/index.html

# ヘルスチェック用エンドポイント
RUN echo 'ok' > /usr/share/nginx/html/health

# 80番ポートを公開
EXPOSE 80

# Nginxをフォアグラウンドで実行
CMD ["nginx", "-g", "daemon off;"]

