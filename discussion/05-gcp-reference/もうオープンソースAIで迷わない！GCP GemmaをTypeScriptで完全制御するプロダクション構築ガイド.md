## もうオープンソースAIで迷わない！GCP GemmaをTypeScriptで完全制御するプロダクション構築ガイド

## はじめに

オープンソースのAIモデルを本格的に活用したいと考えている開発者にとって、こんな悩みはありませんか？

> 「オープンソースAIって実際どこまで使えるの？」
> 「GCPでGemmaを運用するのって複雑そう...」
> 「TypeScriptでプロダクション環境に組み込める？」

本記事では、**GCP上でGemmaモデルをTypeScriptで完全制御し、プロダクション対応のAIシステムを構築する方法**を、実際のコード例とともに詳しく解説します。

## GCP Gemmaとは

GCP Gemmaは、**Googleが開発したオープンウェイトの大規模言語モデル**で、Google Cloudプラットフォームで簡単にデプロイ・運用できます。

| 特徴 | 内容 | メリット |
|------|------|----------|
| **オープンウェイト** | モデルの重みが公開されている | カスタマイズとファインチューニングが可能 |
| **複数バリエーション** | 1B〜27Bパラメーターの幅広いサイズ | 用途に応じた最適選択 |
| **マルチモーダル対応** | テキスト、画像、コードに対応 | 多様なユースケースに活用 |
| **プロダクション対応** | GCP統合でスケーラブルな運用 | エンタープライズ級の安定性 |

**2025年版 Gemmaモデル一覧**

| モデル | サイズ | 特徴 | 適用分野 |
|--------|--------|------|----------|
| **Gemma 2** | 2B, 9B, 27B | 第2世代の基盤モデル | テキスト生成、要約、抽出 |
| **CodeGemma** | 2B, 7B | コード特化型 | コード生成、デバッグ、リファクタリング |
| **PaliGemma** | 3B | 視覚言語モデル | 画像解析、キャプション生成 |
| **MedGemma** | 2B, 7B | 医療特化型 | 医療文書処理、診断支援 |
| **ShieldGemma 2** | 2B, 9B, 27B | 安全性評価特化 | コンテンツフィルタリング |

## TypeScript環境のセットアップ

### 1\. プロジェクトの初期化

```bash
# 新しいプロジェクトを作成
mkdir gemma-typescript-app
cd gemma-typescript-app

# パッケージの初期化
npm init -y

# 必要な依存関係をインストール
npm install @google-cloud/aiplatform @google-cloud/storage
npm install express cors helmet rate-limiter-flexible
npm install @types/express @types/cors @types/node
npm install typescript ts-node nodemon dotenv
```

### 2\. TypeScript設定

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 3\. 環境変数設定

```typescript:src/config/environment.ts
import * as dotenv from 'dotenv';

dotenv.config();

export const CONFIG = {
  // GCP設定
  gcp: {
    projectId: process.env.GOOGLE_CLOUD_PROJECT || '',
    location: process.env.GOOGLE_CLOUD_LOCATION || 'us-central1',
    keyPath: process.env.GOOGLE_APPLICATION_CREDENTIALS || '',
  },
  
  // Vertex AI設定
  vertexAI: {
    apiEndpoint: `${process.env.GOOGLE_CLOUD_LOCATION || 'us-central1'}-aiplatform.googleapis.com`,
    modelEndpoints: {
      gemma2_2b: process.env.GEMMA_2B_ENDPOINT || '',
      gemma2_9b: process.env.GEMMA_9B_ENDPOINT || '',
      gemma2_27b: process.env.GEMMA_27B_ENDPOINT || '',
      codegemma_7b: process.env.CODEGEMMA_7B_ENDPOINT || '',
      paligemma_3b: process.env.PALIGEMMA_3B_ENDPOINT || ''
    }
  },
  
  // サーバー設定
  server: {
    port: parseInt(process.env.PORT || '3000'),
    environment: process.env.NODE_ENV || 'development'
  },
  
  // Redis設定（キャッシュ・レート制限用）
  redis: {
    url: process.env.REDIS_URL || 'redis://localhost:6379',
    keyPrefix: process.env.REDIS_KEY_PREFIX || 'gemma:',
    ttl: parseInt(process.env.CACHE_TTL || '3600')
  }
} as const;

// 必須環境変数のバリデーション
export function validateEnvironment(): void {
  const required = [
    'GOOGLE_CLOUD_PROJECT',
    'GOOGLE_CLOUD_LOCATION',
    'GOOGLE_APPLICATION_CREDENTIALS'
  ];
  
  const missing = required.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(`必須の環境変数が設定されていません: ${missing.join(', ')}`);
  }
}
```

## Gemmaクライアントの実装

### 1\. 基本的なGemmaクライアント

```typescript:src/services/gemma-client.ts
import { PredictionServiceClient } from '@google-cloud/aiplatform';
import { CONFIG, validateEnvironment } from '../config/environment';

export interface GemmaModelConfig {
  maxTokens?: number;
  temperature?: number;
  topP?: number;
  topK?: number;
  stopSequences?: string[];
}

export interface GemmaPrediction {
  text: string;
  finishReason: string;
  metadata?: Record<string, any>;
}

export class GemmaClient {
  private client: PredictionServiceClient;
  
  constructor() {
    validateEnvironment();
    
    this.client = new PredictionServiceClient({
      apiEndpoint: CONFIG.vertexAI.apiEndpoint,
      keyFilename: CONFIG.gcp.keyPath
    });
  }
  
  /**
   * Gemma 2モデルでテキスト生成
   */
  async generateText(
    prompt: string,
    modelSize: '2b' | '9b' | '27b' = '9b',
    config: GemmaModelConfig = {}
  ): Promise<GemmaPrediction> {
    const endpoint = this.getModelEndpoint('gemma2', modelSize);
    
    const instance = {
      prompt,
      max_tokens: config.maxTokens || 1024,
      temperature: config.temperature || 0.7,
      top_p: config.topP || 0.9,
      top_k: config.topK || 40,
      stop_sequences: config.stopSequences || []
    };
    
    try {
      const [response] = await this.client.predict({
        endpoint,
        instances: [instance]
      });
      
      return this.parsePredictionResponse(response);
    } catch (error) {
      console.error('Gemma prediction error:', error);
      throw new Error(`テキスト生成に失敗しました: ${error}`);
    }
  }
  
  /**
   * CodeGemmaでコード生成
   */
  async generateCode(
    prompt: string,
    language: string = 'typescript',
    modelSize: '2b' | '7b' = '7b'
  ): Promise<GemmaPrediction> {
    const endpoint = this.getModelEndpoint('codegemma', modelSize);
    
    const codePrompt = `
# ${language.toUpperCase()}コード生成

${prompt}

\`\`\`${language}
`;
    
    const instance = {
      prompt: codePrompt,
      max_tokens: 2048,
      temperature: 0.2,
      top_p: 0.9,
      stop_sequences: ['```', '\n\n\n']
    };
    
    try {
      const [response] = await this.client.predict({
        endpoint,
        instances: [instance]
      });
      
      const result = this.parsePredictionResponse(response);
      
      // コードブロックの後処理
      result.text = this.cleanupCodeOutput(result.text, language);
      
      return result;
    } catch (error) {
      console.error('CodeGemma prediction error:', error);
      throw new Error(`コード生成に失敗しました: ${error}`);
    }
  }
  
  /**
   * PaliGemmaで画像解析
   */
  async analyzeImage(
    imageBase64: string,
    prompt: string = 'この画像について説明してください'
  ): Promise<GemmaPrediction> {
    const endpoint = this.getModelEndpoint('paligemma', '3b');
    
    const instance = {
      image: {
        bytes_base64_encoded: imageBase64
      },
      prompt,
      max_tokens: 1024,
      temperature: 0.4
    };
    
    try {
      const [response] = await this.client.predict({
        endpoint,
        instances: [instance]
      });
      
      return this.parsePredictionResponse(response);
    } catch (error) {
      console.error('PaliGemma prediction error:', error);
      throw new Error(`画像解析に失敗しました: ${error}`);
    }
  }
  
  /**
   * バッチ処理
   */
  async generateBatch(
    prompts: string[],
    modelSize: '2b' | '9b' | '27b' = '9b',
    config: GemmaModelConfig = {}
  ): Promise<GemmaPrediction[]> {
    const endpoint = this.getModelEndpoint('gemma2', modelSize);
    
    const instances = prompts.map(prompt => ({
      prompt,
      max_tokens: config.maxTokens || 1024,
      temperature: config.temperature || 0.7,
      top_p: config.topP || 0.9,
      top_k: config.topK || 40
    }));
    
    try {
      const [response] = await this.client.predict({
        endpoint,
        instances
      });
      
      return response.predictions?.map(pred => 
        this.parsePredictionResponse({ predictions: [pred] })
      ) || [];
    } catch (error) {
      console.error('Batch prediction error:', error);
      throw new Error(`バッチ生成に失敗しました: ${error}`);
    }
  }
  
  /**
   * モデルエンドポイントを取得
   */
  private getModelEndpoint(modelType: string, size: string): string {
    const endpointKey = `${modelType}_${size}` as keyof typeof CONFIG.vertexAI.modelEndpoints;
    const endpoint = CONFIG.vertexAI.modelEndpoints[endpointKey];
    
    if (!endpoint) {
      throw new Error(`モデルエンドポイントが設定されていません: ${endpointKey}`);
    }
    
    return this.client.endpointPath(
      CONFIG.gcp.projectId,
      CONFIG.gcp.location,
      endpoint
    );
  }
  
  /**
   * レスポンスを解析
   */
  private parsePredictionResponse(response: any): GemmaPrediction {
    const prediction = response.predictions?.[0];
    
    if (!prediction) {
      throw new Error('予測結果が取得できませんでした');
    }
    
    return {
      text: prediction.generated_text || prediction.text || '',
      finishReason: prediction.finish_reason || 'complete',
      metadata: {
        tokenCount: prediction.token_count,
        safetyRatings: prediction.safety_ratings
      }
    };
  }
  
  /**
   * コード出力の後処理
   */
  private cleanupCodeOutput(text: string, language: string): string {
    // コードブロックの開始・終了タグを除去
    let cleaned = text.replace(/^```\w*\n?/, '').replace(/\n?```$/, '');
    
    // 不要な説明文を除去
    const lines = cleaned.split('\n');
    const codeStartIndex = lines.findIndex(line => 
      line.trim().length > 0 && !line.startsWith('#') && !line.startsWith('//')
    );
    
    if (codeStartIndex > 0) {
      cleaned = lines.slice(codeStartIndex).join('\n');
    }
    
    return cleaned.trim();
  }
}
```

### 2\. キャッシュ機能付きクライアント

```typescript:src/services/cached-gemma-client.ts
import { GemmaClient, GemmaModelConfig, GemmaPrediction } from './gemma-client';
import { createHash } from 'crypto';
import Redis from 'ioredis';
import { CONFIG } from '../config/environment';

export class CachedGemmaClient extends GemmaClient {
  private redis: Redis;
  
  constructor() {
    super();
    this.redis = new Redis(CONFIG.redis.url, {
      keyPrefix: CONFIG.redis.keyPrefix
    });
  }
  
  /**
   * キャッシュ機能付きテキスト生成
   */
  async generateTextCached(
    prompt: string,
    modelSize: '2b' | '9b' | '27b' = '9b',
    config: GemmaModelConfig = {},
    cacheKey?: string
  ): Promise<GemmaPrediction> {
    const key = cacheKey || this.generateCacheKey('text', prompt, modelSize, config);
    
    // キャッシュから取得を試行
    const cached = await this.redis.get(key);
    if (cached) {
      console.log(`キャッシュヒット: ${key}`);
      return JSON.parse(cached);
    }
    
    // キャッシュにない場合は新規生成
    const result = await this.generateText(prompt, modelSize, config);
    
    // 結果をキャッシュに保存
    await this.redis.setex(key, CONFIG.redis.ttl, JSON.stringify(result));
    console.log(`キャッシュ保存: ${key}`);
    
    return result;
  }
  
  /**
   * キャッシュ機能付きコード生成
   */
  async generateCodeCached(
    prompt: string,
    language: string = 'typescript',
    modelSize: '2b' | '7b' = '7b',
    cacheKey?: string
  ): Promise<GemmaPrediction> {
    const key = cacheKey || this.generateCacheKey('code', prompt, modelSize, { language });
    
    const cached = await this.redis.get(key);
    if (cached) {
      console.log(`コードキャッシュヒット: ${key}`);
      return JSON.parse(cached);
    }
    
    const result = await this.generateCode(prompt, language, modelSize);
    await this.redis.setex(key, CONFIG.redis.ttl, JSON.stringify(result));
    
    return result;
  }
  
  /**
   * キャッシュの手動削除
   */
  async clearCache(pattern?: string): Promise<number> {
    const searchPattern = pattern || '*';
    const keys = await this.redis.keys(searchPattern);
    
    if (keys.length === 0) {
      return 0;
    }
    
    return await this.redis.del(...keys);
  }
  
  /**
   * キャッシュ統計の取得
   */
  async getCacheStats(): Promise<{
    totalKeys: number;
    memoryUsage: string;
    hitRate: number;
  }> {
    const info = await this.redis.info('memory');
    const keyCount = await this.redis.dbsize();
    
    // メモリ使用量を解析
    const memoryMatch = info.match(/used_memory_human:([^\r\n]+)/);
    const memoryUsage = memoryMatch ? memoryMatch[1] : 'unknown';
    
    return {
      totalKeys: keyCount,
      memoryUsage,
      hitRate: 0 // 実際の実装ではヒット率計算ロジックを追加
    };
  }
  
  /**
   * キャッシュキーを生成
   */
  private generateCacheKey(
    type: string,
    prompt: string,
    modelSize: string,
    config: any
  ): string {
    const data = JSON.stringify({ type, prompt, modelSize, config });
    const hash = createHash('md5').update(data).digest('hex');
    return `${type}:${modelSize}:${hash}`;
  }
  
  /**
   * リソースのクリーンアップ
   */
  async disconnect(): Promise<void> {
    await this.redis.disconnect();
  }
}
```

## Web APIサーバーの実装

### 1\. Express APIサーバー

```typescript:src/api/server.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import { rateLimit } from 'express-rate-limit';
import { CachedGemmaClient } from '../services/cached-gemma-client';
import { CONFIG } from '../config/environment';

const app = express();
const gemmaClient = new CachedGemmaClient();

// ミドルウェア設定
app.use(helmet());
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
  credentials: true
}));
app.use(express.json({ limit: '10mb' }));

// レート制限
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // リクエスト数上限
  message: 'Too many requests, please try again later.',
  standardHeaders: true,
  legacyHeaders: false,
});
app.use('/api', apiLimiter);

// より厳しい制限（AI生成系）
const aiLimiter = rateLimit({
  windowMs: 60 * 1000, // 1分
  max: 10, // AI生成は1分10回まで
  message: 'AI generation rate limit exceeded.',
});

/**
 * テキスト生成API
 */
app.post('/api/generate/text', aiLimiter, async (req, res) => {
  try {
    const {
      prompt,
      modelSize = '9b',
      maxTokens = 1024,
      temperature = 0.7,
      useCache = true
    } = req.body;
    
    if (!prompt || typeof prompt !== 'string') {
      return res.status(400).json({
        error: 'プロンプトは必須です'
      });
    }
    
    const config = {
      maxTokens,
      temperature,
      topP: 0.9,
      topK: 40
    };
    
    const startTime = Date.now();
    
    const result = useCache
      ? await gemmaClient.generateTextCached(prompt, modelSize, config)
      : await gemmaClient.generateText(prompt, modelSize, config);
    
    const responseTime = Date.now() - startTime;
    
    res.json({
      success: true,
      data: {
        text: result.text,
        finishReason: result.finishReason,
        metadata: {
          ...result.metadata,
          responseTime,
          modelSize,
          cached: useCache
        }
      }
    });
  } catch (error) {
    console.error('Text generation error:', error);
    res.status(500).json({
      error: 'テキスト生成に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * コード生成API
 */
app.post('/api/generate/code', aiLimiter, async (req, res) => {
  try {
    const {
      prompt,
      language = 'typescript',
      modelSize = '7b',
      useCache = true
    } = req.body;
    
    if (!prompt) {
      return res.status(400).json({
        error: 'プロンプトは必須です'
      });
    }
    
    const startTime = Date.now();
    
    const result = useCache
      ? await gemmaClient.generateCodeCached(prompt, language, modelSize)
      : await gemmaClient.generateCode(prompt, language, modelSize);
    
    const responseTime = Date.now() - startTime;
    
    res.json({
      success: true,
      data: {
        code: result.text,
        language,
        finishReason: result.finishReason,
        metadata: {
          ...result.metadata,
          responseTime,
          modelSize
        }
      }
    });
  } catch (error) {
    console.error('Code generation error:', error);
    res.status(500).json({
      error: 'コード生成に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * 画像解析API
 */
app.post('/api/analyze/image', aiLimiter, async (req, res) => {
  try {
    const { image, prompt = 'この画像について説明してください' } = req.body;
    
    if (!image) {
      return res.status(400).json({
        error: '画像データは必須です'
      });
    }
    
    // Base64データの検証
    const base64Regex = /^data:image\/(png|jpg|jpeg|gif|webp);base64,/;
    if (!base64Regex.test(image)) {
      return res.status(400).json({
        error: '無効な画像フォーマットです'
      });
    }
    
    // Base64プレフィックスを除去
    const imageBase64 = image.replace(base64Regex, '');
    
    const startTime = Date.now();
    const result = await gemmaClient.analyzeImage(imageBase64, prompt);
    const responseTime = Date.now() - startTime;
    
    res.json({
      success: true,
      data: {
        analysis: result.text,
        finishReason: result.finishReason,
        metadata: {
          ...result.metadata,
          responseTime
        }
      }
    });
  } catch (error) {
    console.error('Image analysis error:', error);
    res.status(500).json({
      error: '画像解析に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * バッチ処理API
 */
app.post('/api/generate/batch', aiLimiter, async (req, res) => {
  try {
    const {
      prompts,
      modelSize = '9b',
      maxTokens = 1024,
      temperature = 0.7
    } = req.body;
    
    if (!Array.isArray(prompts) || prompts.length === 0) {
      return res.status(400).json({
        error: 'プロンプト配列は必須です'
      });
    }
    
    if (prompts.length > 10) {
      return res.status(400).json({
        error: 'バッチサイズは10個までです'
      });
    }
    
    const config = { maxTokens, temperature };
    const startTime = Date.now();
    
    const results = await gemmaClient.generateBatch(prompts, modelSize, config);
    const responseTime = Date.now() - startTime;
    
    res.json({
      success: true,
      data: {
        results: results.map((result, index) => ({
          prompt: prompts[index],
          text: result.text,
          finishReason: result.finishReason,
          metadata: result.metadata
        })),
        batchSize: prompts.length,
        totalResponseTime: responseTime
      }
    });
  } catch (error) {
    console.error('Batch processing error:', error);
    res.status(500).json({
      error: 'バッチ処理に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * キャッシュ管理API
 */
app.get('/api/cache/stats', async (req, res) => {
  try {
    const stats = await gemmaClient.getCacheStats();
    res.json({
      success: true,
      data: stats
    });
  } catch (error) {
    console.error('Cache stats error:', error);
    res.status(500).json({
      error: 'キャッシュ統計の取得に失敗しました'
    });
  }
});

app.delete('/api/cache', async (req, res) => {
  try {
    const { pattern } = req.query;
    const deletedCount = await gemmaClient.clearCache(pattern as string);
    
    res.json({
      success: true,
      data: {
        deletedKeys: deletedCount,
        message: `${deletedCount}個のキャッシュエントリを削除しました`
      }
    });
  } catch (error) {
    console.error('Cache clear error:', error);
    res.status(500).json({
      error: 'キャッシュクリアに失敗しました'
    });
  }
});

/**
 * ヘルスチェックAPI
 */
app.get('/api/health', (req, res) => {
  res.json({
    status: 'OK',
    timestamp: new Date().toISOString(),
    environment: CONFIG.server.environment,
    uptime: process.uptime()
  });
});

// エラーハンドリング
app.use((error: any, req: any, res: any, next: any) => {
  console.error('Unhandled error:', error);
  res.status(500).json({
    error: '内部サーバーエラーが発生しました',
    requestId: req.headers['x-request-id']
  });
});

// サーバー起動
const server = app.listen(CONFIG.server.port, () => {
  console.log(`🚀 Gemma API Server started on port ${CONFIG.server.port}`);
  console.log(`🌍 Environment: ${CONFIG.server.environment}`);
});

// グレースフルシャットダウン
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully...');
  server.close(async () => {
    await gemmaClient.disconnect();
    process.exit(0);
  });
});

export default app;
```

### 2\. フロントエンド連携用TypeScriptクライアント

```typescript:src/client/gemma-api-client.ts
export interface GemmaAPIResponse<T = any> {
  success: boolean;
  data?: T;
  error?: string;
  message?: string;
}

export interface GenerationOptions {
  modelSize?: '2b' | '9b' | '27b';
  maxTokens?: number;
  temperature?: number;
  useCache?: boolean;
}

export interface CodeGenerationOptions {
  language?: string;
  modelSize?: '2b' | '7b';
  useCache?: boolean;
}

export class GemmaAPIClient {
  private baseUrl: string;
  private apiKey?: string;
  
  constructor(baseUrl: string, apiKey?: string) {
    this.baseUrl = baseUrl.replace(/\/$/, '');
    this.apiKey = apiKey;
  }
  
  /**
   * テキスト生成
   */
  async generateText(
    prompt: string,
    options: GenerationOptions = {}
  ): Promise<{
    text: string;
    finishReason: string;
    metadata: any;
  }> {
    const response = await this.request<any>('/api/generate/text', {
      method: 'POST',
      body: { prompt, ...options }
    });
    
    if (!response.success) {
      throw new Error(response.error || 'テキスト生成に失敗しました');
    }
    
    return response.data;
  }
  
  /**
   * コード生成
   */
  async generateCode(
    prompt: string,
    options: CodeGenerationOptions = {}
  ): Promise<{
    code: string;
    language: string;
    finishReason: string;
    metadata: any;
  }> {
    const response = await this.request<any>('/api/generate/code', {
      method: 'POST',
      body: { prompt, ...options }
    });
    
    if (!response.success) {
      throw new Error(response.error || 'コード生成に失敗しました');
    }
    
    return response.data;
  }
  
  /**
   * 画像解析
   */
  async analyzeImage(
    imageFile: File,
    prompt: string = 'この画像について説明してください'
  ): Promise<{
    analysis: string;
    finishReason: string;
    metadata: any;
  }> {
    const imageBase64 = await this.fileToBase64(imageFile);
    
    const response = await this.request<any>('/api/analyze/image', {
      method: 'POST',
      body: {
        image: imageBase64,
        prompt
      }
    });
    
    if (!response.success) {
      throw new Error(response.error || '画像解析に失敗しました');
    }
    
    return response.data;
  }
  
  /**
   * バッチ処理
   */
  async generateBatch(
    prompts: string[],
    options: GenerationOptions = {}
  ): Promise<{
    results: Array<{
      prompt: string;
      text: string;
      finishReason: string;
      metadata: any;
    }>;
    batchSize: number;
    totalResponseTime: number;
  }> {
    const response = await this.request<any>('/api/generate/batch', {
      method: 'POST',
      body: { prompts, ...options }
    });
    
    if (!response.success) {
      throw new Error(response.error || 'バッチ処理に失敗しました');
    }
    
    return response.data;
  }
  
  /**
   * キャッシュ統計取得
   */
  async getCacheStats(): Promise<{
    totalKeys: number;
    memoryUsage: string;
    hitRate: number;
  }> {
    const response = await this.request<any>('/api/cache/stats');
    
    if (!response.success) {
      throw new Error(response.error || 'キャッシュ統計の取得に失敗しました');
    }
    
    return response.data;
  }
  
  /**
   * キャッシュクリア
   */
  async clearCache(pattern?: string): Promise<{
    deletedKeys: number;
    message: string;
  }> {
    const url = pattern ? `/api/cache?pattern=${encodeURIComponent(pattern)}` : '/api/cache';
    const response = await this.request<any>(url, { method: 'DELETE' });
    
    if (!response.success) {
      throw new Error(response.error || 'キャッシュクリアに失敗しました');
    }
    
    return response.data;
  }
  
  /**
   * ヘルスチェック
   */
  async healthCheck(): Promise<{
    status: string;
    timestamp: string;
    environment: string;
    uptime: number;
  }> {
    const response = await this.request<any>('/api/health');
    return response;
  }
  
  /**
   * HTTP リクエスト実行
   */
  private async request<T>(
    endpoint: string,
    options: {
      method?: string;
      body?: any;
      headers?: Record<string, string>;
    } = {}
  ): Promise<GemmaAPIResponse<T>> {
    const url = `${this.baseUrl}${endpoint}`;
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
      ...options.headers
    };
    
    if (this.apiKey) {
      headers['Authorization'] = `Bearer ${this.apiKey}`;
    }
    
    try {
      const response = await fetch(url, {
        method: options.method || 'GET',
        headers,
        body: options.body ? JSON.stringify(options.body) : undefined
      });
      
      const data = await response.json();
      
      if (!response.ok) {
        throw new Error(data.error || `HTTP ${response.status}: ${response.statusText}`);
      }
      
      return data;
    } catch (error) {
      console.error('API request failed:', error);
      throw error;
    }
  }
  
  /**
   * ファイルをBase64に変換
   */
  private fileToBase64(file: File): Promise<string> {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.onload = () => resolve(reader.result as string);
      reader.onerror = reject;
      reader.readAsDataURL(file);
    });
  }
}

// 使用例
export function createGemmaClient(): GemmaAPIClient {
  return new GemmaAPIClient(
    process.env.NEXT_PUBLIC_GEMMA_API_URL || 'http://localhost:3000',
    process.env.NEXT_PUBLIC_GEMMA_API_KEY
  );
}
```

## 実用的なアプリケーション例

### 1\. インテリジェントなコードレビューツール

```typescript:src/examples/code-review-tool.ts
import { CachedGemmaClient } from '../services/cached-gemma-client';

export interface CodeReviewResult {
  overallScore: number;
  issues: Array<{
    type: 'security' | 'performance' | 'style' | 'bug' | 'suggestion';
    severity: 'critical' | 'major' | 'minor';
    line?: number;
    message: string;
    suggestion?: string;
  }>;
  summary: string;
  improvedCode?: string;
}

export class IntelligentCodeReviewer {
  private gemmaClient: CachedGemmaClient;
  
  constructor() {
    this.gemmaClient = new CachedGemmaClient();
  }
  
  /**
   * コードレビューを実行
   */
  async reviewCode(
    code: string,
    language: string = 'typescript',
    includeImprovedVersion: boolean = true
  ): Promise<CodeReviewResult> {
    const reviewPrompt = `
あなたは経験豊富なシニアソフトウェアエンジニアです。
以下の${language}コードを総合的にレビューし、JSON形式で結果を返してください。

## レビュー対象コード
\`\`\`${language}
${code}
\`\`\`

## 期待する出力形式（JSON）
{
  "overallScore": [1-10の評価点],
  "issues": [
    {
      "type": "security|performance|style|bug|suggestion",
      "severity": "critical|major|minor",
      "line": [該当行番号（任意）],
      "message": "具体的な問題点",
      "suggestion": "改善提案（任意）"
    }
  ],
  "summary": "総合的な評価コメント（日本語）",
  "improvedCode": "${includeImprovedVersion ? '改善されたコード（任意）' : ''}"
}

## レビュー観点
1. セキュリティ脆弱性
2. パフォーマンス問題
3. コーディングスタイル
4. 潜在的バグ
5. 保守性・可読性
6. エラーハンドリング
7. ベストプラクティスの遵守
`;
    
    try {
      const result = await this.gemmaClient.generateTextCached(
        reviewPrompt,
        '9b',
        {
          temperature: 0.3, // 一貫性重視
          maxTokens: 2048
        },
        `code-review-${language}-${this.hashCode(code)}`
      );
      
      // JSONを解析
      const jsonStart = result.text.indexOf('{');
      const jsonEnd = result.text.lastIndexOf('}') + 1;
      
      if (jsonStart === -1 || jsonEnd === 0) {
        throw new Error('JSON形式の結果が得られませんでした');
      }
      
      const jsonStr = result.text.slice(jsonStart, jsonEnd);
      const reviewResult: CodeReviewResult = JSON.parse(jsonStr);
      
      // 結果の後処理・検証
      this.validateReviewResult(reviewResult);
      
      return reviewResult;
    } catch (error) {
      console.error('Code review failed:', error);
      throw new Error(`コードレビューに失敗しました: ${error}`);
    }
  }
  
  /**
   * 複数ファイルの一括レビュー
   */
  async reviewMultipleFiles(
    files: Array<{ filename: string; content: string; language: string }>
  ): Promise<Array<{ filename: string; review: CodeReviewResult }>> {
    const results = [];
    
    for (const file of files) {
      try {
        console.log(`Reviewing ${file.filename}...`);
        const review = await this.reviewCode(
          file.content,
          file.language,
          false // 一括処理では改善コードは含めない
        );
        
        results.push({
          filename: file.filename,
          review
        });
      } catch (error) {
        console.error(`Failed to review ${file.filename}:`, error);
        results.push({
          filename: file.filename,
          review: {
            overallScore: 0,
            issues: [{
              type: 'bug',
              severity: 'critical',
              message: `レビューエラー: ${error}`,
            }],
            summary: 'レビュー処理中にエラーが発生しました',
          } as CodeReviewResult
        });
      }
    }
    
    return results;
  }
  
  /**
   * プロジェクト全体のサマリーレポート生成
   */
  async generateProjectSummary(
    reviews: Array<{ filename: string; review: CodeReviewResult }>
  ): Promise<{
    overallHealth: 'excellent' | 'good' | 'fair' | 'poor';
    totalIssues: number;
    criticalIssues: number;
    averageScore: number;
    recommendations: string[];
    topIssueTypes: Array<{ type: string; count: number }>;
  }> {
    const totalFiles = reviews.length;
    const totalScore = reviews.reduce((sum, r) => sum + r.review.overallScore, 0);
    const averageScore = totalScore / totalFiles;
    
    const allIssues = reviews.flatMap(r => r.review.issues);
    const criticalIssues = allIssues.filter(i => i.severity === 'critical').length;
    
    // 問題タイプの集計
    const issueTypeCounts = allIssues.reduce((acc, issue) => {
      acc[issue.type] = (acc[issue.type] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);
    
    const topIssueTypes = Object.entries(issueTypeCounts)
      .map(([type, count]) => ({ type, count }))
      .sort((a, b) => b.count - a.count)
      .slice(0, 5);
    
    // 全体的な健康度判定
    let overallHealth: 'excellent' | 'good' | 'fair' | 'poor';
    if (averageScore >= 9 && criticalIssues === 0) {
      overallHealth = 'excellent';
    } else if (averageScore >= 7 && criticalIssues <= 2) {
      overallHealth = 'good';
    } else if (averageScore >= 5 && criticalIssues <= 5) {
      overallHealth = 'fair';
    } else {
      overallHealth = 'poor';
    }
    
    // 推奨事項の生成
    const recommendations = [
      criticalIssues > 0 ? `${criticalIssues}個のクリティカルな問題を優先的に修正してください` : null,
      topIssueTypes[0] ? `「${topIssueTypes[0].type}」関連の問題が多く見つかりました（${topIssueTypes[0].count}件）` : null,
      averageScore < 7 ? 'コード品質の全体的な改善が必要です' : null,
      '定期的なコードレビューの実施を継続してください'
    ].filter(Boolean) as string[];
    
    return {
      overallHealth,
      totalIssues: allIssues.length,
      criticalIssues,
      averageScore: Number(averageScore.toFixed(2)),
      recommendations,
      topIssueTypes
    };
  }
  
  /**
   * レビュー結果の検証
   */
  private validateReviewResult(result: CodeReviewResult): void {
    if (typeof result.overallScore !== 'number' || result.overallScore < 1 || result.overallScore > 10) {
      result.overallScore = 5; // デフォルト値
    }
    
    if (!Array.isArray(result.issues)) {
      result.issues = [];
    }
    
    if (typeof result.summary !== 'string') {
      result.summary = '総合的な評価コメントが生成できませんでした';
    }
  }
  
  /**
   * 文字列のハッシュ値を生成
   */
  private hashCode(str: string): string {
    let hash = 0;
    for (let i = 0; i < str.length; i++) {
      const char = str.charCodeAt(i);
      hash = ((hash << 5) - hash) + char;
      hash = hash & hash; // 32bit整数に変換
    }
    return Math.abs(hash).toString(16);
  }
}

// 使用例
export async function demonstrateCodeReview() {
  const reviewer = new IntelligentCodeReviewer();
  
  const sampleCode = `
function fetchUserData(userId) {
  const data = fetch('/api/users/' + userId)
    .then(response => response.json())
    .then(data => {
      localStorage.setItem('userData', JSON.stringify(data));
      return data;
    })
    .catch(error => {
      console.log(error);
    });
  return data;
}
`;
  
  console.log('🔍 コードレビューを実行中...');
  const review = await reviewer.reviewCode(sampleCode, 'javascript');
  
  console.log('📊 レビュー結果:');
  console.log(`評価点: ${review.overallScore}/10`);
  console.log(`問題数: ${review.issues.length}`);
  console.log(`総評: ${review.summary}`);
  
  if (review.issues.length > 0) {
    console.log('\n🚨 発見された問題:');
    review.issues.forEach((issue, index) => {
      console.log(`${index + 1}. [${issue.severity.toUpperCase()}] ${issue.type}: ${issue.message}`);
      if (issue.suggestion) {
        console.log(`   💡 提案: ${issue.suggestion}`);
      }
    });
  }
  
  return review;
}
```

### 2\. 使用方法の実例

```typescript:src/examples/usage-examples.ts
import { CachedGemmaClient } from '../services/cached-gemma-client';
import { IntelligentCodeReviewer } from './code-review-tool';

/**
 * 基本的な使用例
 */
export async function basicUsageExamples() {
  const client = new CachedGemmaClient();
  
  console.log('🚀 Gemma基本使用例');
  
  // 1. テキスト生成
  console.log('\n📝 テキスト生成:');
  const textResult = await client.generateTextCached(
    'TypeScriptの利点を3つ、初心者にもわかりやすく説明してください',
    '9b',
    { temperature: 0.7, maxTokens: 500 }
  );
  console.log(textResult.text);
  
  // 2. コード生成
  console.log('\n💻 コード生成:');
  const codeResult = await client.generateCodeCached(
    'Express.jsでRESTful APIを作成する基本的なコード',
    'typescript',
    '7b'
  );
  console.log(codeResult.text);
  
  // 3. バッチ処理
  console.log('\n📦 バッチ処理:');
  const batchPrompts = [
    '関数型プログラミングとは何ですか？',
    'リアクティブプログラミングの特徴は？',
    'マイクロサービスアーキテクチャのメリットは？'
  ];
  
  const batchResults = await client.generateBatch(batchPrompts, '9b', {
    temperature: 0.8,
    maxTokens: 300
  });
  
  batchResults.forEach((result, index) => {
    console.log(`\n質問 ${index + 1}: ${batchPrompts[index]}`);
    console.log(`回答: ${result.text.substring(0, 200)}...`);
  });
  
  await client.disconnect();
}

/**
 * プロダクション運用例
 */
export async function productionUsageExample() {
  const client = new CachedGemmaClient();
  const reviewer = new IntelligentCodeReviewer();
  
  console.log('🏭 プロダクション運用例');
  
  // 実際のプロジェクトファイルを想定
  const projectFiles = [
    {
      filename: 'src/api/users.ts',
      content: `
export async function getUser(id: string) {
  const user = await db.users.findById(id);
  return user;
}

export async function createUser(data: any) {
  const user = await db.users.create(data);
  return user;
}
      `,
      language: 'typescript'
    },
    {
      filename: 'src/utils/validation.ts',
      content: `
export function validateEmail(email) {
  return email.includes('@');
}

export function hashPassword(password) {
  return password + 'salt123';
}
      `,
      language: 'typescript'
    }
  ];
  
  console.log('\n🔍 プロジェクト全体のコードレビュー実行...');
  
  // 複数ファイルのレビュー
  const reviews = await reviewer.reviewMultipleFiles(projectFiles);
  
  // サマリーレポート生成
  const summary = await reviewer.generateProjectSummary(reviews);
  
  console.log('\n📊 プロジェクトサマリー:');
  console.log(`健全性: ${summary.overallHealth}`);
  console.log(`平均スコア: ${summary.averageScore}/10`);
  console.log(`総問題数: ${summary.totalIssues}`);
  console.log(`クリティカル問題: ${summary.criticalIssues}`);
  
  console.log('\n💡 推奨事項:');
  summary.recommendations.forEach((rec, index) => {
    console.log(`${index + 1}. ${rec}`);
  });
  
  console.log('\n🔝 頻出問題タイプ:');
  summary.topIssueTypes.forEach(({ type, count }) => {
    console.log(`${type}: ${count}件`);
  });
  
  // キャッシュ統計の確認
  const cacheStats = await client.getCacheStats();
  console.log('\n💾 キャッシュ統計:');
  console.log(`キー数: ${cacheStats.totalKeys}`);
  console.log(`メモリ使用量: ${cacheStats.memoryUsage}`);
  
  await client.disconnect();
}

/**
 * エラーハンドリング例
 */
export async function errorHandlingExample() {
  const client = new CachedGemmaClient();
  
  console.log('⚠️ エラーハンドリング例');
  
  try {
    // 意図的に長すぎるプロンプトで失敗させる
    const longPrompt = 'A'.repeat(100000);
    await client.generateText(longPrompt);
  } catch (error) {
    console.log('❌ 期待されるエラーをキャッチ:', error.message);
  }
  
  // リトライ機能付きの実行例
  async function generateWithRetry(
    prompt: string,
    maxRetries: number = 3
  ): Promise<string> {
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const result = await client.generateText(prompt);
        return result.text;
      } catch (error) {
        console.log(`試行 ${attempt} 失敗: ${error.message}`);
        
        if (attempt === maxRetries) {
          throw new Error(`${maxRetries}回の試行後も失敗: ${error.message}`);
        }
        
        // 指数バックオフで待機
        const delay = Math.pow(2, attempt - 1) * 1000;
        await new Promise(resolve => setTimeout(resolve, delay));
      }
    }
    
    return '';
  }
  
  try {
    const result = await generateWithRetry('簡単なテスト質問に答えてください');
    console.log('✅ リトライ成功:', result.substring(0, 100));
  } catch (error) {
    console.log('❌ 最終的な失敗:', error.message);
  }
  
  await client.disconnect();
}
```

## まとめ

本記事では、**GCP GemmaをTypeScriptで活用し、プロダクション対応のAIシステムを構築する方法**を包括的に解説しました。

**🎯 GCP Gemmaの主要メリット**

>* **オープンウェイト**: モデルカスタマイズとファインチューニングが可能
>* **多様なモデル**: 用途別に最適化された専用モデル群
>* **GCP統合**: スケーラブルで安定したインフラ基盤
>* **コスト効率**: オンプレミス運用も可能で柔軟な価格体系

**💡 実装のポイント**

>* **キャッシュ活用**: Redis統合で応答性能とコスト最適化
>* **バッチ処理**: 複数タスクの効率的な並列実行
>* **エラーハンドリング**: プロダクション運用に必要な堅牢性
>* **API設計**: REST APIでフロントエンドとの疎結合実現

**⚡ 実用的な応用例**

>* **インテリジェントコードレビュー**: 自動化されたコード品質チェック
>* **技術ドキュメント生成**: API仕様書やREADMEの自動作成
>* **多言語対応サポート**: 国際化プロジェクトでの翻訳支援
>* **コンテンツ管理**: ブログやマーケティング素材の生成

TypeScriptでGemmaを活用することで、**オープンソースAIの柔軟性とプロダクション運用の安定性**を両立できます。ぜひあなたのプロジェクトでも活用してみてください！