## もうサーバー管理で迷わない！Google Cloud Functions をTypeScriptで完全制御する実践ガイド

## はじめに

「たった数行の関数のためにサーバー全体を管理するのは面倒すぎる」
「イベント処理や軽量なAPI作成で、インフラ管理に時間を奪われたくない」

そんな悩みを抱える開発者の方に最適な解決策があります。Google Cloud Functions なら、**TypeScript** でサーバーレス関数を作成し、イベント駆動やHTTPトリガーで自動実行される環境を**数分で構築**できます。

この記事では、**Cloud Functions API** と **TypeScript** を組み合わせて関数のデプロイから監視まで全てをプログラマブルに制御し、実際のプロジェクトで使える実用的なコードパターンを解説します。記事を読み終える頃には、あなたもFunction as a Service の真の価値を実感できているはずです 🚀

>* Google Cloud Functions の基本概念とトリガー種類の理解
>* TypeScript + Cloud Functions API での関数管理自動化
>* HTTP・イベント・スケジュールトリガーの実装パターン
>* デプロイメント・監視・エラーハンドリングの運用手法

## Google Cloud Functions とは？サーバーレス関数の決定版

Google Cloud Functions は、Googleが提供する**フルマネージドなサーバーレス関数実行プラットフォーム**です。

**1. イベント駆動アーキテクチャ**

>* Cloud Storage・Pub/Sub・Firebase などのイベントに自動反応
>* HTTP リクエストやスケジュール実行にも対応
>* マイクロサービス間の疎結合な連携を実現

**2. ゼロ運用負荷**

>* サーバー管理・スケーリング・パッチ適用が完全自動
>* 実行時間のみの従量課金制（アイドル時間は無料）
>* 自動的な負荷分散とフォルトトレラント

**3. 豊富な実行環境とトリガー**

>* Node.js・Python・Go・Java など複数ランタイム対応
>* 1st gen・2nd gen の選択で最適なパフォーマンス
>* 豊富なGCPサービスとの統合

**⚖️ 他のサーバーレス関数サービスとの比較**

| サービス | ランタイム | 実行時間制限 | 同時実行数 | 主要トリガー |
|---------|-----------|-------------|-----------|------------|
| **Cloud Functions** | Node.js, Python, Go, Java | 540秒 | 3000 | HTTP, Pub/Sub, Storage |
| **AWS Lambda** | 多言語対応 | 900秒 | 1000 | API Gateway, S3, DynamoDB |
| **Azure Functions** | 多言語対応 | 無制限 | 200 | HTTP, Timer, Service Bus |

**📌 Cloud Functions を選ぶべきシーン**

>* GCPサービスとの連携を重視したイベント処理
>* 軽量なAPI作成やWebhook処理
>* バッチ処理やスケジュール実行
>* マイクロサービスアーキテクチャの一部として

## 開発環境のセットアップ

Cloud Functions をTypeScriptで管理するための開発環境を構築しましょう。

**📦 必要なツールのインストール**

```bash
# Google Cloud SDK のインストール（公式サイトからダウンロード）
# https://cloud.google.com/sdk/docs/install

# Cloud SDK の初期化とログイン
gcloud init
gcloud auth login
gcloud auth application-default login

# プロジェクトの設定
gcloud config set project your-functions-project

# 必要なAPIの有効化
gcloud services enable cloudfunctions.googleapis.com
gcloud services enable cloudbuild.googleapis.com
gcloud services enable eventarc.googleapis.com

# Functions Framework のインストール（ローカル開発用）
npm install -g @google-cloud/functions-framework
```

**🚀 TypeScript プロジェクトの初期化**

```bash
# プロジェクトディレクトリの作成
mkdir cloud-functions-typescript && cd cloud-functions-typescript

# Node.jsプロジェクトの初期化
npm init -y

# Google Cloud Functions 関連ライブラリ
npm install @google-cloud/functions-framework
npm install @google-cloud/storage @google-cloud/pubsub
npm install express cors

# TypeScript開発環境
npm install -D typescript @types/node @types/express
npm install -D ts-node nodemon @google-cloud/functions-framework

# ユーティリティライブラリ
npm install dotenv winston axios
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
    "outDir": "./lib",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "lib", "test"]
}
```

```json:package.json
{
  "name": "cloud-functions-typescript",
  "version": "1.0.0",
  "main": "lib/index.js",
  "scripts": {
    "build": "tsc",
    "dev": "nodemon --exec ts-node src/index.ts",
    "start": "functions-framework --target=main --source=lib",
    "deploy": "npm run build && gcloud functions deploy",
    "logs": "gcloud functions logs read --limit 100"
  },
  "engines": {
    "node": "18"
  }
}
```

## Cloud Functions 管理システムの実装

実用的なCloud Functions 管理システムをTypeScriptで構築してみましょう。

### 1\. 関数管理クラスの実装

```typescript:src/services/CloudFunctionsManager.ts
import { CloudFunctionsServiceClient } from '@google-cloud/functions';
import { logger } from '../utils/logger';

export interface FunctionConfig {
  name: string;
  location: string;
  runtime: 'nodejs18' | 'nodejs16' | 'python39' | 'python310' | 'go119';
  entryPoint: string;
  sourceArchiveUrl?: string;
  sourceRepository?: {
    url: string;
    branch: string;
  };
  httpsTrigger?: {
    securityLevel?: 'SECURE_ALWAYS' | 'SECURE_OPTIONAL';
  };
  eventTrigger?: {
    eventType: string;
    resource: string;
    service?: string;
  };
  environmentVariables?: Record<string, string>;
  availableMemoryMb?: number;
  timeout?: string;
  maxInstances?: number;
  minInstances?: number;
  vpcConnector?: string;
  labels?: Record<string, string>;
}

export interface FunctionInfo {
  name: string;
  location: string;
  runtime: string;
  status: string;
  trigger: string;
  url?: string;
  updateTime: string;
  versionId: string;
  sourceArchiveUrl: string;
}

export class CloudFunctionsManager {
  private client: CloudFunctionsServiceClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.client = new CloudFunctionsServiceClient();
  }

  async deployFunction(config: FunctionConfig): Promise<FunctionInfo> {
    const {
      name,
      location,
      runtime = 'nodejs18',
      entryPoint,
      sourceArchiveUrl,
      sourceRepository,
      httpsTrigger,
      eventTrigger,
      environmentVariables = {},
      availableMemoryMb = 256,
      timeout = '60s',
      maxInstances = 3000,
      minInstances = 0,
      vpcConnector,
      labels = {},
    } = config;

    const parent = `projects/${this.projectId}/locations/${location}`;
    const functionName = `${parent}/functions/${name}`;

    try {
      logger.info(`Deploying function: ${name} in location: ${location}`);

      // 関数の設定を構築
      const cloudFunction: any = {
        name: functionName,
        description: `TypeScript Cloud Function: ${name}`,
        runtime,
        entryPoint,
        environmentVariables,
        availableMemoryMb,
        timeout,
        maxInstances,
        minInstances,
        labels: {
          'deployment-tool': 'typescript-manager',
          ...labels,
        },
      };

      // ソースコードの設定
      if (sourceArchiveUrl) {
        cloudFunction.sourceArchiveUrl = sourceArchiveUrl;
      } else if (sourceRepository) {
        cloudFunction.sourceRepository = {
          url: sourceRepository.url,
          ...(sourceRepository.branch && { 
            sourcePath: `refs/heads/${sourceRepository.branch}` 
          }),
        };
      } else {
        throw new Error('Either sourceArchiveUrl or sourceRepository must be provided');
      }

      // トリガーの設定
      if (httpsTrigger) {
        cloudFunction.httpsTrigger = {
          securityLevel: httpsTrigger.securityLevel || 'SECURE_ALWAYS',
        };
      } else if (eventTrigger) {
        cloudFunction.eventTrigger = {
          eventType: eventTrigger.eventType,
          resource: eventTrigger.resource,
          ...(eventTrigger.service && { service: eventTrigger.service }),
        };
      } else {
        // デフォルトはHTTPS トリガー
        cloudFunction.httpsTrigger = {
          securityLevel: 'SECURE_ALWAYS',
        };
      }

      // VPC コネクタの設定
      if (vpcConnector) {
        cloudFunction.vpcConnector = vpcConnector;
      }

      const request = {
        parent,
        cloudFunction,
      };

      const [operation] = await this.client.createFunction(request);
      logger.info(`Function deployment operation started: ${operation.name}`);

      // オペレーション完了を待つ
      await this.waitForOperation(operation.name!);

      // デプロイ後の関数情報を取得
      const functionInfo = await this.getFunction(name, location);

      logger.info(`Successfully deployed function: ${name}`);
      return functionInfo;

    } catch (error) {
      logger.error(`Failed to deploy function ${name}:`, error);
      throw new Error(`Function deployment failed: ${error.message}`);
    }
  }

  async getFunction(name: string, location: string): Promise<FunctionInfo> {
    try {
      const functionName = `projects/${this.projectId}/locations/${location}/functions/${name}`;

      const [cloudFunction] = await this.client.getFunction({
        name: functionName,
      });

      return {
        name: this.extractFunctionNameFromPath(cloudFunction.name || ''),
        location: this.extractLocationFromPath(cloudFunction.name || ''),
        runtime: cloudFunction.runtime || '',
        status: this.getFunctionStatus(cloudFunction),
        trigger: this.getTriggerType(cloudFunction),
        url: cloudFunction.httpsTrigger?.url || undefined,
        updateTime: cloudFunction.updateTime?.toDateString() || '',
        versionId: cloudFunction.versionId?.toString() || '',
        sourceArchiveUrl: cloudFunction.sourceArchiveUrl || '',
      };

    } catch (error) {
      logger.error(`Failed to get function ${name}:`, error);
      throw new Error(`Function retrieval failed: ${error.message}`);
    }
  }

  async listFunctions(location: string): Promise<FunctionInfo[]> {
    try {
      const parent = `projects/${this.projectId}/locations/${location}`;

      const [functions] = await this.client.listFunctions({ parent });

      return functions.map(func => ({
        name: this.extractFunctionNameFromPath(func.name || ''),
        location: this.extractLocationFromPath(func.name || ''),
        runtime: func.runtime || '',
        status: this.getFunctionStatus(func),
        trigger: this.getTriggerType(func),
        url: func.httpsTrigger?.url || undefined,
        updateTime: func.updateTime?.toDateString() || '',
        versionId: func.versionId?.toString() || '',
        sourceArchiveUrl: func.sourceArchiveUrl || '',
      }));

    } catch (error) {
      logger.error(`Failed to list functions in location ${location}:`, error);
      throw new Error(`Function listing failed: ${error.message}`);
    }
  }

  async updateFunction(name: string, location: string, config: Partial<FunctionConfig>): Promise<FunctionInfo> {
    try {
      const functionName = `projects/${this.projectId}/locations/${location}/functions/${name}`;

      logger.info(`Updating function: ${name}`);

      // 現在の関数設定を取得
      const [currentFunction] = await this.client.getFunction({
        name: functionName,
      });

      // 更新する項目をマージ
      const updatedFunction = {
        ...currentFunction,
        ...(config.environmentVariables && { 
          environmentVariables: config.environmentVariables 
        }),
        ...(config.availableMemoryMb && { 
          availableMemoryMb: config.availableMemoryMb 
        }),
        ...(config.timeout && { timeout: config.timeout }),
        ...(config.maxInstances && { maxInstances: config.maxInstances }),
        ...(config.minInstances && { minInstances: config.minInstances }),
        ...(config.labels && { 
          labels: { ...currentFunction.labels, ...config.labels } 
        }),
      };

      const [operation] = await this.client.updateFunction({
        cloudFunction: updatedFunction,
      });

      await this.waitForOperation(operation.name!);

      const functionInfo = await this.getFunction(name, location);
      logger.info(`Successfully updated function: ${name}`);

      return functionInfo;

    } catch (error) {
      logger.error(`Failed to update function ${name}:`, error);
      throw new Error(`Function update failed: ${error.message}`);
    }
  }

  async deleteFunction(name: string, location: string): Promise<void> {
    try {
      const functionName = `projects/${this.projectId}/locations/${location}/functions/${name}`;

      logger.info(`Deleting function: ${name}`);

      const [operation] = await this.client.deleteFunction({
        name: functionName,
      });

      await this.waitForOperation(operation.name!);

      logger.info(`Successfully deleted function: ${name}`);

    } catch (error) {
      logger.error(`Failed to delete function ${name}:`, error);
      throw new Error(`Function deletion failed: ${error.message}`);
    }
  }

  async callFunction(name: string, location: string, data: any): Promise<any> {
    try {
      const functionName = `projects/${this.projectId}/locations/${location}/functions/${name}`;

      logger.info(`Calling function: ${name}`);

      const [result] = await this.client.callFunction({
        name: functionName,
        data: JSON.stringify(data),
      });

      logger.info(`Function ${name} executed successfully`);
      return JSON.parse(result.result || '{}');

    } catch (error) {
      logger.error(`Failed to call function ${name}:`, error);
      throw new Error(`Function call failed: ${error.message}`);
    }
  }

  private getFunctionStatus(func: any): string {
    switch (func.status) {
      case 'CLOUD_FUNCTION_STATUS_UNSPECIFIED':
        return 'Unspecified';
      case 'ACTIVE':
        return 'Active';
      case 'OFFLINE':
        return 'Offline';
      case 'DEPLOY_IN_PROGRESS':
        return 'Deploying';
      case 'DELETE_IN_PROGRESS':
        return 'Deleting';
      case 'UNKNOWN':
        return 'Unknown';
      default:
        return 'Unknown';
    }
  }

  private getTriggerType(func: any): string {
    if (func.httpsTrigger) {
      return 'HTTPS';
    } else if (func.eventTrigger) {
      return func.eventTrigger.eventType || 'Event';
    }
    return 'Unknown';
  }

  private extractFunctionNameFromPath(fullName: string): string {
    const parts = fullName.split('/');
    return parts[parts.length - 1] || '';
  }

  private extractLocationFromPath(fullName: string): string {
    const match = fullName.match(/\/locations\/([^/]+)\//);
    return match ? match[1] : '';
  }

  private async waitForOperation(operationName: string): Promise<void> {
    logger.info(`Waiting for operation: ${operationName}`);
    
    // 実際の実装では適切なオペレーション完了待ちを実装
    await new Promise(resolve => setTimeout(resolve, 15000)); // 15秒の待機
  }
}
```

### 2\. 実用的な関数実装例

```typescript:src/functions/httpFunctions.ts
import { HttpFunction } from '@google-cloud/functions-framework';
import { Request, Response } from 'express';
import { logger } from '../utils/logger';

// シンプルなHelloWorld API
export const helloWorld: HttpFunction = (req: Request, res: Response) => {
  const name = req.query.name || req.body.name || 'World';
  
  logger.info(`Hello function called with name: ${name}`);
  
  res.set('Access-Control-Allow-Origin', '*');
  res.set('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  res.set('Access-Control-Allow-Headers', 'Content-Type');
  
  if (req.method === 'OPTIONS') {
    res.status(204).send('');
    return;
  }

  const response = {
    message: `Hello, ${name}!`,
    timestamp: new Date().toISOString(),
    method: req.method,
    headers: req.headers,
  };

  res.status(200).json(response);
};

// データ処理API
export const processData: HttpFunction = async (req: Request, res: Response) => {
  try {
    if (req.method !== 'POST') {
      res.status(405).json({ error: 'Method not allowed' });
      return;
    }

    const { data, operation } = req.body;

    if (!data) {
      res.status(400).json({ error: 'Data is required' });
      return;
    }

    logger.info(`Processing data with operation: ${operation}`);

    let result;
    switch (operation) {
      case 'uppercase':
        result = typeof data === 'string' ? data.toUpperCase() : data;
        break;
      case 'reverse':
        result = typeof data === 'string' ? data.split('').reverse().join('') : data;
        break;
      case 'length':
        result = typeof data === 'string' ? data.length : JSON.stringify(data).length;
        break;
      default:
        result = data;
    }

    const response = {
      success: true,
      operation,
      originalData: data,
      result,
      processedAt: new Date().toISOString(),
    };

    res.status(200).json(response);

  } catch (error) {
    logger.error('Error in processData function:', error);
    res.status(500).json({
      error: 'Internal server error',
      message: error.message,
    });
  }
};

// ファイルアップロード処理API
export const uploadFile: HttpFunction = async (req: Request, res: Response) => {
  try {
    if (req.method !== 'POST') {
      res.status(405).json({ error: 'Method not allowed' });
      return;
    }

    const { fileName, fileContent, contentType } = req.body;

    if (!fileName || !fileContent) {
      res.status(400).json({ error: 'fileName and fileContent are required' });
      return;
    }

    logger.info(`Processing file upload: ${fileName}`);

    // 実際の実装では Cloud Storage にファイルをアップロード
    const fileSize = Buffer.byteLength(fileContent, 'utf8');
    const fileId = `${Date.now()}-${fileName}`;

    // ここで実際のファイル処理を行う
    await processFileUpload(fileId, fileContent, contentType);

    const response = {
      success: true,
      fileId,
      fileName,
      fileSize,
      contentType,
      uploadedAt: new Date().toISOString(),
    };

    res.status(200).json(response);

  } catch (error) {
    logger.error('Error in uploadFile function:', error);
    res.status(500).json({
      error: 'File upload failed',
      message: error.message,
    });
  }
};

async function processFileUpload(fileId: string, content: string, contentType: string): Promise<void> {
  // ファイルアップロード処理のシミュレーション
  logger.info(`Processing file upload: ${fileId}, contentType: ${contentType}`);
  
  // 実際の実装では:
  // - Cloud Storage へのアップロード
  // - ファイル形式の検証
  // - ウイルススキャン
  // - サムネイル生成等
  
  await new Promise(resolve => setTimeout(resolve, 1000));
}
```

### 3\. イベント駆動関数の実装

```typescript:src/functions/eventFunctions.ts
import { CloudEvent } from '@google-cloud/functions-framework';
import { logger } from '../utils/logger';

// Cloud Storage イベント処理
export const processStorageEvent = (cloudEvent: CloudEvent) => {
  logger.info('Storage event received:', {
    id: cloudEvent.id,
    type: cloudEvent.type,
    source: cloudEvent.source,
  });

  const data = cloudEvent.data;
  const fileName = data?.name;
  const bucketName = data?.bucket;
  const eventType = cloudEvent.type;

  logger.info(`Processing storage event: ${eventType} for file: ${fileName} in bucket: ${bucketName}`);

  switch (eventType) {
    case 'google.cloud.storage.object.v1.finalized':
      return handleFileCreated(fileName, bucketName, data);
    case 'google.cloud.storage.object.v1.deleted':
      return handleFileDeleted(fileName, bucketName, data);
    case 'google.cloud.storage.object.v1.updated':
      return handleFileUpdated(fileName, bucketName, data);
    default:
      logger.warn(`Unhandled storage event type: ${eventType}`);
  }
};

// Pub/Sub メッセージ処理
export const processPubSubMessage = (cloudEvent: CloudEvent) => {
  logger.info('Pub/Sub event received:', {
    id: cloudEvent.id,
    type: cloudEvent.type,
  });

  const message = cloudEvent.data?.message;
  if (!message) {
    logger.error('No message found in Pub/Sub event');
    return;
  }

  // Base64デコード
  const messageData = message.data ? 
    Buffer.from(message.data, 'base64').toString() : 
    '{}';

  const attributes = message.attributes || {};

  logger.info('Processing Pub/Sub message:', {
    messageId: message.messageId,
    publishTime: message.publishTime,
    attributes,
  });

  try {
    const parsedData = JSON.parse(messageData);
    return handlePubSubMessage(parsedData, attributes);
  } catch (error) {
    logger.error('Failed to parse Pub/Sub message data:', error);
    return handleRawPubSubMessage(messageData, attributes);
  }
};

// Firebase イベント処理
export const processFirestoreEvent = (cloudEvent: CloudEvent) => {
  logger.info('Firestore event received:', {
    id: cloudEvent.id,
    type: cloudEvent.type,
    source: cloudEvent.source,
  });

  const data = cloudEvent.data;
  const eventType = cloudEvent.type;
  
  logger.info(`Processing Firestore event: ${eventType}`);

  switch (eventType) {
    case 'google.cloud.firestore.document.v1.created':
      return handleDocumentCreated(data);
    case 'google.cloud.firestore.document.v1.updated':
      return handleDocumentUpdated(data);
    case 'google.cloud.firestore.document.v1.deleted':
      return handleDocumentDeleted(data);
    default:
      logger.warn(`Unhandled Firestore event type: ${eventType}`);
  }
};

// スケジュール実行（Cron）関数
export const scheduledFunction = (cloudEvent: CloudEvent) => {
  logger.info('Scheduled function triggered:', {
    id: cloudEvent.id,
    time: cloudEvent.time,
  });

  return performScheduledTask();
};

// イベントハンドラーの実装

async function handleFileCreated(fileName: string, bucketName: string, data: any): Promise<void> {
  logger.info(`File created: ${fileName} in bucket: ${bucketName}`);
  
  // ファイル作成時の処理例:
  // - 画像のサムネイル生成
  // - データベースへのメタデータ保存  
  // - 通知の送信
  
  if (fileName.match(/\.(jpg|jpeg|png)$/i)) {
    await generateThumbnail(bucketName, fileName);
  }
  
  if (fileName.match(/\.(csv|json)$/i)) {
    await processDataFile(bucketName, fileName);
  }
}

async function handleFileDeleted(fileName: string, bucketName: string, data: any): Promise<void> {
  logger.info(`File deleted: ${fileName} from bucket: ${bucketName}`);
  
  // ファイル削除時の処理例:
  // - 関連データのクリーンアップ
  // - キャッシュの無効化
  // - 監査ログの記録
}

async function handleFileUpdated(fileName: string, bucketName: string, data: any): Promise<void> {
  logger.info(`File updated: ${fileName} in bucket: ${bucketName}`);
  
  // ファイル更新時の処理例:
  // - バージョン管理
  // - 変更通知
  // - 再処理の実行
}

async function handlePubSubMessage(data: any, attributes: Record<string, string>): Promise<void> {
  logger.info('Processing structured Pub/Sub message:', data);
  
  // 構造化されたメッセージの処理
  const messageType = attributes.messageType || 'unknown';
  
  switch (messageType) {
    case 'user_signup':
      await processUserSignup(data);
      break;
    case 'order_created':
      await processOrderCreated(data);
      break;
    case 'payment_completed':
      await processPaymentCompleted(data);
      break;
    default:
      logger.info(`Processing generic message of type: ${messageType}`);
  }
}

async function handleRawPubSubMessage(rawData: string, attributes: Record<string, string>): Promise<void> {
  logger.info('Processing raw Pub/Sub message:', rawData);
  
  // 生のメッセージデータの処理
}

async function handleDocumentCreated(data: any): Promise<void> {
  logger.info('Firestore document created:', data);
  
  // ドキュメント作成時の処理例:
  // - 検索インデックスの更新
  // - キャッシュの更新
  // - 通知の送信
}

async function handleDocumentUpdated(data: any): Promise<void> {
  logger.info('Firestore document updated:', data);
  
  // ドキュメント更新時の処理例:
  // - 変更履歴の記録
  // - 関連データの同期
  // - キャッシュの無効化
}

async function handleDocumentDeleted(data: any): Promise<void> {
  logger.info('Firestore document deleted:', data);
  
  // ドキュメント削除時の処理例:
  // - 関連データのクリーンアップ
  // - インデックスの更新
  // - バックアップの作成
}

async function performScheduledTask(): Promise<void> {
  logger.info('Performing scheduled task');
  
  // スケジュール実行の処理例:
  // - データベースのクリーンアップ
  // - レポート生成
  // - バックアップ作成
  // - システムヘルスチェック
  
  const startTime = Date.now();
  
  try {
    // タスクの実行
    await runDatabaseCleanup();
    await generateDailyReport();
    await performHealthCheck();
    
    const executionTime = Date.now() - startTime;
    logger.info(`Scheduled task completed in ${executionTime}ms`);
    
  } catch (error) {
    logger.error('Scheduled task failed:', error);
    throw error;
  }
}

// ヘルパー関数
async function generateThumbnail(bucketName: string, fileName: string): Promise<void> {
  logger.info(`Generating thumbnail for ${fileName}`);
  // サムネイル生成処理
}

async function processDataFile(bucketName: string, fileName: string): Promise<void> {
  logger.info(`Processing data file ${fileName}`);
  // データファイル処理
}

async function processUserSignup(data: any): Promise<void> {
  logger.info('Processing user signup:', data);
  // ユーザー登録処理
}

async function processOrderCreated(data: any): Promise<void> {
  logger.info('Processing order created:', data);
  // 注文作成処理
}

async function processPaymentCompleted(data: any): Promise<void> {
  logger.info('Processing payment completed:', data);
  // 決済完了処理
}

async function runDatabaseCleanup(): Promise<void> {
  logger.info('Running database cleanup');
  // データベースクリーンアップ
}

async function generateDailyReport(): Promise<void> {
  logger.info('Generating daily report');
  // 日次レポート生成
}

async function performHealthCheck(): Promise<void> {
  logger.info('Performing health check');
  // ヘルスチェック実行
}
```

### 4\. 統合デプロイメントスクリプト

```typescript:src/deployment/deploymentManager.ts
import { CloudFunctionsManager, FunctionConfig } from '../services/CloudFunctionsManager';
import { logger } from '../utils/logger';

export interface DeploymentPlan {
  projectId: string;
  location: string;
  functions: FunctionConfig[];
  sourceArchiveUrl?: string;
}

export class FunctionDeploymentManager {
  private functionsManager: CloudFunctionsManager;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.functionsManager = new CloudFunctionsManager(projectId);
  }

  async deployAll(plan: DeploymentPlan): Promise<void> {
    const { location, functions, sourceArchiveUrl } = plan;

    logger.info(`Starting deployment of ${functions.length} functions`);

    try {
      // 並列デプロイメント（制限付き）
      const concurrency = 3; // 同時デプロイ数の制限
      const batches = this.createBatches(functions, concurrency);

      for (const batch of batches) {
        const deployPromises = batch.map(async (funcConfig) => {
          try {
            const config = {
              ...funcConfig,
              location,
              sourceArchiveUrl: sourceArchiveUrl || funcConfig.sourceArchiveUrl,
            };

            const result = await this.functionsManager.deployFunction(config);
            logger.info(`Successfully deployed: ${funcConfig.name}`);
            return result;
          } catch (error) {
            logger.error(`Failed to deploy ${funcConfig.name}:`, error);
            throw error;
          }
        });

        await Promise.allSettled(deployPromises);
      }

      logger.info('All functions deployment completed');

    } catch (error) {
      logger.error('Deployment failed:', error);
      throw error;
    }
  }

  async deployProductionSetup(): Promise<void> {
    const location = 'asia-northeast1';
    
    const functions: FunctionConfig[] = [
      // HTTP トリガー関数
      {
        name: 'api-hello-world',
        location,
        runtime: 'nodejs18',
        entryPoint: 'helloWorld',
        sourceArchiveUrl: `gs://${this.projectId}-functions-source/api-functions.zip`,
        httpsTrigger: { securityLevel: 'SECURE_ALWAYS' },
        availableMemoryMb: 256,
        timeout: '60s',
        maxInstances: 100,
        minInstances: 0,
        environmentVariables: {
          NODE_ENV: 'production',
          LOG_LEVEL: 'info',
        },
        labels: {
          environment: 'production',
          function_type: 'api',
        },
      },
      
      {
        name: 'api-process-data',
        location,
        runtime: 'nodejs18',
        entryPoint: 'processData',
        sourceArchiveUrl: `gs://${this.projectId}-functions-source/api-functions.zip`,
        httpsTrigger: { securityLevel: 'SECURE_ALWAYS' },
        availableMemoryMb: 512,
        timeout: '300s',
        maxInstances: 50,
        minInstances: 1,
        environmentVariables: {
          NODE_ENV: 'production',
          LOG_LEVEL: 'info',
          MAX_PROCESSING_SIZE: '10MB',
        },
        labels: {
          environment: 'production',
          function_type: 'processing',
        },
      },

      // Storage イベント関数
      {
        name: 'storage-processor',
        location,
        runtime: 'nodejs18',
        entryPoint: 'processStorageEvent',
        sourceArchiveUrl: `gs://${this.projectId}-functions-source/event-functions.zip`,
        eventTrigger: {
          eventType: 'google.cloud.storage.object.v1.finalized',
          resource: `projects/_/buckets/${this.projectId}-uploads`,
        },
        availableMemoryMb: 1024,
        timeout: '540s',
        maxInstances: 10,
        minInstances: 0,
        environmentVariables: {
          NODE_ENV: 'production',
          THUMBNAIL_BUCKET: `${this.projectId}-thumbnails`,
          PROCESSED_BUCKET: `${this.projectId}-processed`,
        },
        labels: {
          environment: 'production',
          function_type: 'event_driven',
        },
      },

      // Pub/Sub イベント関数
      {
        name: 'pubsub-processor',
        location,
        runtime: 'nodejs18',
        entryPoint: 'processPubSubMessage',
        sourceArchiveUrl: `gs://${this.projectId}-functions-source/event-functions.zip`,
        eventTrigger: {
          eventType: 'google.cloud.pubsub.topic.v1.messagePublished',
          resource: `projects/${this.projectId}/topics/notifications`,
        },
        availableMemoryMb: 512,
        timeout: '300s',
        maxInstances: 20,
        minInstances: 0,
        environmentVariables: {
          NODE_ENV: 'production',
          NOTIFICATION_SERVICE_URL: process.env.NOTIFICATION_SERVICE_URL || '',
        },
        labels: {
          environment: 'production',
          function_type: 'message_processing',
        },
      },

      // スケジュール実行関数
      {
        name: 'scheduled-cleanup',
        location,
        runtime: 'nodejs18',
        entryPoint: 'scheduledFunction',
        sourceArchiveUrl: `gs://${this.projectId}-functions-source/scheduled-functions.zip`,
        eventTrigger: {
          eventType: 'google.cloud.scheduler.job.v1.executed',
          resource: `projects/${this.projectId}/locations/${location}/jobs/daily-cleanup`,
        },
        availableMemoryMb: 1024,
        timeout: '540s',
        maxInstances: 1,
        minInstances: 0,
        environmentVariables: {
          NODE_ENV: 'production',
          CLEANUP_DAYS: '30',
          BACKUP_BUCKET: `${this.projectId}-backups`,
        },
        labels: {
          environment: 'production',
          function_type: 'scheduled',
        },
      },
    ];

    const deploymentPlan: DeploymentPlan = {
      projectId: this.projectId,
      location,
      functions,
    };

    await this.deployAll(deploymentPlan);
  }

  async deployDevelopmentSetup(): Promise<void> {
    const location = 'asia-northeast1';
    
    const functions: FunctionConfig[] = [
      {
        name: 'dev-api-hello',
        location,
        runtime: 'nodejs18',
        entryPoint: 'helloWorld',
        sourceArchiveUrl: `gs://${this.projectId}-functions-source/dev-api-functions.zip`,
        httpsTrigger: { securityLevel: 'SECURE_OPTIONAL' },
        availableMemoryMb: 256,
        timeout: '60s',
        maxInstances: 10,
        minInstances: 0,
        environmentVariables: {
          NODE_ENV: 'development',
          LOG_LEVEL: 'debug',
        },
        labels: {
          environment: 'development',
          function_type: 'api',
        },
      },
    ];

    const deploymentPlan: DeploymentPlan = {
      projectId: this.projectId,
      location,
      functions,
    };

    await this.deployAll(deploymentPlan);
  }

  private createBatches<T>(items: T[], batchSize: number): T[][] {
    const batches: T[][] = [];
    for (let i = 0; i < items.length; i += batchSize) {
      batches.push(items.slice(i, i + batchSize));
    }
    return batches;
  }

  async listAllFunctions(location: string): Promise<void> {
    try {
      const functions = await this.functionsManager.listFunctions(location);
      
      logger.info(`Found ${functions.length} functions in location: ${location}`);
      
      functions.forEach(func => {
        logger.info(`- ${func.name}: ${func.status} (${func.trigger}) - ${func.url || 'No URL'}`);
      });

    } catch (error) {
      logger.error('Failed to list functions:', error);
    }
  }
}

// 実行例
if (require.main === module) {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const manager = new FunctionDeploymentManager(projectId);
  const environment = process.argv[2] || 'production';

  async function main() {
    try {
      if (environment === 'production') {
        await manager.deployProductionSetup();
      } else {
        await manager.deployDevelopmentSetup();
      }

      // デプロイ後の確認
      await manager.listAllFunctions('asia-northeast1');

    } catch (error) {
      logger.error('Deployment script failed:', error);
      process.exit(1);
    }
  }

  main();
}
```

## 監視とログ管理

```typescript:src/services/FunctionMonitoringService.ts
import { LoggingServiceClient } from '@google-cloud/logging';
import { MetricServiceClient } from '@google-cloud/monitoring';
import { logger } from '../utils/logger';

export interface FunctionMetrics {
  functionName: string;
  location: string;
  executions: number;
  errors: number;
  errorRate: number;
  averageDuration: number;
  memoryUsage: number;
  timestamp: Date;
}

export interface LogEntry {
  timestamp: Date;
  severity: string;
  message: string;
  functionName: string;
  executionId: string;
  labels: Record<string, string>;
}

export class FunctionMonitoringService {
  private loggingClient: LoggingServiceClient;
  private metricsClient: MetricServiceClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.loggingClient = new LoggingServiceClient();
    this.metricsClient = new MetricServiceClient();
  }

  async getFunctionMetrics(
    functionName: string, 
    location: string, 
    hours = 1
  ): Promise<FunctionMetrics[]> {
    const now = new Date();
    const startTime = new Date(now.getTime() - hours * 60 * 60 * 1000);

    try {
      const projectPath = this.metricsClient.projectPath(this.projectId);

      // 実行回数の取得
      const executionsFilter = `
        resource.type="cloud_function" AND
        resource.labels.function_name="${functionName}" AND
        resource.labels.region="${location}" AND
        metric.type="cloudfunctions.googleapis.com/function/executions"
      `;

      const [executionMetrics] = await this.metricsClient.listTimeSeries({
        name: projectPath,
        filter: executionsFilter,
        interval: {
          endTime: { seconds: Math.floor(now.getTime() / 1000) },
          startTime: { seconds: Math.floor(startTime.getTime() / 1000) },
        },
      });

      // エラー数の取得
      const errorsFilter = `
        resource.type="cloud_function" AND
        resource.labels.function_name="${functionName}" AND
        resource.labels.region="${location}" AND
        metric.type="cloudfunctions.googleapis.com/function/executions" AND
        metric.labels.status!="ok"
      `;

      const [errorMetrics] = await this.metricsClient.listTimeSeries({
        name: projectPath,
        filter: errorsFilter,
        interval: {
          endTime: { seconds: Math.floor(now.getTime() / 1000) },
          startTime: { seconds: Math.floor(startTime.getTime() / 1000) },
        },
      });

      // 実行時間の取得
      const durationFilter = `
        resource.type="cloud_function" AND
        resource.labels.function_name="${functionName}" AND
        resource.labels.region="${location}" AND
        metric.type="cloudfunctions.googleapis.com/function/execution_times"
      `;

      const [durationMetrics] = await this.metricsClient.listTimeSeries({
        name: projectPath,
        filter: durationFilter,
        interval: {
          endTime: { seconds: Math.floor(now.getTime() / 1000) },
          startTime: { seconds: Math.floor(startTime.getTime() / 1000) },
        },
      });

      // メトリクスを統合
      const metrics: FunctionMetrics[] = [];
      
      for (let i = 0; i < executionMetrics.length; i++) {
        const executions = executionMetrics[i]?.points?.[0]?.value?.int64Value || 0;
        const errors = errorMetrics[i]?.points?.[0]?.value?.int64Value || 0;
        const duration = durationMetrics[i]?.points?.[0]?.value?.doubleValue || 0;

        metrics.push({
          functionName,
          location,
          executions: Number(executions),
          errors: Number(errors),
          errorRate: executions > 0 ? (Number(errors) / Number(executions)) * 100 : 0,
          averageDuration: duration * 1000, // ミリ秒に変換
          memoryUsage: 0, // 実際の実装では適切に取得
          timestamp: new Date(executionMetrics[i]?.points?.[0]?.interval?.endTime?.seconds! * 1000),
        });
      }

      return metrics;

    } catch (error) {
      logger.error(`Failed to get metrics for function ${functionName}:`, error);
      throw new Error(`Metrics retrieval failed: ${error.message}`);
    }
  }

  async getFunctionLogs(
    functionName: string, 
    hours = 1, 
    severity?: string
  ): Promise<LogEntry[]> {
    try {
      const now = new Date();
      const startTime = new Date(now.getTime() - hours * 60 * 60 * 1000);

      let filter = `
        resource.type="cloud_function" AND
        resource.labels.function_name="${functionName}" AND
        timestamp>="${startTime.toISOString()}" AND
        timestamp<="${now.toISOString()}"
      `;

      if (severity) {
        filter += ` AND severity>="${severity}"`;
      }

      const [entries] = await this.loggingClient.listLogEntries({
        resourceNames: [`projects/${this.projectId}`],
        filter,
        orderBy: 'timestamp desc',
        pageSize: 1000,
      });

      return entries.map(entry => ({
        timestamp: new Date(entry.timestamp?.seconds! * 1000),
        severity: entry.severity || 'DEFAULT',
        message: entry.textPayload || entry.jsonPayload?.message || 'No message',
        functionName,
        executionId: entry.labels?.execution_id || '',
        labels: entry.labels || {},
      }));

    } catch (error) {
      logger.error(`Failed to get logs for function ${functionName}:`, error);
      throw new Error(`Log retrieval failed: ${error.message}`);
    }
  }

  async createFunctionAlert(functionName: string, location: string): Promise<void> {
    try {
      const alertPolicy = {
        displayName: `Cloud Function ${functionName} Error Rate Alert`,
        conditions: [
          {
            displayName: 'High error rate condition',
            conditionThreshold: {
              filter: `
                resource.type="cloud_function" AND
                resource.labels.function_name="${functionName}" AND
                resource.labels.region="${location}" AND
                metric.type="cloudfunctions.googleapis.com/function/executions" AND
                metric.labels.status!="ok"
              `,
              comparison: 'COMPARISON_GREATER_THAN',
              thresholdValue: 0.1, // 10% エラー率
              duration: { seconds: 300 },
              aggregations: [
                {
                  alignmentPeriod: { seconds: 60 },
                  perSeriesAligner: 'ALIGN_RATE',
                  crossSeriesReducer: 'REDUCE_SUM',
                },
              ],
            },
          },
        ],
        enabled: true,
      };

      logger.info(`Creating alert policy for function: ${functionName}`);
      // 実際の実装では AlertPolicyServiceClient を使用してアラートを作成

    } catch (error) {
      logger.error('Failed to create alert policy:', error);
      throw new Error(`Alert policy creation failed: ${error.message}`);
    }
  }

  async generateFunctionReport(location: string): Promise<any> {
    try {
      // 全関数の一覧を取得（実際の実装では CloudFunctionsManager を使用）
      const functions = ['api-hello-world', 'api-process-data', 'storage-processor'];
      
      const report = {
        location,
        generatedAt: new Date().toISOString(),
        totalFunctions: functions.length,
        functionsMetrics: [] as any[],
        summary: {
          totalExecutions: 0,
          totalErrors: 0,
          averageErrorRate: 0,
          averageDuration: 0,
        },
      };

      for (const functionName of functions) {
        try {
          const metrics = await this.getFunctionMetrics(functionName, location, 24);
          const recentLogs = await this.getFunctionLogs(functionName, 1, 'ERROR');

          const functionReport = {
            name: functionName,
            metrics: metrics.length > 0 ? metrics[0] : null,
            recentErrors: recentLogs.length,
            status: recentLogs.length > 10 ? 'UNHEALTHY' : 'HEALTHY',
          };

          report.functionsMetrics.push(functionReport);

          if (metrics.length > 0) {
            report.summary.totalExecutions += metrics[0].executions;
            report.summary.totalErrors += metrics[0].errors;
          }
        } catch (error) {
          logger.warn(`Failed to get metrics for function ${functionName}:`, error);
        }
      }

      report.summary.averageErrorRate = 
        report.summary.totalExecutions > 0 ? 
        (report.summary.totalErrors / report.summary.totalExecutions) * 100 : 0;

      report.summary.averageDuration = 
        report.functionsMetrics.reduce((sum, func) => 
          sum + (func.metrics?.averageDuration || 0), 0) / report.functionsMetrics.length;

      logger.info('Function monitoring report generated:', {
        totalFunctions: report.totalFunctions,
        totalExecutions: report.summary.totalExecutions,
        totalErrors: report.summary.totalErrors,
        averageErrorRate: report.summary.averageErrorRate,
      });

      return report;

    } catch (error) {
      logger.error('Failed to generate function report:', error);
      throw new Error(`Function report generation failed: ${error.message}`);
    }
  }
}
```

## ローカル開発とテスト

**🛠️ ローカル開発環境**

```typescript:src/utils/localDevelopment.ts
import { Request, Response } from 'express';
import express from 'express';
import cors from 'cors';
import { logger } from './logger';

// ローカル開発用サーバー
export class LocalFunctionServer {
  private app: express.Application;
  private port: number;

  constructor(port = 8080) {
    this.app = express();
    this.port = port;
    this.setupMiddleware();
  }

  private setupMiddleware(): void {
    this.app.use(cors());
    this.app.use(express.json());
    this.app.use(express.urlencoded({ extended: true }));
    
    // ログミドルウェア
    this.app.use((req: Request, res: Response, next) => {
      logger.info(`${req.method} ${req.path}`, {
        headers: req.headers,
        body: req.body,
        query: req.query,
      });
      next();
    });
  }

  registerFunction(path: string, handler: Function): void {
    this.app.all(path, (req: Request, res: Response) => {
      try {
        // Cloud Functions の環境変数をシミュレート
        process.env.K_SERVICE = 'local-function';
        process.env.K_REVISION = 'local-revision-001';
        
        handler(req, res);
      } catch (error) {
        logger.error('Function execution error:', error);
        res.status(500).json({
          error: 'Internal server error',
          message: error.message,
        });
      }
    });
  }

  registerEventFunction(path: string, handler: Function): void {
    this.app.post(path, (req: Request, res: Response) => {
      try {
        // Cloud Event のシミュレート
        const cloudEvent = {
          id: `local-event-${Date.now()}`,
          source: 'local-development',
          type: req.headers['ce-type'] || 'local.test.event',
          time: new Date().toISOString(),
          data: req.body.data || req.body,
        };

        handler(cloudEvent);
        res.status(200).json({ success: true, eventId: cloudEvent.id });
      } catch (error) {
        logger.error('Event function execution error:', error);
        res.status(500).json({
          error: 'Internal server error',
          message: error.message,
        });
      }
    });
  }

  start(): void {
    this.app.listen(this.port, () => {
      logger.info(`Local Functions server is running on http://localhost:${this.port}`);
    });
  }
}

// テスト用のモック関数
export function createMockCloudEvent(type: string, data: any): any {
  return {
    id: `mock-event-${Date.now()}`,
    source: 'mock-testing',
    type,
    time: new Date().toISOString(),
    data,
  };
}

export function createMockPubSubEvent(messageData: any, attributes: Record<string, string> = {}): any {
  return {
    id: `mock-pubsub-${Date.now()}`,
    source: 'mock-pubsub',
    type: 'google.cloud.pubsub.topic.v1.messagePublished',
    time: new Date().toISOString(),
    data: {
      message: {
        messageId: `mock-message-${Date.now()}`,
        publishTime: new Date().toISOString(),
        data: Buffer.from(JSON.stringify(messageData)).toString('base64'),
        attributes,
      },
    },
  };
}
```

**📋 統合テストスイート**

```typescript:src/test/integrationTests.ts
import axios from 'axios';
import { LocalFunctionServer } from '../utils/localDevelopment';
import { logger } from '../utils/logger';

// HTTP関数の統合テスト
export class FunctionIntegrationTest {
  private server: LocalFunctionServer;
  private baseUrl: string;

  constructor(port = 8080) {
    this.server = new LocalFunctionServer(port);
    this.baseUrl = `http://localhost:${port}`;
  }

  async testHttpFunctions(): Promise<void> {
    logger.info('Starting HTTP function integration tests');

    // Hello World 関数のテスト
    await this.testHelloWorldFunction();
    
    // Data Processing 関数のテスト
    await this.testDataProcessingFunction();
    
    // File Upload 関数のテスト
    await this.testFileUploadFunction();

    logger.info('HTTP function integration tests completed');
  }

  private async testHelloWorldFunction(): Promise<void> {
    try {
      logger.info('Testing Hello World function');

      // GET リクエストのテスト
      const getResponse = await axios.get(`${this.baseUrl}/hello?name=TestUser`);
      
      if (getResponse.status !== 200) {
        throw new Error(`Expected status 200, got ${getResponse.status}`);
      }
      
      if (!getResponse.data.message.includes('TestUser')) {
        throw new Error('Response does not contain expected name');
      }

      // POST リクエストのテスト
      const postResponse = await axios.post(`${this.baseUrl}/hello`, {
        name: 'PostTestUser',
      });
      
      if (!postResponse.data.message.includes('PostTestUser')) {
        throw new Error('POST request test failed');
      }

      logger.info('Hello World function test passed');

    } catch (error) {
      logger.error('Hello World function test failed:', error);
      throw error;
    }
  }

  private async testDataProcessingFunction(): Promise<void> {
    try {
      logger.info('Testing Data Processing function');

      const testCases = [
        {
          data: 'hello world',
          operation: 'uppercase',
          expected: 'HELLO WORLD',
        },
        {
          data: 'abcdef',
          operation: 'reverse',
          expected: 'fedcba',
        },
        {
          data: 'test string',
          operation: 'length',
          expected: 11,
        },
      ];

      for (const testCase of testCases) {
        const response = await axios.post(`${this.baseUrl}/process-data`, {
          data: testCase.data,
          operation: testCase.operation,
        });

        if (response.data.result !== testCase.expected) {
          throw new Error(
            `Test failed for operation ${testCase.operation}. Expected: ${testCase.expected}, Got: ${response.data.result}`
          );
        }
      }

      // エラーケースのテスト
      try {
        await axios.post(`${this.baseUrl}/process-data`, {
          operation: 'uppercase',
          // data フィールドなし
        });
      } catch (error) {
        if (error.response?.status !== 400) {
          throw new Error('Expected 400 error for missing data');
        }
      }

      logger.info('Data Processing function test passed');

    } catch (error) {
      logger.error('Data Processing function test failed:', error);
      throw error;
    }
  }

  private async testFileUploadFunction(): Promise<void> {
    try {
      logger.info('Testing File Upload function');

      const testFile = {
        fileName: 'test.txt',
        fileContent: 'This is a test file content',
        contentType: 'text/plain',
      };

      const response = await axios.post(`${this.baseUrl}/upload-file`, testFile);

      if (!response.data.success) {
        throw new Error('File upload should succeed');
      }

      if (!response.data.fileId) {
        throw new Error('Response should include fileId');
      }

      if (response.data.fileName !== testFile.fileName) {
        throw new Error('Response fileName does not match');
      }

      logger.info('File Upload function test passed');

    } catch (error) {
      logger.error('File Upload function test failed:', error);
      throw error;
    }
  }

  async testEventFunctions(): Promise<void> {
    logger.info('Starting event function integration tests');

    // Storage イベントのテスト
    await this.testStorageEventFunction();
    
    // Pub/Sub イベントのテスト
    await this.testPubSubEventFunction();

    logger.info('Event function integration tests completed');
  }

  private async testStorageEventFunction(): Promise<void> {
    try {
      logger.info('Testing Storage Event function');

      const storageEvent = {
        data: {
          bucket: 'test-bucket',
          name: 'test-image.jpg',
          contentType: 'image/jpeg',
          size: '1024',
          timeCreated: new Date().toISOString(),
        },
      };

      const response = await axios.post(`${this.baseUrl}/storage-event`, storageEvent, {
        headers: {
          'ce-type': 'google.cloud.storage.object.v1.finalized',
        },
      });

      if (response.status !== 200) {
        throw new Error(`Expected status 200, got ${response.status}`);
      }

      logger.info('Storage Event function test passed');

    } catch (error) {
      logger.error('Storage Event function test failed:', error);
      throw error;
    }
  }

  private async testPubSubEventFunction(): Promise<void> {
    try {
      logger.info('Testing Pub/Sub Event function');

      const messageData = {
        userId: '12345',
        action: 'user_signup',
        timestamp: new Date().toISOString(),
      };

      const pubSubEvent = {
        data: {
          message: {
            messageId: 'test-message-123',
            publishTime: new Date().toISOString(),
            data: Buffer.from(JSON.stringify(messageData)).toString('base64'),
            attributes: {
              messageType: 'user_signup',
            },
          },
        },
      };

      const response = await axios.post(`${this.baseUrl}/pubsub-event`, pubSubEvent, {
        headers: {
          'ce-type': 'google.cloud.pubsub.topic.v1.messagePublished',
        },
      });

      if (response.status !== 200) {
        throw new Error(`Expected status 200, got ${response.status}`);
      }

      logger.info('Pub/Sub Event function test passed');

    } catch (error) {
      logger.error('Pub/Sub Event function test failed:', error);
      throw error;
    }
  }
}

// テスト実行
if (require.main === module) {
  async function runTests() {
    const tester = new FunctionIntegrationTest();

    try {
      await tester.testHttpFunctions();
      await tester.testEventFunctions();
      
      logger.info('All integration tests passed successfully');
      process.exit(0);
    } catch (error) {
      logger.error('Integration tests failed:', error);
      process.exit(1);
    }
  }

  runTests();
}
```

## プロダクション運用とベストプラクティス

**🛠️ ユーティリティとヘルパー関数**

```typescript:src/utils/logger.ts
import winston from 'winston';

// Cloud Functions 向けの構造化ログ設定
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json(),
    winston.format.printf(info => {
      // Cloud Logging 向けの構造化ログフォーマット
      return JSON.stringify({
        timestamp: info.timestamp,
        severity: info.level.toUpperCase(),
        message: info.message,
        ...(info.error && { error: info.error }),
        ...(info.stack && { stack: info.stack }),
        ...info.metadata,
        // Cloud Functions のメタデータ
        function_name: process.env.K_SERVICE,
        function_revision: process.env.K_REVISION,
        execution_id: process.env.FUNCTION_EXECUTION_ID,
      });
    })
  ),
  defaultMeta: {
    service: 'cloud-functions',
  },
  transports: [
    new winston.transports.Console(),
  ],
});

// Cloud Logging との連携用ヘルパー
export const logStructured = (severity: string, message: string, metadata?: any) => {
  logger.log(severity, message, { metadata });
};

export { logger };
```

**📋 設定管理とベストプラクティス**

```typescript:src/config/index.ts
import dotenv from 'dotenv';

// 環境に応じた設定読み込み
if (process.env.NODE_ENV !== 'production') {
  dotenv.config();
}

export const config = {
  // Google Cloud 設定
  projectId: process.env.GOOGLE_CLOUD_PROJECT || process.env.GCLOUD_PROJECT || '',
  region: process.env.FUNCTION_REGION || 'asia-northeast1',
  
  // 関数設定
  defaultTimeout: parseInt(process.env.DEFAULT_TIMEOUT || '300'),
  defaultMemory: parseInt(process.env.DEFAULT_MEMORY || '256'),
  maxInstances: parseInt(process.env.MAX_INSTANCES || '100'),
  
  // ログ設定
  logLevel: process.env.LOG_LEVEL || 'info',
  enableStructuredLogging: process.env.ENABLE_STRUCTURED_LOGGING !== 'false',
  
  // 外部サービス設定
  databaseUrl: process.env.DATABASE_URL || '',
  redisUrl: process.env.REDIS_URL || '',
  storageKeyFile: process.env.GOOGLE_APPLICATION_CREDENTIALS || '',
  
  // セキュリティ設定
  corsOrigins: (process.env.CORS_ORIGINS || '').split(',').filter(Boolean),
  apiKey: process.env.API_KEY || '',
  jwtSecret: process.env.JWT_SECRET || '',
  
  // 監視設定
  enableMetrics: process.env.ENABLE_METRICS !== 'false',
  metricsInterval: parseInt(process.env.METRICS_INTERVAL || '300'),
  
  // 開発・デバッグ設定
  isDevelopment: process.env.NODE_ENV === 'development',
  isProduction: process.env.NODE_ENV === 'production',
  enableDebug: process.env.ENABLE_DEBUG === 'true',
};

// 必須設定の検証
const requiredConfigs = ['projectId'];
for (const key of requiredConfigs) {
  if (!config[key as keyof typeof config]) {
    throw new Error(`Required configuration ${key} is missing`);
  }
}

// 環境固有の設定検証
if (config.isProduction) {
  const productionRequiredConfigs = ['apiKey', 'jwtSecret'];
  for (const key of productionRequiredConfigs) {
    if (!config[key as keyof typeof config]) {
      console.warn(`Production configuration ${key} is missing`);
    }
  }
}
```

## デプロイメントスクリプトとCI/CD

```bash:scripts/deploy-functions.sh
#!/bin/bash

# Google Cloud Functions デプロイスクリプト
set -e

# 設定値
PROJECT_ID=${1:-$GOOGLE_CLOUD_PROJECT}
REGION=${2:-"asia-northeast1"}
ENVIRONMENT=${3:-"production"}

if [ -z "$PROJECT_ID" ]; then
  echo "Error: PROJECT_ID is required"
  echo "Usage: $0 <PROJECT_ID> [REGION] [ENVIRONMENT]"
  exit 1
fi

echo "🚀 Deploying Cloud Functions..."
echo "Project: $PROJECT_ID"
echo "Region: $REGION"
echo "Environment: $ENVIRONMENT"

# 1. TypeScript のビルド
echo "📦 Building TypeScript..."
npm run build

# 2. 環境変数の設定
ENV_FILE=".env.${ENVIRONMENT}"
if [ -f "$ENV_FILE" ]; then
  echo "📋 Loading environment variables from $ENV_FILE"
  export $(cat $ENV_FILE | grep -v '^#' | xargs)
fi

# 3. HTTP トリガー関数のデプロイ
echo "🌐 Deploying HTTP functions..."

gcloud functions deploy api-hello-world \
  --gen2 \
  --runtime nodejs18 \
  --region $REGION \
  --source . \
  --entry-point helloWorld \
  --trigger-http \
  --allow-unauthenticated \
  --memory 256Mi \
  --timeout 60s \
  --max-instances 100 \
  --min-instances 0 \
  --set-env-vars "NODE_ENV=${ENVIRONMENT},LOG_LEVEL=info" \
  --update-labels "environment=${ENVIRONMENT},function_type=api"

gcloud functions deploy api-process-data \
  --gen2 \
  --runtime nodejs18 \
  --region $REGION \
  --source . \
  --entry-point processData \
  --trigger-http \
  --allow-unauthenticated \
  --memory 512Mi \
  --timeout 300s \
  --max-instances 50 \
  --min-instances 1 \
  --set-env-vars "NODE_ENV=${ENVIRONMENT},LOG_LEVEL=info,MAX_PROCESSING_SIZE=10MB" \
  --update-labels "environment=${ENVIRONMENT},function_type=processing"

# 4. Storage トリガー関数のデプロイ
echo "📁 Deploying Storage event functions..."

gcloud functions deploy storage-processor \
  --gen2 \
  --runtime nodejs18 \
  --region $REGION \
  --source . \
  --entry-point processStorageEvent \
  --trigger-bucket "${PROJECT_ID}-uploads" \
  --memory 1024Mi \
  --timeout 540s \
  --max-instances 10 \
  --min-instances 0 \
  --set-env-vars "NODE_ENV=${ENVIRONMENT},THUMBNAIL_BUCKET=${PROJECT_ID}-thumbnails,PROCESSED_BUCKET=${PROJECT_ID}-processed" \
  --update-labels "environment=${ENVIRONMENT},function_type=event_driven"

# 5. Pub/Sub トリガー関数のデプロイ
echo "📨 Deploying Pub/Sub event functions..."

# Pub/Sub トピックが存在しない場合は作成
if ! gcloud pubsub topics describe notifications --project=$PROJECT_ID 2>/dev/null; then
  echo "📢 Creating Pub/Sub topic: notifications"
  gcloud pubsub topics create notifications --project=$PROJECT_ID
fi

gcloud functions deploy pubsub-processor \
  --gen2 \
  --runtime nodejs18 \
  --region $REGION \
  --source . \
  --entry-point processPubSubMessage \
  --trigger-topic notifications \
  --memory 512Mi \
  --timeout 300s \
  --max-instances 20 \
  --min-instances 0 \
  --set-env-vars "NODE_ENV=${ENVIRONMENT},NOTIFICATION_SERVICE_URL=${NOTIFICATION_SERVICE_URL}" \
  --update-labels "environment=${ENVIRONMENT},function_type=message_processing"

# 6. Cloud Scheduler ジョブの作成
echo "⏰ Setting up scheduled functions..."

# Cloud Scheduler ジョブの作成（存在しない場合のみ）
if ! gcloud scheduler jobs describe daily-cleanup --location=$REGION --project=$PROJECT_ID 2>/dev/null; then
  echo "📅 Creating Cloud Scheduler job: daily-cleanup"
  gcloud scheduler jobs create http daily-cleanup \
    --location $REGION \
    --schedule "0 2 * * *" \
    --uri "https://$REGION-$PROJECT_ID.cloudfunctions.net/scheduled-cleanup" \
    --http-method POST \
    --project $PROJECT_ID
fi

gcloud functions deploy scheduled-cleanup \
  --gen2 \
  --runtime nodejs18 \
  --region $REGION \
  --source . \
  --entry-point scheduledFunction \
  --trigger-http \
  --no-allow-unauthenticated \
  --memory 1024Mi \
  --timeout 540s \
  --max-instances 1 \
  --min-instances 0 \
  --set-env-vars "NODE_ENV=${ENVIRONMENT},CLEANUP_DAYS=30,BACKUP_BUCKET=${PROJECT_ID}-backups" \
  --update-labels "environment=${ENVIRONMENT},function_type=scheduled"

echo "✅ All functions deployed successfully!"

# 7. デプロイ結果の確認
echo "📋 Deployment summary:"
gcloud functions list --regions=$REGION --project=$PROJECT_ID --format="table(name,runtime,trigger,status)"

echo ""
echo "🌍 Function URLs:"
gcloud functions describe api-hello-world --region=$REGION --project=$PROJECT_ID --format="value(serviceConfig.uri)"
gcloud functions describe api-process-data --region=$REGION --project=$PROJECT_ID --format="value(serviceConfig.uri)"
```

## まとめ

Google Cloud Functions × TypeScript の組み合わせで、サーバーレス関数開発の生産性と運用効率が飛躍的に向上する環境を構築できました。

**✅ この記事で身についたスキル**

>* Cloud Functions の作成・管理とライフサイクル制御
>* TypeScript + Cloud Functions API での実用的な関数管理自動化
>* HTTP・イベント・スケジュールトリガーの実装パターン
>* 監視・ログ管理・エラーハンドリングの運用手法
>* ローカル開発からプロダクション運用までの完全ワークフロー

**🎯 次のステップ**

>* Eventarc との連携による高度なイベント駆動アーキテクチャ
>* Cloud Run との使い分けとマイクロサービス連携
>* Firebase Functions との統合開発環境の構築

Cloud Functions なら、従来のサーバー運用の複雑さを一切気にせず、ビジネスロジックの実装に集中できます。イベント駆動でスケーラブル、かつコスト効率の高いサーバーレスアーキテクチャを、あなたも今日から始めてみませんか？