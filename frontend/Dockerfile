# Build stage
FROM node:22-alpine AS builder

WORKDIR /app

# パッケージファイルをコピーして依存関係をインストール
COPY package*.json ./
RUN npm ci --only=production --silent

# ソースコードをコピーしてビルド
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine

# セキュリティ: 不要なパッケージを削除
RUN apk --no-cache add curl

# Nginxの設定をコピー
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/default.conf /etc/nginx/conf.d/default.conf

# ビルドされたファイルをコピー
COPY --from=builder /app/dist /usr/share/nginx/html

# Nginxが必要なディレクトリを作成し、適切な権限を設定
RUN mkdir -p /var/cache/nginx/client_temp \
             /var/cache/nginx/proxy_temp \
             /var/cache/nginx/fastcgi_temp \
             /var/cache/nginx/uwsgi_temp \
             /var/cache/nginx/scgi_temp && \
    chown -R nginx:nginx /var/cache/nginx && \
    chown -R nginx:nginx /usr/share/nginx/html && \
    chown -R nginx:nginx /var/log/nginx && \
    chmod -R 755 /usr/share/nginx/html

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

EXPOSE 80

# rootユーザーで実行（権限問題を回避）
CMD ["nginx", "-g", "daemon off;"]