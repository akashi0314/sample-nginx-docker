# React+TypeScript フロントエンド本番用 Docker Compose + Nginx

本番環境で使用するDocker Compose + Nginx構成のサンプルです。

## プロジェクト構造

```
frontend-docker-project/
├── docker-compose.yml
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   ├── tsconfig.json
│   ├── vite.config.ts
│   ├── src/
│   │   ├── App.tsx
│   │   ├── main.tsx
│   │   ├── index.css
│   │   └── components/
│   │       └── Welcome.tsx
│   ├── public/
│   │   ├── index.html
│   │   └── favicon.ico
│   └── nginx/
│       ├── nginx.conf
│       └── default.conf
└── README.md
```

## 1. Viteデフォルト構成を使用

このサンプルでは、Viteで生成されるデフォルトのReact+TypeScriptアプリをそのまま使用します。

### プロジェクト作成コマンド

```bash
# frontendディレクトリでViteプロジェクト作成
cd frontend
npx create-vite@latest . --template react-ts

# 依存関係をインストール
npm install

# 開発サーバーテスト（オプション）
npm run dev

# ビルドテスト
npm run build
```

### デフォルトファイル構成

作成されるデフォルトファイル：
- `src/main.tsx` - エントリーポイント
- `src/App.tsx` - メインコンポーネント
- `src/App.css` - アプリスタイル
- `src/index.css` - グローバルスタイル
- `src/assets/react.svg` - Reactロゴ
- `public/vite.svg` - Viteロゴ

### package.json（自動生成）
デフォルトで以下の内容が生成されます：

```json
{
  "name": "frontend",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "lint": "eslint .",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^18.3.1",
    "react-dom": "^18.3.1"
  },
  "devDependencies": {
    "@eslint/js": "^9.9.0",
    "@types/react": "^18.3.3",
    "@types/react-dom": "^18.3.0",
    "@vitejs/plugin-react": "^4.3.1",
    "eslint": "^9.9.0",
    "eslint-plugin-react-hooks": "^5.1.0-rc.0",
    "eslint-plugin-react-refresh": "^0.4.9",
    "globals": "^15.9.0",
    "typescript": "^5.5.3",
    "vite": "^5.4.1"
  }
}
```

### vite.config.ts（自動生成）
```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

// https://vitejs.dev/config/
export default defineConfig({
  plugins: [react()],
})
```

## 2. Docker設定

### frontend/nginx/nginx.conf
```nginx
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    # パフォーマンス最適化
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    client_max_body_size 16M;

    # gzip圧縮
    gzip on;
    gzip_vary on;
    gzip_comp_level 6;
    gzip_min_length 1024;
    gzip_proxied any;
    gzip_types
        application/atom+xml
        application/javascript
        application/json
        application/rss+xml
        application/vnd.ms-fontobject
        application/x-font-ttf
        application/x-web-app-manifest+json
        application/xhtml+xml
        application/xml
        font/opentype
        image/svg+xml
        image/x-icon
        text/css
        text/javascript
        text/plain
        text/xml;

    include /etc/nginx/conf.d/*.conf;
}
```

### frontend/nginx/default.conf
```nginx
server {
    listen 80;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # セキュリティヘッダー
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header X-Download-Options "noopen" always;
    add_header X-Permitted-Cross-Domain-Policies "none" always;

    # React Router対応 - SPAのルーティング
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 静的ファイルのキャッシュ設定
    location ~* \.(js|css|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        add_header Access-Control-Allow-Origin "*";
    }

    # 画像ファイルのキャッシュ
    location ~* \.(png|jpg|jpeg|gif|ico|svg|webp)$ {
        expires 1M;
        add_header Cache-Control "public";
        add_header Access-Control-Allow-Origin "*";
    }

    # HTMLファイルはキャッシュしない
    location ~* \.html$ {
        expires -1;
        add_header Cache-Control "no-cache, no-store, must-revalidate";
        add_header Pragma "no-cache";
    }

    # ヘルスチェック用エンドポイント
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }

    # 不要なファイルへのアクセスをブロック
    location ~ /\.(ht|git|env) {
        deny all;
        return 404;
    }

    # favicon.icoのログを無効化
    location = /favicon.ico {
        log_not_found off;
        access_log off;
    }

    # robots.txtのログを無効化
    location = /robots.txt {
        log_not_found off;
        access_log off;
    }
}
```

## 4. Nginx設定

### frontend/Dockerfile
```dockerfile
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
```

## 4. Docker Compose設定

### docker-compose.yml
```yaml
services:
  # Node.js build stage
  frontend-builder:
    image: node:22-alpine
    container_name: frontend-builder
    working_dir: /app
    volumes:
      - ./frontend:/app
      - /app/node_modules  # 匿名ボリュームでnode_modulesをコンテナ内に保持
    command: >
      sh -c "
        npm install &&
        npm run build
      "
    networks:
      - app-network

  # Nginx production stage
  frontend:
    image: nginx:alpine
    container_name: frontend-app
    ports:
      - "80:80"
    volumes:
      - ./frontend/dist:/usr/share/nginx/html:ro
      - ./frontend/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      frontend-builder:
        condition: service_completed_successfully
    restart: unless-stopped
    environment:
      - NODE_ENV=production
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  app-network:
    driver: bridge
```

## 5. 実行方法

### アプリケーションの起動
```bash
# プロジェクトをクローン/作成
mkdir frontend-docker-project
cd frontend-docker-project

# 上記のファイル構造を作成後...

# ビルドして起動
docker-compose up -d --build

# ログ確認
docker-compose logs -f frontend

# ブラウザで http://localhost にアクセス
```

### 管理コマンド
```bash
# 状態確認
docker-compose ps

# ヘルスチェック確認
docker-compose exec frontend curl http://localhost/health

# Nginxの設定テスト
docker-compose exec frontend nginx -t

# 設定リロード
docker-compose exec frontend nginx -s reload

# 停止
docker-compose down

# 完全削除（イメージも含む）
docker-compose down --rmi all -v
```

### デプロイ用コマンド
```bash
# イメージをビルド
docker-compose build --no-cache

# バックグラウンドで起動
docker-compose up -d

# ログ監視
docker-compose logs -f --tail=100 frontend
```

## 学習ポイント

1. **マルチステージビルド**: 最適化された軽量な本番イメージ
2. **Nginx最適化**: パフォーマンスとセキュリティの設定
3. **SPA対応**: React Routerに対応したルーティング設定
4. **キャッシュ戦略**: 静的ファイルの適切なキャッシュ設定
5. **ヘルスチェック**: コンテナの正常性監視
6. **セキュリティ**: 適切なヘッダーと権限設定

この構成で本番環境にデプロイ可能なReact+TypeScriptアプリケーションが完成します！