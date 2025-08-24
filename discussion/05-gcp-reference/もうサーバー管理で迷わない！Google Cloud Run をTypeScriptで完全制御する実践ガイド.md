## もうサーバー管理で迷わない！Google Cloud Run をTypeScriptで完全制御する実践ガイド

## はじめに

「サーバーのスケーリングやメンテナンスが複雑すぎて、アプリケーション開発に集中できない」

そんな悩みを抱える開発者の方に朗報です。Google Cloud Run なら、**TypeScript**を使ってコンテナベースのアプリケーションを完全サーバーレスで運用でき、スケーリングからインフラ管理まで全てをGoogleが自動化してくれます。

この記事では、**Cloud Run API** と **TypeScript** を組み合わせてサービスのデプロイからトラフィック管理、監視まで全てをプログラマブルに制御し、実際の運用で使える実践的なコードパターンを解説します。記事を読み終える頃には、あなたもServerless as Code の真髄を体感できているはずです 🚀

>* Google Cloud Run の基本概念とメリット・デメリットの理解
>* TypeScript + Cloud Run API でのサービス管理自動化
>* トラフィック制御、リビジョン管理、CI/CD統合の実装
>* 監視・ログ管理・コスト最適化の設定パターン

## Google Cloud Run とは？

Google Cloud Run は、Googleが提供する**フルマネージドなサーバーレスコンテナプラットフォーム**です。

**1. ゼロからのスケーラビリティ**

>* リクエスト数に応じて自動スケール（0〜1000インスタンス）
>* コールドスタート時間を最小化する高速起動
>* トラフィックがない時は完全に0にスケールダウン

**2. シンプルなコンテナデプロイ**

>* Dockerfile があれば即座にデプロイ可能
>* Docker、Buildpacks、Cloud Build との完全統合
>* ポータビリティ - どこでも動作するコンテナ

**3. 運用負荷ゼロ**

>* インフラ管理不要（パッチ適用、セキュリティ更新が自動）
>* 従量課金制 - 実際のリクエスト処理時間のみ課金
>* 99.95%のSLA保証

**⚖️ 他のサーバーレスプラットフォームとの比較**

| サービス | 実行環境 | スケール範囲 | 料金体系 | 適用場面 |
|---------|---------|-------------|----------|---------|
| **Cloud Run** | コンテナ | 0-1000 | 実行時間 + リクエスト数 | Webアプリ・API |
| **AWS Lambda** | 関数 | 0-1000 | 実行時間のみ | 短時間処理 |
| **App Engine** | ランタイム限定 | 0-自動 | インスタンス時間 | Webアプリケーション |

**📌 Cloud Runを選ぶべきシーン**

>* 既存のコンテナ化されたアプリケーションをサーバーレス化したい
>* トラフィックが不定期で、コスト効率を重視したい
>* Kubernetesの管理負荷なしでコンテナオーケストレーションを使いたい

## 開発環境のセットアップ

Cloud RunをTypeScriptで管理するための開発環境を構築しましょう。

**📦 必要なツールのインストール**

```bash
# Google Cloud SDK のインストール（公式サイトからダウンロード）
# https://cloud.google.com/sdk/docs/install

# Cloud SDK の初期化とログイン
gcloud init
gcloud auth login
gcloud auth application-default login

# プロジェクトの作成と設定
gcloud projects create your-cloudrun-project --name="My Cloud Run Project"
gcloud config set project your-cloudrun-project

# 必要なAPIの有効化
gcloud services enable run.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable containerregistry.googleapis.com

# Dockerのセットアップ
gcloud auth configure-docker
```

**🚀 TypeScriptプロジェクトの初期化**

```bash
# プロジェクトディレクトリの作成
mkdir cloudrun-typescript-manager && cd cloudrun-typescript-manager

# Node.jsプロジェクトの初期化
npm init -y

# Google Cloud Client Library のインストール
npm install @google-cloud/run
npm install express cors helmet
npm install -D typescript @types/node @types/express ts-node nodemon

# 設定管理・ユーティリティライブラリ
npm install dotenv winston
npm install -D @types/dotenv
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

## 2. Cloud Run サービス管理の実装

実用的なCloud Runサービス管理システムをTypeScriptで構築してみましょう。

### 1\. 基本的なサービス操作クラス

```typescript:src/services/CloudRunManager.ts
import { ServicesClient } from '@google-cloud/run';
import { logger } from '../utils/logger';

export interface ServiceConfig {
  name: string;
  location: string;
  image: string;
  port?: number;
  memory?: string;
  cpu?: string;
  maxInstances?: number;
  minInstances?: number;
  concurrency?: number;
  timeout?: number;
  env?: Record<string, string>;
  labels?: Record<string, string>;
  allowUnauthenticated?: boolean;
}

export interface ServiceInfo {
  name: string;
  location: string;
  url: string;
  image: string;
  status: string;
  traffic: TrafficTarget[];
  createdAt: string;
  updatedAt: string;
  latestRevision: string;
}

export interface TrafficTarget {
  revisionName?: string;
  percent: number;
  latestRevision?: boolean;
  tag?: string;
  url?: string;
}

export class CloudRunManager {
  private client: ServicesClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.client = new ServicesClient();
  }

  async deployService(config: ServiceConfig): Promise<ServiceInfo> {
    const {
      name,
      location,
      image,
      port = 8080,
      memory = '512Mi',
      cpu = '1000m',
      maxInstances = 100,
      minInstances = 0,
      concurrency = 80,
      timeout = 300,
      env = {},
      labels = {},
      allowUnauthenticated = false,
    } = config;

    const parent = `projects/${this.projectId}/locations/${location}`;
    
    // 環境変数を Cloud Run フォーマットに変換
    const envVars = Object.entries(env).map(([name, value]) => ({
      name,
      value,
    }));

    const service = {
      metadata: {
        name,
        labels: {
          'cloud.googleapis.com/location': location,
          ...labels,
        },
        annotations: {
          'run.googleapis.com/ingress': 'all',
          'run.googleapis.com/ingress-status': 'all',
        },
      },
      spec: {
        template: {
          metadata: {
            annotations: {
              'autoscaling.knative.dev/maxScale': maxInstances.toString(),
              'autoscaling.knative.dev/minScale': minInstances.toString(),
              'run.googleapis.com/cpu-throttling': 'true',
              'run.googleapis.com/execution-environment': 'gen2',
            },
          },
          spec: {
            containerConcurrency: concurrency,
            timeoutSeconds: timeout,
            containers: [
              {
                image,
                ports: [
                  {
                    containerPort: port,
                    name: 'http1',
                  },
                ],
                env: envVars,
                resources: {
                  limits: {
                    cpu,
                    memory,
                  },
                },
              },
            ],
          },
        },
        traffic: [
          {
            percent: 100,
            latestRevision: true,
          },
        ],
      },
    };

    try {
      logger.info(`Deploying service: ${name} in location: ${location}`);
      
      const [operation] = await this.client.replaceService({
        service,
      });

      logger.info(`Service deployment operation started: ${operation.name}`);
      
      // オペレーション完了を待つ
      await this.waitForOperation(operation.name!);
      
      // デプロイ後のサービス情報を取得
      const serviceInfo = await this.getService(name, location);
      
      // 認証なしアクセスの設定
      if (allowUnauthenticated) {
        await this.setIamPolicy(name, location);
      }
      
      logger.info(`Successfully deployed service: ${name}`);
      return serviceInfo;

    } catch (error) {
      logger.error(`Failed to deploy service ${name}:`, error);
      throw new Error(`Service deployment failed: ${error.message}`);
    }
  }

  async getService(name: string, location: string): Promise<ServiceInfo> {
    try {
      const serviceName = `projects/${this.projectId}/locations/${location}/services/${name}`;
      
      const [service] = await this.client.getService({
        name: serviceName,
      });

      return {
        name: service.metadata?.name || '',
        location: service.metadata?.labels?.['cloud.googleapis.com/location'] || location,
        url: service.status?.url || '',
        image: service.spec?.template?.spec?.containers?.[0]?.image || '',
        status: this.getServiceStatus(service),
        traffic: service.status?.traffic || [],
        createdAt: service.metadata?.creationTimestamp || '',
        updatedAt: service.metadata?.generation?.toString() || '',
        latestRevision: service.status?.latestCreatedRevisionName || '',
      };

    } catch (error) {
      logger.error(`Failed to get service ${name}:`, error);
      throw new Error(`Service retrieval failed: ${error.message}`);
    }
  }

  async listServices(location: string): Promise<ServiceInfo[]> {
    try {
      const parent = `projects/${this.projectId}/locations/${location}`;
      
      const [services] = await this.client.listServices({ parent });

      return services.map(service => ({
        name: service.metadata?.name || '',
        location: service.metadata?.labels?.['cloud.googleapis.com/location'] || location,
        url: service.status?.url || '',
        image: service.spec?.template?.spec?.containers?.[0]?.image || '',
        status: this.getServiceStatus(service),
        traffic: service.status?.traffic || [],
        createdAt: service.metadata?.creationTimestamp || '',
        updatedAt: service.metadata?.generation?.toString() || '',
        latestRevision: service.status?.latestCreatedRevisionName || '',
      }));

    } catch (error) {
      logger.error(`Failed to list services in location ${location}:`, error);
      throw new Error(`Service listing failed: ${error.message}`);
    }
  }

  async deleteService(name: string, location: string): Promise<void> {
    try {
      const serviceName = `projects/${this.projectId}/locations/${location}/services/${name}`;
      
      logger.info(`Deleting service: ${name} in location: ${location}`);
      
      const [operation] = await this.client.deleteService({
        name: serviceName,
      });

      await this.waitForOperation(operation.name!);
      
      logger.info(`Successfully deleted service: ${name}`);

    } catch (error) {
      logger.error(`Failed to delete service ${name}:`, error);
      throw new Error(`Service deletion failed: ${error.message}`);
    }
  }

  async updateTraffic(
    name: string, 
    location: string, 
    trafficTargets: TrafficTarget[]
  ): Promise<void> {
    try {
      const serviceName = `projects/${this.projectId}/locations/${location}/services/${name}`;
      
      // 現在のサービス設定を取得
      const [currentService] = await this.client.getService({
        name: serviceName,
      });

      // トラフィック設定のみを更新
      currentService.spec!.traffic = trafficTargets.map(target => ({
        percent: target.percent,
        ...(target.revisionName && { revisionName: target.revisionName }),
        ...(target.latestRevision && { latestRevision: target.latestRevision }),
        ...(target.tag && { tag: target.tag }),
      }));

      logger.info(`Updating traffic for service: ${name}`);
      
      const [operation] = await this.client.replaceService({
        service: currentService,
      });

      await this.waitForOperation(operation.name!);
      
      logger.info(`Successfully updated traffic for service: ${name}`);

    } catch (error) {
      logger.error(`Failed to update traffic for service ${name}:`, error);
      throw new Error(`Traffic update failed: ${error.message}`);
    }
  }

  private getServiceStatus(service: any): string {
    const conditions = service.status?.conditions || [];
    const readyCondition = conditions.find((c: any) => c.type === 'Ready');
    
    if (readyCondition?.status === 'True') {
      return 'Ready';
    } else if (readyCondition?.status === 'False') {
      return `NotReady: ${readyCondition.message || 'Unknown error'}`;
    } else {
      return 'Unknown';
    }
  }

  private async setIamPolicy(serviceName: string, location: string): Promise<void> {
    try {
      // Cloud Run サービスに allUsers の Cloud Run Invoker 権限を付与
      const resourceName = `projects/${this.projectId}/locations/${location}/services/${serviceName}`;
      
      // 実際の実装では @google-cloud/iam を使用
      logger.info(`Setting IAM policy for service: ${serviceName}`);
      
    } catch (error) {
      logger.warn(`Failed to set IAM policy for service ${serviceName}:`, error.message);
    }
  }

  private async waitForOperation(operationName: string): Promise<void> {
    // オペレーション完了待ちの実装
    // 実際の実装では適切なポーリングロジックを実装
    logger.info(`Waiting for operation: ${operationName}`);
    
    // 簡単な実装例（実際にはより堅牢な実装が必要）
    await new Promise(resolve => setTimeout(resolve, 10000));
  }
}
```

### 2\. リビジョン管理とトラフィック制御

```typescript:src/services/TrafficManager.ts
import { CloudRunManager, TrafficTarget } from './CloudRunManager';
import { logger } from '../utils/logger';

export interface DeploymentStrategy {
  type: 'rolling' | 'blue-green' | 'canary';
  canaryPercent?: number;
  rolloutDuration?: number;
}

export interface RollbackConfig {
  targetRevision: string;
  reason?: string;
}

export class TrafficManager {
  private cloudRunManager: CloudRunManager;

  constructor(projectId: string) {
    this.cloudRunManager = new CloudRunManager(projectId);
  }

  async deployWithStrategy(
    serviceName: string, 
    location: string, 
    strategy: DeploymentStrategy
  ): Promise<void> {
    try {
      const service = await this.cloudRunManager.getService(serviceName, location);
      
      switch (strategy.type) {
        case 'rolling':
          await this.performRollingDeployment(serviceName, location);
          break;
        case 'blue-green':
          await this.performBlueGreenDeployment(serviceName, location);
          break;
        case 'canary':
          await this.performCanaryDeployment(
            serviceName, 
            location, 
            strategy.canaryPercent || 10
          );
          break;
      }
    } catch (error) {
      logger.error(`Failed to deploy with strategy ${strategy.type}:`, error);
      throw new Error(`Deployment failed: ${error.message}`);
    }
  }

  private async performRollingDeployment(
    serviceName: string, 
    location: string
  ): Promise<void> {
    logger.info(`Performing rolling deployment for ${serviceName}`);
    
    // ローリングデプロイメント: 新しいリビジョンに100%のトラフィックを向ける
    const trafficTargets: TrafficTarget[] = [
      {
        percent: 100,
        latestRevision: true,
      },
    ];

    await this.cloudRunManager.updateTraffic(serviceName, location, trafficTargets);
    logger.info(`Rolling deployment completed for ${serviceName}`);
  }

  private async performBlueGreenDeployment(
    serviceName: string, 
    location: string
  ): Promise<void> {
    logger.info(`Performing blue-green deployment for ${serviceName}`);
    
    const service = await this.cloudRunManager.getService(serviceName, location);
    const currentRevision = service.latestRevision;

    // 段階1: 新しいリビジョンに0%のトラフィックを割り当て
    let trafficTargets: TrafficTarget[] = [
      {
        percent: 100,
        revisionName: currentRevision,
      },
      {
        percent: 0,
        latestRevision: true,
        tag: 'staging',
      },
    ];

    await this.cloudRunManager.updateTraffic(serviceName, location, trafficTargets);
    
    // ここで新しいバージョンのテストを実行
    await this.validateNewRevision(serviceName, location);
    
    // 段階2: 新しいリビジョンに100%のトラフィックを切り替え
    trafficTargets = [
      {
        percent: 100,
        latestRevision: true,
      },
    ];

    await this.cloudRunManager.updateTraffic(serviceName, location, trafficTargets);
    logger.info(`Blue-green deployment completed for ${serviceName}`);
  }

  private async performCanaryDeployment(
    serviceName: string, 
    location: string, 
    canaryPercent: number
  ): Promise<void> {
    logger.info(`Performing canary deployment for ${serviceName} with ${canaryPercent}% traffic`);
    
    const service = await this.cloudRunManager.getService(serviceName, location);
    const currentRevision = service.latestRevision;

    // 段階1: Canaryリビジョンに指定された割合のトラフィックを割り当て
    let trafficTargets: TrafficTarget[] = [
      {
        percent: 100 - canaryPercent,
        revisionName: currentRevision,
      },
      {
        percent: canaryPercent,
        latestRevision: true,
        tag: 'canary',
      },
    ];

    await this.cloudRunManager.updateTraffic(serviceName, location, trafficTargets);
    
    // Canaryメトリクスの監視期間
    logger.info(`Monitoring canary deployment for ${serviceName}...`);
    await this.monitorCanaryMetrics(serviceName, location);
    
    // 段階2: 問題がなければ100%に切り替え
    trafficTargets = [
      {
        percent: 100,
        latestRevision: true,
      },
    ];

    await this.cloudRunManager.updateTraffic(serviceName, location, trafficTargets);
    logger.info(`Canary deployment completed for ${serviceName}`);
  }

  async rollbackService(
    serviceName: string, 
    location: string, 
    config: RollbackConfig
  ): Promise<void> {
    try {
      logger.info(`Rolling back ${serviceName} to revision: ${config.targetRevision}`);
      
      const trafficTargets: TrafficTarget[] = [
        {
          percent: 100,
          revisionName: config.targetRevision,
        },
      ];

      await this.cloudRunManager.updateTraffic(serviceName, location, trafficTargets);
      
      logger.info(`Successfully rolled back ${serviceName}. Reason: ${config.reason || 'Manual rollback'}`);

    } catch (error) {
      logger.error(`Failed to rollback service ${serviceName}:`, error);
      throw new Error(`Rollback failed: ${error.message}`);
    }
  }

  private async validateNewRevision(serviceName: string, location: string): Promise<void> {
    // 新しいリビジョンの検証ロジック
    logger.info(`Validating new revision for ${serviceName}`);
    
    // ヘルスチェックやスモークテストを実行
    // 実際の実装では、staging URLにリクエストを送信してレスポンスを確認
    await new Promise(resolve => setTimeout(resolve, 5000)); // 5秒の検証時間
  }

  private async monitorCanaryMetrics(serviceName: string, location: string): Promise<void> {
    // Canaryメトリクスの監視
    logger.info(`Monitoring canary metrics for ${serviceName}`);
    
    // 実際の実装では、Cloud Monitoring APIを使ってエラー率やレスポンス時間を監視
    await new Promise(resolve => setTimeout(resolve, 30000)); // 30秒の監視期間
  }
}
```

### 3\. 実用的なWebアプリケーション例

```typescript:src/apps/web-server.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { logger } from '../utils/logger';

const app = express();
const PORT = process.env.PORT || 8080;

// セキュリティミドルウェア
app.use(helmet());
app.use(cors());
app.use(express.json());

// ヘルスチェックエンドポイント
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    version: process.env.SERVICE_VERSION || '1.0.0',
  });
});

// メインAPIエンドポイント
app.get('/api/hello', (req, res) => {
  const name = req.query.name || 'World';
  logger.info(`Hello endpoint called with name: ${name}`);
  
  res.json({
    message: `Hello, ${name}!`,
    timestamp: new Date().toISOString(),
    instance: process.env.K_SERVICE || 'local',
  });
});

// データ処理エンドポイント
app.post('/api/process', async (req, res) => {
  try {
    const { data } = req.body;
    
    if (!data) {
      return res.status(400).json({ error: 'Data is required' });
    }

    logger.info(`Processing data: ${JSON.stringify(data)}`);
    
    // データ処理のシミュレーション
    const result = await processData(data);
    
    res.json({
      success: true,
      result,
      processedAt: new Date().toISOString(),
    });

  } catch (error) {
    logger.error('Error processing data:', error);
    res.status(500).json({
      error: 'Internal server error',
      message: error.message,
    });
  }
});

// グレースフルシャットダウン
process.on('SIGINT', () => {
  logger.info('Received SIGINT, shutting down gracefully');
  process.exit(0);
});

process.on('SIGTERM', () => {
  logger.info('Received SIGTERM, shutting down gracefully');
  process.exit(0);
});

async function processData(data: any): Promise<any> {
  // 実際のデータ処理ロジック
  return new Promise(resolve => {
    setTimeout(() => {
      resolve({
        processed: true,
        originalData: data,
        processedData: `Processed: ${JSON.stringify(data)}`,
      });
    }, 1000);
  });
}

app.listen(PORT, () => {
  logger.info(`Server is running on port ${PORT}`);
  logger.info(`Environment: ${process.env.NODE_ENV || 'development'}`);
});
```

### 4\. デプロイメント設定

```dockerfile:Dockerfile
# マルチステージビルドでイメージサイズを最適化
FROM node:18-slim AS build

WORKDIR /app

# パッケージファイルをコピー（キャッシュ活用）
COPY package*.json ./
RUN npm ci --only=production

# TypeScriptのビルド用依存関係をインストール
COPY package*.json ./
RUN npm ci

# ソースコードをコピーしてビルド
COPY . .
RUN npm run build

# 本番用の軽量イメージ
FROM node:18-slim

WORKDIR /app

# 必要なファイルのみをコピー
COPY --from=build /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY --from=build /app/package*.json ./

# 非rootユーザーで実行（セキュリティ強化）
RUN groupadd -r appuser && useradd -r -g appuser appuser
RUN chown -R appuser:appuser /app
USER appuser

# ポートを公開
EXPOSE 8080

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

# アプリケーション起動
CMD ["node", "dist/apps/web-server.js"]
```

### 5\. 実用的な使用例とデプロイスクリプト

```typescript:src/examples/deploymentWorkflow.ts
import { CloudRunManager, TrafficManager } from '../services';
import { logger } from '../utils/logger';

async function deployWebApplication() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const location = 'asia-northeast1';
  
  const cloudRunManager = new CloudRunManager(projectId);
  const trafficManager = new TrafficManager(projectId);

  // 本番環境のWebアプリケーション設定
  const productionConfig = {
    name: 'web-app-prod',
    location,
    image: `gcr.io/${projectId}/web-app:latest`,
    port: 8080,
    memory: '512Mi',
    cpu: '1000m',
    maxInstances: 100,
    minInstances: 0,
    concurrency: 80,
    timeout: 300,
    env: {
      NODE_ENV: 'production',
      LOG_LEVEL: 'info',
      DATABASE_URL: process.env.DATABASE_URL || '',
    },
    labels: {
      environment: 'production',
      version: 'v1.0.0',
    },
    allowUnauthenticated: true,
  };

  try {
    // 1. サービスのデプロイ
    logger.info('Deploying production web application...');
    const service = await cloudRunManager.deployService(productionConfig);
    logger.info(`Service deployed: ${service.url}`);

    // 2. Canaryデプロイメントの実行
    await trafficManager.deployWithStrategy(
      productionConfig.name,
      location,
      {
        type: 'canary',
        canaryPercent: 20,
      }
    );

    // 3. APIサービスのデプロイ
    const apiConfig = {
      name: 'api-service',
      location,
      image: `gcr.io/${projectId}/api-service:latest`,
      port: 8080,
      memory: '1Gi',
      cpu: '2000m',
      maxInstances: 50,
      minInstances: 1, // API は常に1インスタンス稼働
      concurrency: 100,
      timeout: 600,
      env: {
        NODE_ENV: 'production',
        LOG_LEVEL: 'warn',
        REDIS_URL: process.env.REDIS_URL || '',
      },
      labels: {
        environment: 'production',
        service: 'api',
      },
    };

    const apiService = await cloudRunManager.deployService(apiConfig);
    logger.info(`API service deployed: ${apiService.url}`);

    // 4. バッチジョブサービスの作成
    const batchConfig = {
      name: 'batch-processor',
      location,
      image: `gcr.io/${projectId}/batch-processor:latest`,
      port: 8080,
      memory: '2Gi',
      cpu: '2000m',
      maxInstances: 10,
      minInstances: 0,
      concurrency: 1, // バッチ処理は1並行に制限
      timeout: 3600, // 1時間のタイムアウト
      env: {
        NODE_ENV: 'production',
        BATCH_SIZE: '1000',
        LOG_LEVEL: 'info',
      },
      labels: {
        environment: 'production',
        type: 'batch',
      },
    };

    const batchService = await cloudRunManager.deployService(batchConfig);
    logger.info(`Batch service deployed: ${batchService.url}`);

    // 5. サービス一覧の確認
    const services = await cloudRunManager.listServices(location);
    logger.info(`Total services deployed: ${services.length}`);
    
    services.forEach(svc => {
      logger.info(`- ${svc.name}: ${svc.status} (${svc.url})`);
    });

  } catch (error) {
    logger.error('Deployment failed:', error);
    throw error;
  }
}

async function performEmergencyRollback() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const location = 'asia-northeast1';
  const serviceName = 'web-app-prod';
  
  const trafficManager = new TrafficManager(projectId);

  try {
    // 前の安定版リビジョンに緊急ロールバック
    await trafficManager.rollbackService(
      serviceName,
      location,
      {
        targetRevision: 'web-app-prod-00002-abc', // 前の安定版リビジョン
        reason: 'Emergency rollback due to production issues',
      }
    );

    logger.info('Emergency rollback completed successfully');

  } catch (error) {
    logger.error('Emergency rollback failed:', error);
    throw error;
  }
}

// Blue-Green デプロイメントの例
async function performBlueGreenDeployment() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const location = 'asia-northeast1';
  const serviceName = 'web-app-prod';
  
  const trafficManager = new TrafficManager(projectId);

  try {
    logger.info('Starting blue-green deployment...');
    
    await trafficManager.deployWithStrategy(
      serviceName,
      location,
      {
        type: 'blue-green',
      }
    );

    logger.info('Blue-green deployment completed successfully');

  } catch (error) {
    logger.error('Blue-green deployment failed:', error);
    
    // 失敗時は自動ロールバック
    await performEmergencyRollback();
    throw error;
  }
}

// 実行例
if (require.main === module) {
  const operation = process.argv[2];
  
  switch (operation) {
    case 'deploy':
      deployWebApplication();
      break;
    case 'rollback':
      performEmergencyRollback();
      break;
    case 'blue-green':
      performBlueGreenDeployment();
      break;
    default:
      console.log('Usage: npm run deploy [deploy|rollback|blue-green]');
  }
}
```

## Cloud Monitoring との連携

```typescript:src/services/CloudRunMonitoringService.ts
import { MetricServiceClient } from '@google-cloud/monitoring';
import { logger } from '../utils/logger';

export interface ServiceMetrics {
  serviceName: string;
  location: string;
  requestCount: number;
  errorRate: number;
  averageLatency: number;
  instanceCount: number;
  cpuUtilization: number;
  memoryUtilization: number;
  timestamp: Date;
}

export class CloudRunMonitoringService {
  private client: MetricServiceClient;
  private projectId: string;
  private projectPath: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.client = new MetricServiceClient();
    this.projectPath = this.client.projectPath(projectId);
  }

  async getServiceMetrics(
    serviceName: string, 
    location: string, 
    hours = 1
  ): Promise<ServiceMetrics[]> {
    const now = new Date();
    const startTime = new Date(now.getTime() - hours * 60 * 60 * 1000);

    try {
      // リクエスト数の取得
      const requestCountMetrics = await this.getMetric(
        'run.googleapis.com/request_count',
        serviceName,
        location,
        startTime,
        now
      );

      // エラー率の取得
      const errorMetrics = await this.getMetric(
        'run.googleapis.com/request_count',
        serviceName,
        location,
        startTime,
        now,
        'response_code_class="5xx"'
      );

      // レスポンス時間の取得
      const latencyMetrics = await this.getMetric(
        'run.googleapis.com/request_latencies',
        serviceName,
        location,
        startTime,
        now
      );

      // CPU使用率の取得
      const cpuMetrics = await this.getMetric(
        'run.googleapis.com/container/cpu/utilizations',
        serviceName,
        location,
        startTime,
        now
      );

      // メモリ使用率の取得
      const memoryMetrics = await this.getMetric(
        'run.googleapis.com/container/memory/utilizations',
        serviceName,
        location,
        startTime,
        now
      );

      // メトリクスを統合
      const metrics: ServiceMetrics[] = [];
      
      for (let i = 0; i < requestCountMetrics.length; i++) {
        metrics.push({
          serviceName,
          location,
          requestCount: requestCountMetrics[i]?.value || 0,
          errorRate: this.calculateErrorRate(requestCountMetrics[i], errorMetrics[i]),
          averageLatency: latencyMetrics[i]?.value || 0,
          instanceCount: 1, // 実際の実装では適切に取得
          cpuUtilization: cpuMetrics[i]?.value || 0,
          memoryUtilization: memoryMetrics[i]?.value || 0,
          timestamp: new Date(requestCountMetrics[i]?.timestamp || now),
        });
      }

      return metrics;

    } catch (error) {
      logger.error(`Failed to get metrics for service ${serviceName}:`, error);
      throw new Error(`Metrics retrieval failed: ${error.message}`);
    }
  }

  async createCustomAlert(serviceName: string, location: string): Promise<void> {
    const alertPolicy = {
      displayName: `Cloud Run ${serviceName} High Error Rate`,
      conditions: [
        {
          displayName: 'High error rate condition',
          conditionThreshold: {
            filter: `
              resource.type="cloud_run_revision" AND 
              resource.labels.service_name="${serviceName}" AND 
              resource.labels.location="${location}" AND
              metric.type="run.googleapis.com/request_count" AND
              metric.labels.response_code_class="5xx"
            `,
            comparison: 'COMPARISON_GREATER_THAN',
            thresholdValue: 0.05, // 5%のエラー率
            duration: {
              seconds: 300, // 5分間
            },
            aggregations: [
              {
                alignmentPeriod: {
                  seconds: 60,
                },
                perSeriesAligner: 'ALIGN_RATE',
                crossSeriesReducer: 'REDUCE_SUM',
                groupByFields: ['resource.labels.service_name'],
              },
            ],
          },
        },
      ],
      enabled: true,
      notificationChannels: [
        // 実際の実装では適切な通知チャンネルを設定
      ],
    };

    try {
      logger.info(`Creating alert policy for service: ${serviceName}`);
      // 実際の実装では AlertPolicyServiceClient を使用
      logger.info(`Alert policy created for service: ${serviceName}`);
    } catch (error) {
      logger.error('Failed to create alert policy:', error);
      throw new Error(`Alert policy creation failed: ${error.message}`);
    }
  }

  private async getMetric(
    metricType: string,
    serviceName: string,
    location: string,
    startTime: Date,
    endTime: Date,
    additionalFilter = ''
  ): Promise<any[]> {
    const filter = `
      resource.type="cloud_run_revision" AND 
      resource.labels.service_name="${serviceName}" AND 
      resource.labels.location="${location}" AND
      metric.type="${metricType}"
      ${additionalFilter ? ' AND ' + additionalFilter : ''}
    `.trim();

    const request = {
      name: this.projectPath,
      filter,
      interval: {
        endTime: { seconds: Math.floor(endTime.getTime() / 1000) },
        startTime: { seconds: Math.floor(startTime.getTime() / 1000) },
      },
      view: 'FULL',
    };

    try {
      const [timeSeries] = await this.client.listTimeSeries(request);
      
      return timeSeries.map(series => ({
        value: series.points?.[0]?.value?.doubleValue || 
               series.points?.[0]?.value?.int64Value || 0,
        timestamp: series.points?.[0]?.interval?.endTime?.seconds! * 1000,
      }));

    } catch (error) {
      logger.warn(`Failed to get metric ${metricType}:`, error.message);
      return [];
    }
  }

  private calculateErrorRate(requestMetric: any, errorMetric: any): number {
    if (!requestMetric || !errorMetric || requestMetric.value === 0) {
      return 0;
    }
    return (errorMetric.value / requestMetric.value) * 100;
  }
}
```

## コスト管理と最適化

```typescript:src/services/CostOptimizationService.ts
import { CloudRunManager } from './CloudRunManager';
import { CloudRunMonitoringService } from './CloudRunMonitoringService';
import { logger } from '../utils/logger';

export interface OptimizationRecommendation {
  serviceName: string;
  location: string;
  recommendation: string;
  impact: 'high' | 'medium' | 'low';
  estimatedSavings: number;
  action: 'scale_down' | 'adjust_memory' | 'adjust_cpu' | 'set_min_instances';
}

export class CostOptimizationService {
  private cloudRunManager: CloudRunManager;
  private monitoringService: CloudRunMonitoringService;

  constructor(projectId: string) {
    this.cloudRunManager = new CloudRunManager(projectId);
    this.monitoringService = new CloudRunMonitoringService(projectId);
  }

  async analyzeServices(location: string): Promise<OptimizationRecommendation[]> {
    try {
      const services = await this.cloudRunManager.listServices(location);
      const recommendations: OptimizationRecommendation[] = [];

      for (const service of services) {
        const serviceMetrics = await this.monitoringService.getServiceMetrics(
          service.name,
          service.location,
          24 // 過去24時間
        );

        const serviceRecommendations = await this.analyzeService(service, serviceMetrics);
        recommendations.push(...serviceRecommendations);
      }

      return recommendations;

    } catch (error) {
      logger.error('Failed to analyze services for cost optimization:', error);
      throw new Error(`Cost analysis failed: ${error.message}`);
    }
  }

  private async analyzeService(
    service: any, 
    metrics: any[]
  ): Promise<OptimizationRecommendation[]> {
    const recommendations: OptimizationRecommendation[] = [];

    if (metrics.length === 0) {
      return recommendations;
    }

    const avgCpuUtilization = metrics.reduce((sum, m) => sum + m.cpuUtilization, 0) / metrics.length;
    const avgMemoryUtilization = metrics.reduce((sum, m) => sum + m.memoryUtilization, 0) / metrics.length;
    const avgRequestCount = metrics.reduce((sum, m) => sum + m.requestCount, 0) / metrics.length;

    // CPU使用率が低い場合
    if (avgCpuUtilization < 20) {
      recommendations.push({
        serviceName: service.name,
        location: service.location,
        recommendation: `CPU使用率が${avgCpuUtilization.toFixed(1)}%と低いため、CPUを削減することを推奨します。`,
        impact: 'medium',
        estimatedSavings: 30,
        action: 'adjust_cpu',
      });
    }

    // メモリ使用率が低い場合
    if (avgMemoryUtilization < 30) {
      recommendations.push({
        serviceName: service.name,
        location: service.location,
        recommendation: `メモリ使用率が${avgMemoryUtilization.toFixed(1)}%と低いため、メモリを削減することを推奨します。`,
        impact: 'medium',
        estimatedSavings: 25,
        action: 'adjust_memory',
      });
    }

    // リクエスト数が極端に少ない場合
    if (avgRequestCount < 10) {
      recommendations.push({
        serviceName: service.name,
        location: service.location,
        recommendation: `リクエスト数が非常に少ない（平均${avgRequestCount.toFixed(1)}req/h）ため、最小インスタンス数を0に設定することを推奨します。`,
        impact: 'high',
        estimatedSavings: 80,
        action: 'set_min_instances',
      });
    }

    return recommendations;
  }

  async applyOptimizations(recommendations: OptimizationRecommendation[]): Promise<void> {
    for (const rec of recommendations) {
      try {
        await this.applyOptimization(rec);
        logger.info(`Applied optimization for ${rec.serviceName}: ${rec.recommendation}`);
      } catch (error) {
        logger.error(`Failed to apply optimization for ${rec.serviceName}:`, error);
      }
    }
  }

  private async applyOptimization(rec: OptimizationRecommendation): Promise<void> {
    const service = await this.cloudRunManager.getService(rec.serviceName, rec.location);
    
    switch (rec.action) {
      case 'adjust_cpu':
        // CPU制限を削減
        await this.updateServiceResources(service, { cpu: '500m' });
        break;
      case 'adjust_memory':
        // メモリ制限を削減
        await this.updateServiceResources(service, { memory: '256Mi' });
        break;
      case 'set_min_instances':
        // 最小インスタンス数を0に設定
        await this.updateServiceInstances(service, { minInstances: 0 });
        break;
      case 'scale_down':
        // 最大インスタンス数を削減
        await this.updateServiceInstances(service, { maxInstances: 10 });
        break;
    }
  }

  private async updateServiceResources(
    service: any, 
    resources: { cpu?: string; memory?: string }
  ): Promise<void> {
    // サービスのリソース設定を更新
    logger.info(`Updating resources for ${service.name}: ${JSON.stringify(resources)}`);
  }

  private async updateServiceInstances(
    service: any, 
    instances: { minInstances?: number; maxInstances?: number }
  ): Promise<void> {
    // サービスのインスタンス設定を更新
    logger.info(`Updating instances for ${service.name}: ${JSON.stringify(instances)}`);
  }

  async generateCostReport(location: string): Promise<any> {
    try {
      const services = await this.cloudRunManager.listServices(location);
      const recommendations = await this.analyzeServices(location);
      
      const report = {
        totalServices: services.length,
        runningServices: services.filter(s => s.status === 'Ready').length,
        recommendations: recommendations.length,
        potentialSavings: recommendations.reduce((sum, rec) => sum + rec.estimatedSavings, 0),
        highImpactRecommendations: recommendations.filter(rec => rec.impact === 'high').length,
        services: services.map(service => ({
          name: service.name,
          status: service.status,
          url: service.url,
          image: service.image,
        })),
        topRecommendations: recommendations
          .sort((a, b) => b.estimatedSavings - a.estimatedSavings)
          .slice(0, 5),
      };

      logger.info('Cost optimization report generated:', {
        totalServices: report.totalServices,
        recommendations: report.recommendations,
        potentialSavings: report.potentialSavings,
      });

      return report;

    } catch (error) {
      logger.error('Failed to generate cost report:', error);
      throw new Error(`Cost report generation failed: ${error.message}`);
    }
  }
}
```

## ユーティリティとヘルパー関数

**🛠️ ログ設定**

```typescript:src/utils/logger.ts
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { 
    service: 'cloudrun-typescript-manager',
    version: process.env.SERVICE_VERSION || '1.0.0',
  },
  transports: [
    new winston.transports.File({ 
      filename: 'logs/error.log', 
      level: 'error' 
    }),
    new winston.transports.File({ 
      filename: 'logs/combined.log' 
    }),
  ],
});

// 開発環境では console にも出力
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    )
  }));
}

export { logger };
```

**設定管理**

```typescript:src/config/index.ts
import dotenv from 'dotenv';

dotenv.config();

export const config = {
  // Google Cloud設定
  projectId: process.env.GOOGLE_CLOUD_PROJECT || '',
  defaultLocation: process.env.DEFAULT_LOCATION || 'asia-northeast1',
  
  // Cloud Run設定
  defaultCpu: process.env.DEFAULT_CPU || '1000m',
  defaultMemory: process.env.DEFAULT_MEMORY || '512Mi',
  defaultMaxInstances: parseInt(process.env.DEFAULT_MAX_INSTANCES || '100'),
  defaultMinInstances: parseInt(process.env.DEFAULT_MIN_INSTANCES || '0'),
  defaultConcurrency: parseInt(process.env.DEFAULT_CONCURRENCY || '80'),
  defaultTimeout: parseInt(process.env.DEFAULT_TIMEOUT || '300'),
  
  // 監視・ログ設定
  logLevel: process.env.LOG_LEVEL || 'info',
  enableMonitoring: process.env.ENABLE_MONITORING === 'true',
  metricsRetentionHours: parseInt(process.env.METRICS_RETENTION_HOURS || '24'),
  
  // セキュリティ設定
  allowUnauthenticated: process.env.ALLOW_UNAUTHENTICATED === 'true',
  
  // コスト最適化設定
  enableCostOptimization: process.env.ENABLE_COST_OPTIMIZATION === 'true',
  costAnalysisInterval: parseInt(process.env.COST_ANALYSIS_INTERVAL || '24'),
};

// 必須設定の検証
if (!config.projectId) {
  throw new Error('GOOGLE_CLOUD_PROJECT environment variable is required');
}
```

**🎯 統合運用スクリプト**

```bash:scripts/deploy.sh
#!/bin/bash

# Cloud Run デプロイスクリプト
set -e

PROJECT_ID=${1:-"your-project-id"}
LOCATION=${2:-"asia-northeast1"}
SERVICE_NAME=${3:-"web-app"}
IMAGE_TAG=${4:-"latest"}

echo "🚀 Deploying to Cloud Run..."
echo "Project: $PROJECT_ID"
echo "Location: $LOCATION"
echo "Service: $SERVICE_NAME"
echo "Image Tag: $IMAGE_TAG"

# 1. Docker イメージのビルド
echo "📦 Building Docker image..."
docker build -t gcr.io/$PROJECT_ID/$SERVICE_NAME:$IMAGE_TAG .

# 2. Container Registry にプッシュ
echo "⬆️ Pushing to Container Registry..."
docker push gcr.io/$PROJECT_ID/$SERVICE_NAME:$IMAGE_TAG

# 3. Cloud Run にデプロイ
echo "🌐 Deploying to Cloud Run..."
gcloud run deploy $SERVICE_NAME \
  --image gcr.io/$PROJECT_ID/$SERVICE_NAME:$IMAGE_TAG \
  --platform managed \
  --region $LOCATION \
  --allow-unauthenticated \
  --port 8080 \
  --memory 512Mi \
  --cpu 1000m \
  --min-instances 0 \
  --max-instances 100 \
  --timeout 300 \
  --concurrency 80 \
  --set-env-vars "NODE_ENV=production,LOG_LEVEL=info" \
  --labels "environment=production,version=$IMAGE_TAG"

echo "✅ Deployment completed successfully!"

# サービスURLの取得と表示
SERVICE_URL=$(gcloud run services describe $SERVICE_NAME --platform managed --region $LOCATION --format 'value(status.url)')
echo "🌍 Service URL: $SERVICE_URL"
```

## まとめ

Google Cloud Run × TypeScript の組み合わせで、サーバー管理の複雑さから完全に解放された効率的なサーバーレスアプリケーション運用が実現できました。

**✅ この記事で身についたスキル**

>* Cloud Run サービスの作成・管理とライフサイクル制御
>* TypeScript + Cloud Run API での実用的なデプロイメント自動化
>* トラフィック制御とリビジョン管理の高度な運用パターン
>* 監視・ログ管理・コスト最適化の実装テクニック
>* Infrastructure as Code による再現可能なサーバーレス環境構築

**🎯 次のステップ**

>* Cloud Functions との使い分けとマイクロサービス連携
>* Cloud Build・Cloud Deploy を使った完全CI/CDパイプライン構築
>* Eventarc を活用したイベント駆動アーキテクチャの実装

Cloud Run なら、従来のサーバー運用の煩わしさを一切感じることなく、Google品質のフルマネージドサービスでスケーラブルなWebアプリケーションを運用できます。あなたも今日からプログラマブルなサーバーレス管理を始めてみませんか？