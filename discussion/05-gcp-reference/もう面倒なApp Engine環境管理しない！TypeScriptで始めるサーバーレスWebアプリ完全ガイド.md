## もう面倒なApp Engine環境管理しない！TypeScriptで始めるサーバーレスWebアプリ完全ガイド

## はじめに

「サーバー管理は面倒、でも高性能なWebアプリケーションを簡単にデプロイしたい」

そんな悩みを抱える開発者の方に朗報です。Google Cloud Platform の **App Engine** なら、サーバー管理不要でスケーラブルなWebアプリケーションを構築できます。

この記事では、**TypeScript** を使ってApp Engineアプリケーションを開発し、実際にデプロイするまでの手順を実践的に解説します。記事を読み終える頃には、あなたも本格的なサーバーレスアプリケーションを運用できるようになっているはずです 💪

>* App Engine の基本概念とメリット・デメリットの理解
>* TypeScript + Express でのApp Engineアプリ開発手法
>* 環境分離とデプロイ戦略のベストプラクティス
>* パフォーマンス最適化とコスト管理のコツ

## App Engineとは？サーバーレスの真価

Google App Engine は、Googleが提供する**フルマネージドなサーバーレスプラットフォーム**です。

**1. ゼロから始められる手軽さ**

>* サーバー設定やOS管理が一切不要
>* `gcloud app deploy` コマンド一つでデプロイ完了
>* 自動スケーリングでトラフィック増にも即座に対応

**2. 開発者ファーストな環境**

>* Node.js、Python、Java、Go など主要言語をサポート
>* TypeScript開発も標準的にサポート
>* 豊富なGCPサービスとの連携（Datastore、Cloud SQL、Pub/Subなど）

**3. エンタープライズレベルの運用品質**

>* 99.95%のSLA保証
>* 世界中のGoogleデータセンターでのマルチリージョン展開
>* セキュリティパッチやランタイム更新はGoogle側で自動実行

**⚖️ 他のサーバーレス選択肢との比較**

| サービス | デプロイの手軽さ | 言語サポート | 料金体系 | 適用場面 |
|---------|---------------|-------------|----------|---------|
| **App Engine** | ⭐⭐⭐ | Node.js, Python, Java, Go | リクエスト数 + 実行時間 | Webアプリ全般 |
| **Cloud Run** | ⭐⭐ | コンテナベース（何でも） | CPU使用時間ベース | マイクロサービス |
| **Cloud Functions** | ⭐⭐⭐ | Node.js, Python, Go | 実行回数ベース | 軽量な処理・API |

**📌 App Engineを選ぶべきシーン**

>* 従来のWebアプリケーション開発パターンを踏襲したい
>* データベース連携が多いアプリケーション
>* チーム開発でのデプロイ作業を標準化したい

## 開発環境のセットアップ

App Engine開発を始める前に、必要なツールと環境を整備しましょう。

### 1\. 必要なツールのインストール

```bash
# Google Cloud SDK のインストール（公式サイトからダウンロード）
# https://cloud.google.com/sdk/docs/install

# Cloud SDK の初期化とログイン
gcloud init
gcloud auth login

# App Engine コンポーネントのインストール
gcloud components install app-engine-nodejs

# プロジェクトの作成と設定
gcloud projects create your-project-id --name="My App Engine Project"
gcloud config set project your-project-id

# App Engine アプリケーションの初期化（リージョンを選択）
gcloud app create --region=asia-northeast1
```

### 2\. TypeScriptプロジェクトの初期化

```bash
# プロジェクトディレクトリの作成
mkdir my-appengine-app && cd my-appengine-app

# Node.jsプロジェクトの初期化
npm init -y

# TypeScript関連の依存関係をインストール
npm install express
npm install -D typescript @types/node @types/express ts-node nodemon

# 本番ビルド用の依存関係
npm install -D @typescript-eslint/eslint-plugin @typescript-eslint/parser eslint
```

**📝 TypeScript設定ファイルの作成**

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020"],
    "module": "commonjs",
    "moduleResolution": "node",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true,
    "removeComments": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

##  アプリケーションの実装

実用的なApp EngineアプリケーションをTypeScriptで構築してみましょう。

### 1\. 基本的なWebサーバーの実装

```typescript:src/server.ts
import express from 'express';
import { Request, Response, NextFunction } from 'express';

const app = express();
const PORT = process.env.PORT || 8080;

// ミドルウェアの設定
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// ヘルスチェック用エンドポイント
app.get('/health', (req: Request, res: Response) => {
  res.json({ 
    status: 'OK', 
    timestamp: new Date().toISOString(),
    version: process.env.GAE_VERSION || 'local'
  });
});

// ルートエンドポイント
app.get('/', (req: Request, res: Response) => {
  const responseData = {
    message: 'Hello from App Engine!',
    environment: process.env.GAE_ENV || 'development',
    service: process.env.GAE_SERVICE || 'default',
    instance: process.env.GAE_INSTANCE || 'local'
  };
  
  res.json(responseData);
});

// エラーハンドリングミドルウェア
app.use((err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error('Error occurred:', err.message);
  res.status(500).json({ 
    error: 'Internal Server Error',
    timestamp: new Date().toISOString()
  });
});

// サーバー起動
app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
  console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
});

export default app;
```

### 2\. データベース連携の実装例

App Engineは**Datastore**というNoSQLデータベースとの連携が非常に簡単です。

```bash
# Datastore用のライブラリをインストール
npm install @google-cloud/datastore
npm install -D @types/google-cloud__datastore
```

```typescript:src/services/userService.ts
import { Datastore } from '@google-cloud/datastore';

const datastore = new Datastore();

export interface User {
  id?: number;
  name: string;
  email: string;
  createdAt: Date;
}

export class UserService {
  private readonly kind = 'User';

  async createUser(userData: Omit<User, 'id' | 'createdAt'>): Promise<User> {
    const key = datastore.key(this.kind);
    const user: User = {
      ...userData,
      createdAt: new Date()
    };

    const entity = {
      key,
      data: user
    };

    try {
      await datastore.save(entity);
      return { ...user, id: key.id as number };
    } catch (error) {
      console.error('Failed to create user:', error);
      throw new Error('User creation failed');
    }
  }

  async getUser(userId: number): Promise<User | null> {
    const key = datastore.key([this.kind, userId]);

    try {
      const [entity] = await datastore.get(key);
      if (!entity) {
        return null;
      }

      return {
        id: key.id as number,
        name: entity.name,
        email: entity.email,
        createdAt: entity.createdAt
      };
    } catch (error) {
      console.error('Failed to get user:', error);
      throw new Error('User retrieval failed');
    }
  }

  async getAllUsers(): Promise<User[]> {
    const query = datastore.createQuery(this.kind);

    try {
      const [entities] = await datastore.runQuery(query);
      return entities.map(entity => ({
        id: entity[datastore.KEY].id as number,
        name: entity.name,
        email: entity.email,
        createdAt: entity.createdAt
      }));
    } catch (error) {
      console.error('Failed to get users:', error);
      throw new Error('Users retrieval failed');
    }
  }
}
```

### 3\. APIルートの実装

```typescript:src/routes/userRoutes.ts
import express from 'express';
import { UserService } from '../services/userService';

const router = express.Router();
const userService = new UserService();

// ユーザー作成
router.post('/users', async (req, res) => {
  try {
    const { name, email } = req.body;
    
    if (!name || !email) {
      return res.status(400).json({ error: 'Name and email are required' });
    }

    const user = await userService.createUser({ name, email });
    res.status(201).json(user);
  } catch (error) {
    console.error('Error creating user:', error);
    res.status(500).json({ error: 'Failed to create user' });
  }
});

// ユーザー取得
router.get('/users/:id', async (req, res) => {
  try {
    const userId = parseInt(req.params.id);
    const user = await userService.getUser(userId);
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    res.json(user);
  } catch (error) {
    console.error('Error getting user:', error);
    res.status(500).json({ error: 'Failed to get user' });
  }
});

// 全ユーザー取得
router.get('/users', async (req, res) => {
  try {
    const users = await userService.getAllUsers();
    res.json(users);
  } catch (error) {
    console.error('Error getting users:', error);
    res.status(500).json({ error: 'Failed to get users' });
  }
});

export default router;
```

サーバーファイルにルートを追加：

```typescript:src/server.ts
import express from 'express';
import userRoutes from './routes/userRoutes';

const app = express();
const PORT = process.env.PORT || 8080;

app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// APIルートの設定
app.use('/api', userRoutes);

// 以下、既存のコード...
```

## App Engine設定とデプロイ

### 1\. app.yaml の設定

App Engineでは`app.yaml`ファイルでアプリケーションの動作を制御します。

```yaml:app.yaml
runtime: nodejs18

# 環境変数の設定
env_variables:
  NODE_ENV: production
  LOG_LEVEL: info

# インスタンスの設定
automatic_scaling:
  min_instances: 0
  max_instances: 10
  target_cpu_utilization: 0.6

# リソースの設定
resources:
  cpu: 1
  memory_gb: 0.5

# ヘルスチェックの設定
readiness_check:
  path: "/health"
  check_interval_sec: 5
  timeout_sec: 4
  failure_threshold: 2
  success_threshold: 2

liveness_check:
  path: "/health"
  check_interval_sec: 30
  timeout_sec: 4
  failure_threshold: 4
  success_threshold: 2
```

### 2\. package.jsonの設定

```json:package.json
{
  "name": "my-appengine-app",
  "version": "1.0.0",
  "scripts": {
    "start": "node dist/server.js",
    "build": "tsc",
    "dev": "nodemon --exec ts-node src/server.ts",
    "lint": "eslint src --ext .ts",
    "deploy": "npm run build && gcloud app deploy",
    "deploy:staging": "npm run build && gcloud app deploy --version=staging --no-promote"
  },
  "dependencies": {
    "express": "^4.18.2",
    "@google-cloud/datastore": "^8.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0",
    "@types/express": "^4.17.17",
    "ts-node": "^10.9.0",
    "nodemon": "^3.0.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.45.0"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

### 3\. デプロイの実行

```bash
# 本番環境へのデプロイ
npm run deploy

# ステージング環境へのデプロイ（本番トラフィックには影響しない）
npm run deploy:staging

# デプロイ完了後、アプリケーションを開く
gcloud app browse
```

## 環境管理とベストプラクティス

**🌍 複数環境での管理**

```bash
# 開発用のサービスを作成
gcloud app deploy --version=dev --no-promote

# 本番用のデフォルトサービス
gcloud app deploy --version=prod --promote

# 特定のバージョンに100%のトラフィックを振り分け
gcloud app services set-traffic default --splits=prod=1.0
```

**📊 ログ監視とデバッグ**

```typescript:src/utils/logger.ts
import { Request, Response, NextFunction } from 'express';

export const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  const startTime = Date.now();

  // レスポンス完了時にログを出力
  res.on('finish', () => {
    const duration = Date.now() - startTime;
    console.log(JSON.stringify({
      method: req.method,
      url: req.url,
      statusCode: res.statusCode,
      userAgent: req.get('User-Agent'),
      duration: `${duration}ms`,
      timestamp: new Date().toISOString()
    }));
  });

  next();
};

export const errorLogger = (err: Error, req: Request, res: Response, next: NextFunction) => {
  console.error(JSON.stringify({
    error: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
    timestamp: new Date().toISOString()
  }));
  
  next(err);
};
```

## コスト最適化のポイント

**1. インスタンスの適切な設定**

```yaml:app.yaml
# コスト削減のための設定例
automatic_scaling:
  min_instances: 0  # アイドル時は0インスタンス
  max_instances: 5  # 予期せぬスケールアウトを防ぐ
  target_cpu_utilization: 0.8  # CPU使用率を高めに設定
```

**2. 不要なバージョンの削除**

```bash
# 古いバージョンの一覧表示
gcloud app versions list

# 不要なバージョンを削除してインスタンス料金を節約
gcloud app versions delete v1 v2 --service=default
```

**3. パフォーマンス最適化**

```typescript:src/server.ts
// レスポンス圧縮の有効化
import compression from 'compression';
app.use(compression());

// 静的ファイルのキャッシュ設定
app.use('/static', express.static('public', {
  maxAge: '1d', // 1日キャッシュ
  etag: false
}));
```

## セキュリティとパフォーマンス

**🔒 セキュリティ対策**

```typescript:src/middleware/security.ts
import helmet from 'helmet';
import rateLimit from 'express-rate-limit';

// セキュリティヘッダーの設定
export const securityMiddleware = helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: ["'self'", "'unsafe-inline'"],
      scriptSrc: ["'self'"],
      imgSrc: ["'self'", "data:", "https:"],
    },
  },
});

// レート制限の設定
export const rateLimitMiddleware = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // 最大100リクエスト/15分
  message: 'Too many requests from this IP',
  standardHeaders: true,
  legacyHeaders: false,
});
```

**⚡ パフォーマンス監視**

```typescript:src/middleware/monitoring.ts
import { Request, Response, NextFunction } from 'express';

export const performanceMonitoring = (req: Request, res: Response, next: NextFunction) => {
  const startTime = process.hrtime.bigint();

  res.on('finish', () => {
    const endTime = process.hrtime.bigint();
    const duration = Number(endTime - startTime) / 1000000; // ナノ秒からミリ秒に変換

    // App Engineの標準ログ形式で出力
    console.log({
      severity: 'INFO',
      message: 'Request completed',
      httpRequest: {
        requestMethod: req.method,
        requestUrl: req.url,
        status: res.statusCode,
        latency: `${duration.toFixed(2)}ms`
      }
    });
  });

  next();
};
```

## まとめ

App Engine × TypeScript の組み合わせで、サーバー管理の煩わしさから解放されたWebアプリケーション開発が実現できました。

**✅ この記事で身についたスキル**

>* App Engineの基本概念と他サービスとの使い分け
>* TypeScript + Expressでの実用的なAPI開発手法
>* Datastoreを使ったデータ永続化パターン
>* 本番運用に必要なログ監視・セキュリティ対策
>* コスト最適化とパフォーマンスチューニングのベストプラクティス

**🎯 次のステップ**

>* Cloud SQLやPub/Subなど他のGCPサービスとの連携
>* CI/CDパイプラインの構築（Cloud Build + GitHub Actions）
>* フロントエンド（React/Vue）との組み合わせでフルスタックアプリケーション構築

App Engineなら、アイデアをすぐに形にして世界中のユーザーに届けることができます。あなたも今日からサーバーレス開発を始めてみませんか？