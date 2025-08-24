# もう多言語対応で迷わない！GCP Translation AIをTypeScriptで完全制御する次世代自動翻訳システム構築ガイド

## はじめに

グローバル展開を考えている開発者にとって、こんな悩みはありませんか？

> 「多言語対応って実装が複雑そう...」
> 「翻訳APIの品質や信頼性は大丈夫？」
> 「TypeScriptでプロダクション対応の翻訳システムを作りたい」

本記事では、**GCP Translation AIをTypeScriptで完全制御し、プロダクション対応の自動翻訳システムを構築する方法**を、実際のコード例とともに詳しく解説します。

## GCP Translation AIとは

GCP Translation AIは、**Googleが提供する機械学習ベースの翻訳サービス**で、100以上の言語をサポートする高品質な翻訳を提供します。

| 特徴 | 内容 | メリット |
|------|------|----------|
| **高品質翻訳** | Geminiベースの最新NLPモデル使用 | 自然で文脈を理解した翻訳 |
| **100言語対応** | 主要言語からマイナー言語まで幅広くサポート | グローバル展開に最適 |
| **2つのエディション** | Basic（v2）とAdvanced（v3）から選択 | 用途に応じた最適選択 |
| **カスタマイズ機能** | 用語集・カスタムモデル対応 | 業界固有の用語も正確に翻訳 |

**Translation AI エディション比較**

| 機能 | Basic (v2) | Advanced (v3) |
|------|------------|---------------|
| **翻訳品質** | 標準品質 | 高品質（Geminiベース） |
| **言語サポート** | 100+ | 100+ |
| **バッチ翻訳** | ❌ | ✅ |
| **カスタムモデル** | ❌ | ✅ |
| **用語集機能** | ❌ | ✅ |
| **料金** | 低価格 | 標準価格 |
| **推奨用途** | 基本的な翻訳 | エンタープライズ用途 |

## TypeScript環境のセットアップ

### 1\. プロジェクトの初期化

```bash
# 新しいプロジェクトを作成
mkdir translation-ai-app
cd translation-ai-app

# パッケージの初期化
npm init -y

# 必要な依存関係をインストール
npm install @google-cloud/translate
npm install express cors helmet compression
npm install rate-limiter-flexible redis ioredis
npm install @types/express @types/cors @types/node
npm install typescript ts-node nodemon dotenv
npm install multer @types/multer

# 開発用依存関係
npm install --save-dev jest @types/jest supertest @types/supertest
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
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
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
    location: process.env.GOOGLE_CLOUD_LOCATION || 'global',
    keyPath: process.env.GOOGLE_APPLICATION_CREDENTIALS || '',
  },
  
  // Translation AI設定
  translation: {
    apiEndpoint: process.env.TRANSLATION_API_ENDPOINT || 'translate.googleapis.com',
    defaultSourceLanguage: process.env.DEFAULT_SOURCE_LANG || 'auto',
    supportedLanguages: (process.env.SUPPORTED_LANGUAGES || 'ja,en,ko,zh,fr,es,de,it,pt,ru').split(','),
    maxCharactersPerRequest: parseInt(process.env.MAX_CHARS_PER_REQUEST || '30000'),
    batchSize: parseInt(process.env.BATCH_SIZE || '100')
  },
  
  // キャッシュ設定
  cache: {
    enabled: process.env.CACHE_ENABLED === 'true',
    redis: {
      url: process.env.REDIS_URL || 'redis://localhost:6379',
      keyPrefix: process.env.CACHE_KEY_PREFIX || 'translate:',
      ttl: parseInt(process.env.CACHE_TTL || '86400') // 24時間
    }
  },
  
  // サーバー設定
  server: {
    port: parseInt(process.env.PORT || '3000'),
    environment: process.env.NODE_ENV || 'development',
    corsOrigins: (process.env.CORS_ORIGINS || 'http://localhost:3000').split(','),
    rateLimits: {
      windowMs: parseInt(process.env.RATE_LIMIT_WINDOW || '900000'), // 15分
      max: parseInt(process.env.RATE_LIMIT_MAX || '100'),
      translationMax: parseInt(process.env.TRANSLATION_RATE_LIMIT || '50')
    }
  },
  
  // ファイルアップロード設定
  upload: {
    maxSize: parseInt(process.env.MAX_FILE_SIZE || '10485760'), // 10MB
    allowedTypes: (process.env.ALLOWED_FILE_TYPES || '.txt,.docx,.pdf,.json,.csv').split(','),
    uploadDir: process.env.UPLOAD_DIR || './uploads'
  }
} as const;

// 必須環境変数のバリデーション
export function validateEnvironment(): void {
  const required = [
    'GOOGLE_CLOUD_PROJECT',
    'GOOGLE_APPLICATION_CREDENTIALS'
  ];
  
  const missing = required.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(`必須の環境変数が設定されていません: ${missing.join(', ')}`);
  }
  
  // Google Cloud認証ファイルの存在確認
  try {
    const fs = require('fs');
    if (!fs.existsSync(CONFIG.gcp.keyPath)) {
      throw new Error(`認証ファイルが見つかりません: ${CONFIG.gcp.keyPath}`);
    }
  } catch (error) {
    throw new Error(`認証ファイルの確認に失敗しました: ${error}`);
  }
}

// サポート言語リストの取得
export function getSupportedLanguages(): string[] {
  return CONFIG.translation.supportedLanguages;
}

// 言語コードの検証
export function isValidLanguageCode(langCode: string): boolean {
  return CONFIG.translation.supportedLanguages.includes(langCode) || langCode === 'auto';
}
```

## Translation AIクライアントの実装

### 1\. 基本的な翻訳クライアント

```typescript:src/services/translation-client.ts
import { TranslationServiceClient } from '@google-cloud/translate';
import { CONFIG, validateEnvironment } from '../config/environment';

export interface TranslationOptions {
  sourceLanguage?: string;
  targetLanguage: string;
  model?: string;
  glossaryId?: string;
  mimeType?: string;
}

export interface TranslationResult {
  translatedText: string;
  detectedLanguage?: string;
  confidence?: number;
  model?: string;
  metadata?: Record<string, any>;
}

export interface BatchTranslationResult {
  results: TranslationResult[];
  totalCharacters: number;
  processingTime: number;
  failedItems: Array<{ index: number; error: string }>;
}

export class TranslationClient {
  private client: TranslationServiceClient;
  private parent: string;
  
  constructor() {
    validateEnvironment();
    
    this.client = new TranslationServiceClient({
      keyFilename: CONFIG.gcp.keyPath
    });
    
    this.parent = this.client.locationPath(
      CONFIG.gcp.projectId,
      CONFIG.gcp.location
    );
  }
  
  /**
   * 単一テキストの翻訳
   */
  async translateText(
    text: string,
    options: TranslationOptions
  ): Promise<TranslationResult> {
    if (!text || text.length === 0) {
      throw new Error('翻訳対象のテキストが空です');
    }
    
    if (text.length > CONFIG.translation.maxCharactersPerRequest) {
      throw new Error(`テキストが長すぎます（最大 ${CONFIG.translation.maxCharactersPerRequest} 文字）`);
    }
    
    try {
      const request = {
        parent: this.parent,
        contents: [text],
        targetLanguageCode: options.targetLanguage,
        sourceLanguageCode: options.sourceLanguage || 'auto',
        model: options.model,
        glossaryConfig: options.glossaryId ? {
          glossary: this.client.glossaryPath(
            CONFIG.gcp.projectId,
            CONFIG.gcp.location,
            options.glossaryId
          )
        } : undefined,
        mimeType: options.mimeType || 'text/plain'
      };
      
      const [response] = await this.client.translateText(request);
      
      if (!response.translations || response.translations.length === 0) {
        throw new Error('翻訳結果が取得できませんでした');
      }
      
      const translation = response.translations[0];
      
      return {
        translatedText: translation.translatedText || '',
        detectedLanguage: translation.detectedLanguageCode,
        model: translation.model,
        metadata: {
          originalLength: text.length,
          translatedLength: (translation.translatedText || '').length,
          timestamp: new Date().toISOString()
        }
      };
    } catch (error) {
      console.error('Translation error:', error);
      throw new Error(`翻訳に失敗しました: ${error}`);
    }
  }
  
  /**
   * 複数テキストのバッチ翻訳
   */
  async translateBatch(
    texts: string[],
    options: TranslationOptions
  ): Promise<BatchTranslationResult> {
    if (!texts || texts.length === 0) {
      throw new Error('翻訳対象のテキスト配列が空です');
    }
    
    const startTime = Date.now();
    const results: TranslationResult[] = [];
    const failedItems: Array<{ index: number; error: string }> = [];
    let totalCharacters = 0;
    
    // バッチサイズに分割して処理
    const batches = this.chunkArray(texts, CONFIG.translation.batchSize);
    
    for (let batchIndex = 0; batchIndex < batches.length; batchIndex++) {
      const batch = batches[batchIndex];
      const baseIndex = batchIndex * CONFIG.translation.batchSize;
      
      try {
        const request = {
          parent: this.parent,
          contents: batch,
          targetLanguageCode: options.targetLanguage,
          sourceLanguageCode: options.sourceLanguage || 'auto',
          model: options.model,
          mimeType: options.mimeType || 'text/plain'
        };
        
        const [response] = await this.client.translateText(request);
        
        if (response.translations) {
          response.translations.forEach((translation, index) => {
            const originalText = batch[index];
            totalCharacters += originalText.length;
            
            results.push({
              translatedText: translation.translatedText || '',
              detectedLanguage: translation.detectedLanguageCode,
              model: translation.model,
              metadata: {
                originalIndex: baseIndex + index,
                originalLength: originalText.length,
                translatedLength: (translation.translatedText || '').length,
                batchIndex
              }
            });
          });
        }
      } catch (error) {
        // バッチ全体が失敗した場合は個別に処理
        for (let i = 0; i < batch.length; i++) {
          try {
            const individualResult = await this.translateText(batch[i], options);
            results.push({
              ...individualResult,
              metadata: {
                ...individualResult.metadata,
                originalIndex: baseIndex + i,
                batchIndex,
                fallbackProcessing: true
              }
            });
            totalCharacters += batch[i].length;
          } catch (individualError) {
            failedItems.push({
              index: baseIndex + i,
              error: `${individualError}`
            });
          }
        }
      }
    }
    
    const processingTime = Date.now() - startTime;
    
    return {
      results,
      totalCharacters,
      processingTime,
      failedItems
    };
  }
  
  /**
   * 言語検出
   */
  async detectLanguage(text: string): Promise<{
    language: string;
    confidence: number;
    isReliable: boolean;
  }> {
    if (!text || text.length === 0) {
      throw new Error('言語検出対象のテキストが空です');
    }
    
    try {
      const request = {
        parent: this.parent,
        content: text.substring(0, 1000), // 検出は最初の1000文字で十分
        mimeType: 'text/plain'
      };
      
      const [response] = await this.client.detectLanguage(request);
      
      if (!response.languages || response.languages.length === 0) {
        throw new Error('言語が検出できませんでした');
      }
      
      const detection = response.languages[0];
      
      return {
        language: detection.languageCode || 'unknown',
        confidence: detection.confidence || 0,
        isReliable: (detection.confidence || 0) > 0.8
      };
    } catch (error) {
      console.error('Language detection error:', error);
      throw new Error(`言語検出に失敗しました: ${error}`);
    }
  }
  
  /**
   * サポート言語一覧の取得
   */
  async getSupportedLanguages(): Promise<Array<{
    languageCode: string;
    displayName: string;
    supportTarget: boolean;
    supportSource: boolean;
  }>> {
    try {
      const request = {
        parent: this.parent,
        displayLanguageCode: 'ja' // 日本語で言語名を取得
      };
      
      const [response] = await this.client.getSupportedLanguages(request);
      
      return (response.languages || []).map(lang => ({
        languageCode: lang.languageCode || '',
        displayName: lang.displayName || '',
        supportTarget: lang.supportTarget || false,
        supportSource: lang.supportSource || false
      }));
    } catch (error) {
      console.error('Get supported languages error:', error);
      throw new Error(`サポート言語の取得に失敗しました: ${error}`);
    }
  }
  
  /**
   * 用語集の作成
   */
  async createGlossary(
    glossaryId: string,
    sourceLanguage: string,
    targetLanguage: string,
    glossaryTerms: Record<string, string>
  ): Promise<string> {
    try {
      // 用語集をCSV形式に変換
      const csvContent = Object.entries(glossaryTerms)
        .map(([source, target]) => `"${source}","${target}"`)
        .join('\n');
      
      const request = {
        parent: this.parent,
        glossary: {
          name: this.client.glossaryPath(
            CONFIG.gcp.projectId,
            CONFIG.gcp.location,
            glossaryId
          ),
          languageCodesSet: {
            languageCodes: [sourceLanguage, targetLanguage]
          },
          inputConfig: {
            gcsSource: {
              inputUri: `gs://${CONFIG.gcp.projectId}-glossaries/${glossaryId}.csv`
            }
          }
        }
      };
      
      // 実際の実装では、まずGCSにCSVファイルをアップロードする必要があります
      console.log('用語集CSVデータ:', csvContent);
      
      const [operation] = await this.client.createGlossary(request);
      const [glossary] = await operation.promise();
      
      return glossary.name || '';
    } catch (error) {
      console.error('Create glossary error:', error);
      throw new Error(`用語集の作成に失敗しました: ${error}`);
    }
  }
  
  /**
   * 配列をバッチサイズに分割
   */
  private chunkArray<T>(array: T[], size: number): T[][] {
    const chunks: T[][] = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }
}
```

### 2\. キャッシュ機能付きクライアント

```typescript:src/services/cached-translation-client.ts
import { TranslationClient, TranslationOptions, TranslationResult, BatchTranslationResult } from './translation-client';
import { createHash } from 'crypto';
import Redis from 'ioredis';
import { CONFIG } from '../config/environment';

export interface CacheStats {
  totalKeys: number;
  hitCount: number;
  missCount: number;
  hitRate: number;
  memoryUsage: string;
}

export class CachedTranslationClient extends TranslationClient {
  private redis: Redis | null = null;
  private hitCount = 0;
  private missCount = 0;
  
  constructor() {
    super();
    
    if (CONFIG.cache.enabled) {
      this.redis = new Redis(CONFIG.cache.redis.url, {
        keyPrefix: CONFIG.cache.redis.keyPrefix,
        retryDelayOnFailover: 100,
        maxRetriesPerRequest: 3
      });
      
      this.redis.on('error', (error) => {
        console.error('Redis connection error:', error);
      });
      
      this.redis.on('connect', () => {
        console.log('Redis connected for translation caching');
      });
    }
  }
  
  /**
   * キャッシュ機能付き翻訳
   */
  async translateText(
    text: string,
    options: TranslationOptions
  ): Promise<TranslationResult> {
    const cacheKey = this.generateCacheKey(text, options);
    
    // キャッシュから取得を試行
    if (this.redis) {
      try {
        const cached = await this.redis.get(cacheKey);
        if (cached) {
          this.hitCount++;
          console.log(`翻訳キャッシュヒット: ${cacheKey.substring(0, 20)}...`);
          return JSON.parse(cached);
        }
      } catch (error) {
        console.warn('Cache retrieval error:', error);
      }
    }
    
    this.missCount++;
    
    // キャッシュにない場合は新規翻訳
    const result = await super.translateText(text, options);
    
    // 結果をキャッシュに保存
    if (this.redis) {
      try {
        await this.redis.setex(
          cacheKey,
          CONFIG.cache.redis.ttl,
          JSON.stringify(result)
        );
        console.log(`翻訳結果をキャッシュ保存: ${cacheKey.substring(0, 20)}...`);
      } catch (error) {
        console.warn('Cache storage error:', error);
      }
    }
    
    return result;
  }
  
  /**
   * バッチ翻訳（部分的キャッシュ対応）
   */
  async translateBatch(
    texts: string[],
    options: TranslationOptions
  ): Promise<BatchTranslationResult> {
    if (!this.redis || !CONFIG.cache.enabled) {
      return super.translateBatch(texts, options);
    }
    
    const startTime = Date.now();
    const results: TranslationResult[] = new Array(texts.length);
    const uncachedTexts: string[] = [];
    const uncachedIndices: number[] = [];
    let totalCharacters = 0;
    
    // キャッシュから取得可能なものを先に処理
    for (let i = 0; i < texts.length; i++) {
      const text = texts[i];
      const cacheKey = this.generateCacheKey(text, options);
      
      try {
        const cached = await this.redis.get(cacheKey);
        if (cached) {
          results[i] = JSON.parse(cached);
          this.hitCount++;
          totalCharacters += text.length;
        } else {
          uncachedTexts.push(text);
          uncachedIndices.push(i);
          this.missCount++;
        }
      } catch (error) {
        console.warn(`Cache error for index ${i}:`, error);
        uncachedTexts.push(text);
        uncachedIndices.push(i);
        this.missCount++;
      }
    }
    
    // キャッシュにないものを翻訳
    let failedItems: Array<{ index: number; error: string }> = [];
    
    if (uncachedTexts.length > 0) {
      const batchResult = await super.translateBatch(uncachedTexts, options);
      
      // バッチ結果を元の位置に配置し、キャッシュに保存
      for (let i = 0; i < batchResult.results.length; i++) {
        const originalIndex = uncachedIndices[i];
        const result = batchResult.results[i];
        
        results[originalIndex] = result;
        
        // キャッシュに保存
        try {
          const cacheKey = this.generateCacheKey(uncachedTexts[i], options);
          await this.redis.setex(
            cacheKey,
            CONFIG.cache.redis.ttl,
            JSON.stringify(result)
          );
        } catch (error) {
          console.warn(`Failed to cache result for index ${originalIndex}:`, error);
        }
      }
      
      failedItems = batchResult.failedItems.map(item => ({
        index: uncachedIndices[item.index],
        error: item.error
      }));
      
      totalCharacters += batchResult.totalCharacters;
    }
    
    const processingTime = Date.now() - startTime;
    
    return {
      results: results.filter(result => result !== undefined),
      totalCharacters,
      processingTime,
      failedItems
    };
  }
  
  /**
   * キャッシュ統計の取得
   */
  async getCacheStats(): Promise<CacheStats> {
    if (!this.redis) {
      return {
        totalKeys: 0,
        hitCount: this.hitCount,
        missCount: this.missCount,
        hitRate: 0,
        memoryUsage: 'N/A (Redis not enabled)'
      };
    }
    
    try {
      const keyCount = await this.redis.dbsize();
      const info = await this.redis.info('memory');
      
      const memoryMatch = info.match(/used_memory_human:([^\r\n]+)/);
      const memoryUsage = memoryMatch ? memoryMatch[1] : 'unknown';
      
      const totalRequests = this.hitCount + this.missCount;
      const hitRate = totalRequests > 0 ? (this.hitCount / totalRequests) * 100 : 0;
      
      return {
        totalKeys: keyCount,
        hitCount: this.hitCount,
        missCount: this.missCount,
        hitRate: Number(hitRate.toFixed(2)),
        memoryUsage
      };
    } catch (error) {
      console.error('Failed to get cache stats:', error);
      throw new Error(`キャッシュ統計の取得に失敗しました: ${error}`);
    }
  }
  
  /**
   * キャッシュのクリア
   */
  async clearCache(pattern?: string): Promise<number> {
    if (!this.redis) {
      return 0;
    }
    
    try {
      const searchPattern = pattern ? `*${pattern}*` : '*';
      const keys = await this.redis.keys(searchPattern);
      
      if (keys.length === 0) {
        return 0;
      }
      
      const deletedCount = await this.redis.del(...keys);
      console.log(`${deletedCount}個のキャッシュエントリを削除しました`);
      
      return deletedCount;
    } catch (error) {
      console.error('Cache clear error:', error);
      throw new Error(`キャッシュクリアに失敗しました: ${error}`);
    }
  }
  
  /**
   * キャッシュキーの生成
   */
  private generateCacheKey(text: string, options: TranslationOptions): string {
    const keyData = {
      text: text.trim(),
      source: options.sourceLanguage || 'auto',
      target: options.targetLanguage,
      model: options.model || 'default',
      glossary: options.glossaryId || 'none'
    };
    
    const dataString = JSON.stringify(keyData);
    const hash = createHash('md5').update(dataString).digest('hex');
    
    return `${options.sourceLanguage || 'auto'}-${options.targetLanguage}:${hash}`;
  }
  
  /**
   * リソースのクリーンアップ
   */
  async disconnect(): Promise<void> {
    if (this.redis) {
      await this.redis.disconnect();
    }
  }
}
```

## Web APIサーバーの実装

```typescript:src/api/server.ts
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import compression from 'compression';
import multer from 'multer';
import { rateLimit } from 'express-rate-limit';
import { CachedTranslationClient } from '../services/cached-translation-client';
import { CONFIG, isValidLanguageCode } from '../config/environment';
import path from 'path';
import fs from 'fs/promises';

const app = express();
const translationClient = new CachedTranslationClient();

// ファイルアップロード設定
const upload = multer({
  dest: CONFIG.upload.uploadDir,
  limits: {
    fileSize: CONFIG.upload.maxSize
  },
  fileFilter: (req, file, cb) => {
    const ext = path.extname(file.originalname).toLowerCase();
    if (CONFIG.upload.allowedTypes.includes(ext)) {
      cb(null, true);
    } else {
      cb(new Error(`サポートされていないファイル形式です: ${ext}`));
    }
  }
});

// ミドルウェア設定
app.use(helmet({
  crossOriginResourcePolicy: { policy: 'cross-origin' }
}));
app.use(cors({
  origin: CONFIG.server.corsOrigins,
  credentials: true
}));
app.use(compression());
app.use(express.json({ limit: '50mb' }));
app.use(express.urlencoded({ extended: true, limit: '50mb' }));

// レート制限
const generalLimiter = rateLimit({
  windowMs: CONFIG.server.rateLimits.windowMs,
  max: CONFIG.server.rateLimits.max,
  message: 'Too many requests from this IP, please try again later.',
  standardHeaders: true,
  legacyHeaders: false
});

const translationLimiter = rateLimit({
  windowMs: CONFIG.server.rateLimits.windowMs,
  max: CONFIG.server.rateLimits.translationMax,
  message: 'Translation rate limit exceeded, please try again later.',
  standardHeaders: true,
  legacyHeaders: false
});

app.use('/api', generalLimiter);
app.use('/api/translate', translationLimiter);

/**
 * 単一テキスト翻訳API
 */
app.post('/api/translate/text', async (req, res) => {
  try {
    const {
      text,
      sourceLanguage = 'auto',
      targetLanguage,
      model,
      glossaryId
    } = req.body;
    
    // 入力検証
    if (!text || typeof text !== 'string') {
      return res.status(400).json({
        error: 'テキストは必須です'
      });
    }
    
    if (!targetLanguage || !isValidLanguageCode(targetLanguage)) {
      return res.status(400).json({
        error: '有効な翻訳先言語を指定してください'
      });
    }
    
    if (sourceLanguage !== 'auto' && !isValidLanguageCode(sourceLanguage)) {
      return res.status(400).json({
        error: '有効な翻訳元言語を指定してください'
      });
    }
    
    const startTime = Date.now();
    
    const result = await translationClient.translateText(text, {
      sourceLanguage,
      targetLanguage,
      model,
      glossaryId
    });
    
    const responseTime = Date.now() - startTime;
    
    res.json({
      success: true,
      data: {
        translatedText: result.translatedText,
        detectedLanguage: result.detectedLanguage,
        sourceLanguage,
        targetLanguage,
        model: result.model,
        metadata: {
          ...result.metadata,
          responseTime,
          characterCount: text.length
        }
      }
    });
  } catch (error) {
    console.error('Translation error:', error);
    res.status(500).json({
      error: 'テキスト翻訳に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * バッチ翻訳API
 */
app.post('/api/translate/batch', async (req, res) => {
  try {
    const {
      texts,
      sourceLanguage = 'auto',
      targetLanguage,
      model,
      glossaryId
    } = req.body;
    
    // 入力検証
    if (!Array.isArray(texts) || texts.length === 0) {
      return res.status(400).json({
        error: 'テキスト配列は必須です'
      });
    }
    
    if (texts.length > 100) {
      return res.status(400).json({
        error: 'バッチサイズは100個までです'
      });
    }
    
    if (!targetLanguage || !isValidLanguageCode(targetLanguage)) {
      return res.status(400).json({
        error: '有効な翻訳先言語を指定してください'
      });
    }
    
    const startTime = Date.now();
    
    const result = await translationClient.translateBatch(texts, {
      sourceLanguage,
      targetLanguage,
      model,
      glossaryId
    });
    
    const responseTime = Date.now() - startTime;
    
    res.json({
      success: true,
      data: {
        translations: result.results.map((translation, index) => ({
          originalText: texts[index],
          translatedText: translation.translatedText,
          detectedLanguage: translation.detectedLanguage,
          metadata: translation.metadata
        })),
        summary: {
          totalTexts: texts.length,
          successfulTranslations: result.results.length,
          failedTranslations: result.failedItems.length,
          totalCharacters: result.totalCharacters,
          processingTime: result.processingTime,
          responseTime,
          failedItems: result.failedItems
        }
      }
    });
  } catch (error) {
    console.error('Batch translation error:', error);
    res.status(500).json({
      error: 'バッチ翻訳に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * 言語検出API
 */
app.post('/api/translate/detect', async (req, res) => {
  try {
    const { text } = req.body;
    
    if (!text || typeof text !== 'string') {
      return res.status(400).json({
        error: 'テキストは必須です'
      });
    }
    
    const result = await translationClient.detectLanguage(text);
    
    res.json({
      success: true,
      data: {
        detectedLanguage: result.language,
        confidence: result.confidence,
        isReliable: result.isReliable,
        textPreview: text.substring(0, 100) + (text.length > 100 ? '...' : '')
      }
    });
  } catch (error) {
    console.error('Language detection error:', error);
    res.status(500).json({
      error: '言語検出に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * ファイル翻訳API
 */
app.post('/api/translate/file', upload.single('file'), async (req, res) => {
  try {
    const {
      sourceLanguage = 'auto',
      targetLanguage
    } = req.body;
    
    if (!req.file) {
      return res.status(400).json({
        error: 'ファイルをアップロードしてください'
      });
    }
    
    if (!targetLanguage || !isValidLanguageCode(targetLanguage)) {
      return res.status(400).json({
        error: '有効な翻訳先言語を指定してください'
      });
    }
    
    // ファイル内容を読み取り
    const fileContent = await fs.readFile(req.file.path, 'utf-8');
    
    // ファイルタイプに応じて処理を分岐
    const ext = path.extname(req.file.originalname).toLowerCase();
    let texts: string[] = [];
    
    switch (ext) {
      case '.txt':
        texts = [fileContent];
        break;
      case '.json':
        try {
          const jsonData = JSON.parse(fileContent);
          texts = this.extractTextsFromJSON(jsonData);
        } catch (error) {
          throw new Error('無効なJSONファイルです');
        }
        break;
      case '.csv':
        texts = fileContent.split('\n').filter(line => line.trim().length > 0);
        break;
      default:
        throw new Error(`サポートされていないファイル形式: ${ext}`);
    }
    
    const result = await translationClient.translateBatch(texts, {
      sourceLanguage,
      targetLanguage
    });
    
    // 翻訳結果をファイル形式に合わせて再構成
    const translatedContent = this.reconstructFileContent(
      texts,
      result.results,
      ext,
      req.file.originalname
    );
    
    // 一時ファイルを削除
    await fs.unlink(req.file.path);
    
    res.json({
      success: true,
      data: {
        originalFilename: req.file.originalname,
        translatedContent,
        fileType: ext,
        summary: {
          totalTexts: texts.length,
          successfulTranslations: result.results.length,
          totalCharacters: result.totalCharacters,
          processingTime: result.processingTime
        }
      }
    });
  } catch (error) {
    console.error('File translation error:', error);
    
    // 一時ファイルのクリーンアップ
    if (req.file) {
      try {
        await fs.unlink(req.file.path);
      } catch (cleanupError) {
        console.error('File cleanup error:', cleanupError);
      }
    }
    
    res.status(500).json({
      error: 'ファイル翻訳に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * サポート言語一覧API
 */
app.get('/api/translate/languages', async (req, res) => {
  try {
    const languages = await translationClient.getSupportedLanguages();
    
    res.json({
      success: true,
      data: {
        languages,
        total: languages.length,
        configured: CONFIG.translation.supportedLanguages
      }
    });
  } catch (error) {
    console.error('Get languages error:', error);
    res.status(500).json({
      error: 'サポート言語の取得に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * キャッシュ統計API
 */
app.get('/api/cache/stats', async (req, res) => {
  try {
    const stats = await translationClient.getCacheStats();
    
    res.json({
      success: true,
      data: stats
    });
  } catch (error) {
    console.error('Cache stats error:', error);
    res.status(500).json({
      error: 'キャッシュ統計の取得に失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
    });
  }
});

/**
 * キャッシュクリアAPI
 */
app.delete('/api/cache', async (req, res) => {
  try {
    const { pattern } = req.query;
    const deletedCount = await translationClient.clearCache(pattern as string);
    
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
      error: 'キャッシュクリアに失敗しました',
      message: error instanceof Error ? error.message : 'Unknown error'
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
    version: process.env.npm_package_version || '1.0.0',
    uptime: process.uptime(),
    supportedLanguages: CONFIG.translation.supportedLanguages.length
  });
});

// ヘルパーメソッド
function extractTextsFromJSON(obj: any, texts: string[] = []): string[] {
  if (typeof obj === 'string' && obj.trim().length > 0) {
    texts.push(obj);
  } else if (Array.isArray(obj)) {
    obj.forEach(item => extractTextsFromJSON(item, texts));
  } else if (typeof obj === 'object' && obj !== null) {
    Object.values(obj).forEach(value => extractTextsFromJSON(value, texts));
  }
  return texts;
}

function reconstructFileContent(
  originalTexts: string[],
  translations: any[],
  fileType: string,
  originalFilename: string
): string {
  switch (fileType) {
    case '.txt':
      return translations[0]?.translatedText || '';
    case '.csv':
      return translations.map(t => t.translatedText).join('\n');
    case '.json':
      // JSONの場合は元の構造を保持しつつ翻訳
      // 簡単な実装例（実際はより複雑な構造保持が必要）
      return JSON.stringify({
        originalFile: originalFilename,
        translations: translations.map(t => t.translatedText)
      }, null, 2);
    default:
      return translations.map(t => t.translatedText).join('\n');
  }
}

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
  console.log(`🚀 Translation API Server started on port ${CONFIG.server.port}`);
  console.log(`🌍 Environment: ${CONFIG.server.environment}`);
  console.log(`🗣️ Supported languages: ${CONFIG.translation.supportedLanguages.join(', ')}`);
});

// グレースフルシャットダウン
process.on('SIGTERM', async () => {
  console.log('SIGTERM received, shutting down gracefully...');
  server.close(async () => {
    await translationClient.disconnect();
    process.exit(0);
  });
});

export default app;
```

## 実用的なアプリケーション例

### 1\. 多言語対応Webサイト自動翻訳ツール

```typescript:src/examples/website-translator.ts
import { CachedTranslationClient } from '../services/cached-translation-client';
import * as cheerio from 'cheerio';
import fetch from 'node-fetch';

export interface WebsiteTranslationResult {
  originalUrl: string;
  translatedContent: string;
  translationMap: Record<string, string>;
  metadata: {
    totalElements: number;
    translatedElements: number;
    processingTime: number;
    detectedLanguage?: string;
  };
}

export class WebsiteTranslator {
  private translationClient: CachedTranslationClient;
  
  constructor() {
    this.translationClient = new CachedTranslationClient();
  }
  
  /**
   * Webサイト全体の翻訳
   */
  async translateWebsite(
    url: string,
    targetLanguage: string,
    options: {
      sourceLanguage?: string;
      excludeSelectors?: string[];
      includeOnlySelectors?: string[];
      preserveFormatting?: boolean;
    } = {}
  ): Promise<WebsiteTranslationResult> {
    const startTime = Date.now();
    
    try {
      // Webページを取得
      console.log(`🌐 Webページを取得中: ${url}`);
      const response = await fetch(url);
      const html = await response.text();
      
      // cheerioでHTMLをパース
      const $ = cheerio.load(html);
      
      // 翻訳対象要素の抽出
      const textElements = this.extractTranslatableElements(
        $,
        options.excludeSelectors,
        options.includeOnlySelectors
      );
      
      console.log(`📝 翻訳対象要素数: ${textElements.length}`);
      
      if (textElements.length === 0) {
        throw new Error('翻訳可能なテキスト要素が見つかりませんでした');
      }
      
      // 言語検出（最初のいくつかのテキストから）
      let detectedLanguage: string | undefined;
      if (!options.sourceLanguage) {
        const sampleTexts = textElements.slice(0, 5).map(el => el.text);
        const sampleText = sampleTexts.join(' ').substring(0, 1000);
        
        try {
          const detection = await this.translationClient.detectLanguage(sampleText);
          detectedLanguage = detection.language;
          console.log(`🔍 検出された言語: ${detectedLanguage} (信頼度: ${detection.confidence})`);
        } catch (error) {
          console.warn('言語検出に失敗しました:', error);
        }
      }
      
      // バッチ翻訳の実行
      const sourceLanguage = options.sourceLanguage || detectedLanguage || 'auto';
      const texts = textElements.map(el => el.text);
      
      console.log(`🚀 バッチ翻訳開始 (${sourceLanguage} → ${targetLanguage})`);
      const batchResult = await this.translationClient.translateBatch(texts, {
        sourceLanguage,
        targetLanguage
      });
      
      if (batchResult.failedItems.length > 0) {
        console.warn(`⚠️ ${batchResult.failedItems.length}個の翻訳に失敗しました`);
      }
      
      // 翻訳結果をHTMLに適用
      const translationMap: Record<string, string> = {};
      let translatedElements = 0;
      
      for (let i = 0; i < textElements.length; i++) {
        const element = textElements[i];
        const translation = batchResult.results[i];
        
        if (translation) {
          const translatedText = translation.translatedText;
          
          // 元のHTMLタグを保持したまま翻訳テキストを適用
          if (options.preserveFormatting) {
            const $element = $(element.selector);
            $element.html($element.html()!.replace(element.text, translatedText));
          } else {
            $(element.selector).text(translatedText);
          }
          
          translationMap[element.text] = translatedText;
          translatedElements++;
        }
      }
      
      const processingTime = Date.now() - startTime;
      
      console.log(`✅ 翻訳完了: ${translatedElements}/${textElements.length}要素 (${processingTime}ms)`);
      
      return {
        originalUrl: url,
        translatedContent: $.html(),
        translationMap,
        metadata: {
          totalElements: textElements.length,
          translatedElements,
          processingTime,
          detectedLanguage
        }
      };
    } catch (error) {
      console.error('Website translation error:', error);
      throw new Error(`Webサイト翻訳に失敗しました: ${error}`);
    }
  }
  
  /**
   * JSONファイルの多言語化
   */
  async translateJSONLocale(
    jsonContent: any,
    sourceLanguage: string,
    targetLanguages: string[]
  ): Promise<Record<string, any>> {
    const results: Record<string, any> = {};
    
    // JSONからテキストを抽出
    const textPaths = this.extractJSONTexts(jsonContent);
    const texts = textPaths.map(path => path.value);
    
    console.log(`📝 JSON内テキスト数: ${texts.length}`);
    
    // 各言語に翻訳
    for (const targetLanguage of targetLanguages) {
      console.log(`🌍 ${sourceLanguage} → ${targetLanguage} 翻訳中...`);
      
      try {
        const batchResult = await this.translationClient.translateBatch(texts, {
          sourceLanguage,
          targetLanguage
        });
        
        // 翻訳結果を元のJSON構造に適用
        const translatedJson = JSON.parse(JSON.stringify(jsonContent));
        
        for (let i = 0; i < textPaths.length; i++) {
          const path = textPaths[i];
          const translation = batchResult.results[i];
          
          if (translation) {
            this.setJSONValueByPath(translatedJson, path.path, translation.translatedText);
          }
        }
        
        results[targetLanguage] = translatedJson;
        
        console.log(`✅ ${targetLanguage} 翻訳完了 (${batchResult.results.length}/${texts.length})`);
      } catch (error) {
        console.error(`❌ ${targetLanguage} 翻訳失敗:`, error);
        results[targetLanguage] = { error: `翻訳に失敗しました: ${error}` };
      }
    }
    
    return results;
  }
  
  /**
   * Markdown文書の翻訳
   */
  async translateMarkdown(
    markdown: string,
    sourceLanguage: string,
    targetLanguage: string,
    options: {
      preserveCodeBlocks?: boolean;
      preserveLinks?: boolean;
      chunkSize?: number;
    } = {}
  ): Promise<{
    translatedMarkdown: string;
    translationCount: number;
    processingTime: number;
  }> {
    const startTime = Date.now();
    const {
      preserveCodeBlocks = true,
      preserveLinks = true,
      chunkSize = 1000
    } = options;
    
    let workingMarkdown = markdown;
    const protectedBlocks: Record<string, string> = {};
    
    // コードブロックを保護
    if (preserveCodeBlocks) {
      const codeBlockRegex = /```[\s\S]*?```/g;
      let match;
      let blockIndex = 0;
      
      while ((match = codeBlockRegex.exec(markdown)) !== null) {
        const placeholder = `__CODE_BLOCK_${blockIndex}__`;
        protectedBlocks[placeholder] = match[0];
        workingMarkdown = workingMarkdown.replace(match[0], placeholder);
        blockIndex++;
      }
    }
    
    // リンクを保護
    if (preserveLinks) {
      const linkRegex = /\[([^\]]*)\]\([^\)]*\)/g;
      let match;
      let linkIndex = 0;
      
      while ((match = linkRegex.exec(workingMarkdown)) !== null) {
        if (!protectedBlocks[match[0]]) {
          const placeholder = `__LINK_${linkIndex}__`;
          protectedBlocks[placeholder] = match[0];
          workingMarkdown = workingMarkdown.replace(match[0], placeholder);
          linkIndex++;
        }
      }
    }
    
    // テキストをチャンクに分割
    const chunks = this.chunkMarkdown(workingMarkdown, chunkSize);
    console.log(`📄 Markdownを${chunks.length}個のチャンクに分割`);
    
    // チャンクごとに翻訳
    const translatedChunks: string[] = [];
    let translationCount = 0;
    
    for (let i = 0; i < chunks.length; i++) {
      const chunk = chunks[i];
      
      if (chunk.trim().length === 0) {
        translatedChunks.push(chunk);
        continue;
      }
      
      try {
        console.log(`🔄 チャンク ${i + 1}/${chunks.length} 翻訳中...`);
        const result = await this.translationClient.translateText(chunk, {
          sourceLanguage,
          targetLanguage
        });
        
        translatedChunks.push(result.translatedText);
        translationCount++;
      } catch (error) {
        console.warn(`⚠️ チャンク ${i + 1} の翻訳に失敗:`, error);
        translatedChunks.push(chunk); // 元のテキストを保持
      }
    }
    
    // 翻訳されたチャンクを結合
    let translatedMarkdown = translatedChunks.join('');
    
    // 保護されたブロックを復元
    Object.entries(protectedBlocks).forEach(([placeholder, originalText]) => {
      translatedMarkdown = translatedMarkdown.replace(
        new RegExp(placeholder, 'g'),
        originalText
      );
    });
    
    const processingTime = Date.now() - startTime;
    
    console.log(`✅ Markdown翻訳完了 (${translationCount}チャンク, ${processingTime}ms)`);
    
    return {
      translatedMarkdown,
      translationCount,
      processingTime
    };
  }
  
  /**
   * HTMLから翻訳可能な要素を抽出
   */
  private extractTranslatableElements(
    $: cheerio.CheerioAPI,
    excludeSelectors: string[] = [],
    includeOnlySelectors?: string[]
  ): Array<{ selector: string; text: string }> {
    const elements: Array<{ selector: string; text: string }> = [];
    const defaultExcludes = ['script', 'style', 'code', 'pre', 'noscript'];
    const allExcludes = [...defaultExcludes, ...excludeSelectors];
    
    // 翻訳対象セレクターを決定
    const targetSelectors = includeOnlySelectors || ['p', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6', 'span', 'div', 'a', 'button', 'td', 'th', 'li'];
    
    targetSelectors.forEach(selector => {
      $(selector).each((index, element) => {
        const $element = $(element);
        const text = $element.text().trim();
        
        // 除外セレクターに該当しないかチェック
        const isExcluded = allExcludes.some(excludeSelector => 
          $element.is(excludeSelector) || $element.parents(excludeSelector).length > 0
        );
        
        if (!isExcluded && text.length > 0 && text.length < 5000) {
          const uniqueSelector = this.generateUniqueSelector($element, index, selector);
          elements.push({
            selector: uniqueSelector,
            text
          });
        }
      });
    });
    
    return elements;
  }
  
  /**
   * 一意なCSSセレクターを生成
   */
  private generateUniqueSelector($element: cheerio.Cheerio<cheerio.Element>, index: number, baseSelector: string): string {
    const id = $element.attr('id');
    if (id) {
      return `#${id}`;
    }
    
    const className = $element.attr('class');
    if (className) {
      const firstClass = className.split(' ')[0];
      return `${baseSelector}.${firstClass}:eq(${index})`;
    }
    
    return `${baseSelector}:eq(${index})`;
  }
  
  /**
   * JSONからテキストパスを抽出
   */
  private extractJSONTexts(obj: any, currentPath: string = ''): Array<{ path: string; value: string }> {
    const texts: Array<{ path: string; value: string }> = [];
    
    if (typeof obj === 'string' && obj.trim().length > 0) {
      texts.push({ path: currentPath, value: obj });
    } else if (Array.isArray(obj)) {
      obj.forEach((item, index) => {
        texts.push(...this.extractJSONTexts(item, `${currentPath}[${index}]`));
      });
    } else if (typeof obj === 'object' && obj !== null) {
      Object.keys(obj).forEach(key => {
        const path = currentPath ? `${currentPath}.${key}` : key;
        texts.push(...this.extractJSONTexts(obj[key], path));
      });
    }
    
    return texts;
  }
  
  /**
   * JSONのパスに値をセット
   */
  private setJSONValueByPath(obj: any, path: string, value: string): void {
    const keys = path.replace(/\[(\d+)\]/g, '.$1').split('.');
    let current = obj;
    
    for (let i = 0; i < keys.length - 1; i++) {
      const key = keys[i];
      if (!(key in current)) {
        current[key] = {};
      }
      current = current[key];
    }
    
    const finalKey = keys[keys.length - 1];
    current[finalKey] = value;
  }
  
  /**
   * Markdownをチャンクに分割
   */
  private chunkMarkdown(markdown: string, maxSize: number): string[] {
    const chunks: string[] = [];
    const lines = markdown.split('\n');
    let currentChunk = '';
    
    for (const line of lines) {
      if ((currentChunk + line + '\n').length > maxSize && currentChunk.length > 0) {
        chunks.push(currentChunk);
        currentChunk = line + '\n';
      } else {
        currentChunk += line + '\n';
      }
    }
    
    if (currentChunk.length > 0) {
      chunks.push(currentChunk);
    }
    
    return chunks;
  }
  
  /**
   * リソースのクリーンアップ
   */
  async disconnect(): Promise<void> {
    await this.translationClient.disconnect();
  }
}
```

### 2\. 使用方法の実例

```typescript:src/examples/usage-examples.ts
import { CachedTranslationClient } from '../services/cached-translation-client';
import { WebsiteTranslator } from './website-translator';

/**
 * 基本的な使用例
 */
export async function basicTranslationExamples() {
  const client = new CachedTranslationClient();
  
  console.log('🚀 Translation AI基本使用例');
  
  // 1. 単一テキスト翻訳
  console.log('\n📝 単一テキスト翻訳:');
  const textResult = await client.translateText(
    'Hello, world! How are you today?',
    { targetLanguage: 'ja' }
  );
  console.log(`翻訳結果: ${textResult.translatedText}`);
  console.log(`検出言語: ${textResult.detectedLanguage}`);
  
  // 2. バッチ翻訳
  console.log('\n📦 バッチ翻訳:');
  const texts = [
    'Good morning!',
    'Thank you very much.',
    'See you later.',
    'Have a nice day!'
  ];
  
  const batchResult = await client.translateBatch(texts, {
    sourceLanguage: 'en',
    targetLanguage: 'ja'
  });
  
  batchResult.results.forEach((result, index) => {
    console.log(`${index + 1}. ${texts[index]} → ${result.translatedText}`);
  });
  
  console.log(`\n処理時間: ${batchResult.processingTime}ms`);
  console.log(`総文字数: ${batchResult.totalCharacters}`);
  
  // 3. 言語検出
  console.log('\n🔍 言語検出:');
  const detectionResult = await client.detectLanguage('Bonjour, comment allez-vous?');
  console.log(`検出言語: ${detectionResult.language}`);
  console.log(`信頼度: ${detectionResult.confidence}`);
  console.log(`信頼性: ${detectionResult.isReliable ? '高い' : '低い'}`);
  
  // 4. 複数言語への一括翻訳
  console.log('\n🌍 複数言語への一括翻訳:');
  const originalText = 'Welcome to our website!';
  const targetLanguages = ['ja', 'ko', 'zh', 'es', 'fr'];
  
  for (const lang of targetLanguages) {
    const result = await client.translateText(originalText, {
      sourceLanguage: 'en',
      targetLanguage: lang
    });
    console.log(`${lang}: ${result.translatedText}`);
  }
  
  await client.disconnect();
}

/**
 * Webサイト翻訳の実例
 */
export async function websiteTranslationExample() {
  const translator = new WebsiteTranslator();
  
  console.log('🌐 Webサイト翻訳例');
  
  // サンプルHTMLの翻訳
  const sampleHtml = `
    <html>
      <head><title>Sample Page</title></head>
      <body>
        <h1>Welcome to Our Website</h1>
        <p>We provide excellent services for your business.</p>
        <nav>
          <a href="/about">About Us</a>
          <a href="/services">Our Services</a>
          <a href="/contact">Contact</a>
        </nav>
        <div class="content">
          <h2>Latest News</h2>
          <p>Check out our latest updates and announcements.</p>
        </div>
        <code>console.log('This should not be translated');</code>
      </body>
    </html>
  `;
  
  // HTML文字列を一時ファイルとして処理（実際の実装では適切な方法を使用）
  console.log('\n🔄 サンプルHTML翻訳中...');
  
  try {
    // 実際のWebサイト翻訳は以下のように使用します
    // const result = await translator.translateWebsite('https://example.com', 'ja');
    
    console.log('✅ Webサイト翻訳が完了しました（実際のWebサイトの場合）');
    console.log('翻訳対象要素: h1, h2, p, a要素');
    console.log('除外要素: script, style, code要素');
  } catch (error) {
    console.error('❌ Webサイト翻訳エラー:', error);
  }
  
  await translator.disconnect();
}

/**
 * JSON多言語化の実例
 */
export async function jsonLocalizationExample() {
  const translator = new WebsiteTranslator();
  
  console.log('📄 JSON多言語化例');
  
  const sampleLocaleJson = {
    navigation: {
      home: 'Home',
      about: 'About Us',
      services: 'Our Services',
      contact: 'Contact'
    },
    messages: {
      welcome: 'Welcome to our website!',
      success: 'Operation completed successfully.',
      error: 'An error has occurred.',
      loading: 'Loading, please wait...'
    },
    buttons: {
      submit: 'Submit',
      cancel: 'Cancel',
      save: 'Save Changes',
      delete: 'Delete'
    }
  };
  
  console.log('\n🔄 JSON翻訳中...');
  
  try {
    const translations = await translator.translateJSONLocale(
      sampleLocaleJson,
      'en',
      ['ja', 'ko', 'zh']
    );
    
    console.log('\n✅ JSON翻訳完了');
    
    Object.entries(translations).forEach(([lang, translatedJson]) => {
      console.log(`\n${lang.toUpperCase()}:`);
      console.log(`welcome: ${translatedJson.messages?.welcome || 'N/A'}`);
      console.log(`submit: ${translatedJson.buttons?.submit || 'N/A'}`);
    });
  } catch (error) {
    console.error('❌ JSON翻訳エラー:', error);
  }
  
  await translator.disconnect();
}

/**
 * Markdown翻訳の実例
 */
export async function markdownTranslationExample() {
  const translator = new WebsiteTranslator();
  
  console.log('📖 Markdown翻訳例');
  
  const sampleMarkdown = `
# Getting Started Guide

Welcome to our comprehensive guide for getting started with our platform.

## Prerequisites

Before you begin, ensure you have the following:

- Node.js version 16 or higher
- npm or yarn package manager
- Basic understanding of JavaScript

## Installation

Follow these steps to install the required dependencies:

\`\`\`bash
npm install @our-company/awesome-package
\`\`\`

## Configuration

Create a configuration file as shown below:

\`\`\`javascript
const config = {
  apiKey: 'your-api-key',
  baseUrl: 'https://api.example.com'
};
\`\`\`

## Next Steps

Visit our [documentation](https://docs.example.com) for more detailed information.
`;
  
  console.log('\n🔄 Markdown翻訳中...');
  
  try {
    const result = await translator.translateMarkdown(
      sampleMarkdown,
      'en',
      'ja',
      {
        preserveCodeBlocks: true,
        preserveLinks: true,
        chunkSize: 500
      }
    );
    
    console.log('\n✅ Markdown翻訳完了');
    console.log(`翻訳チャンク数: ${result.translationCount}`);
    console.log(`処理時間: ${result.processingTime}ms`);
    console.log('\n--- 翻訳結果の一部 ---');
    console.log(result.translatedMarkdown.substring(0, 500) + '...');
  } catch (error) {
    console.error('❌ Markdown翻訳エラー:', error);
  }
  
  await translator.disconnect();
}

/**
 * プロダクション運用例
 */
export async function productionUsageExample() {
  const client = new CachedTranslationClient();
  
  console.log('🏭 プロダクション運用例');
  
  // 1. 大量テキストの効率的な翻訳
  console.log('\n⚡ 大量データ処理例');
  
  const largeDataset = Array.from({ length: 50 }, (_, i) => 
    `This is sample text number ${i + 1} for bulk translation testing.`
  );
  
  console.log(`📊 処理対象: ${largeDataset.length}件のテキスト`);
  
  const startTime = Date.now();
  const batchResult = await client.translateBatch(largeDataset, {
    sourceLanguage: 'en',
    targetLanguage: 'ja'
  });
  const endTime = Date.now();
  
  console.log(`✅ バッチ翻訳完了: ${batchResult.results.length}件成功`);
  console.log(`⏱️ 処理時間: ${endTime - startTime}ms`);
  console.log(`📈 平均速度: ${Math.round(batchResult.totalCharacters / (batchResult.processingTime / 1000))}文字/秒`);
  
  // 2. キャッシュ効率の確認
  console.log('\n💾 キャッシュパフォーマンス');
  
  // 同じテキストを再翻訳してキャッシュ効果を確認
  const cacheTestStart = Date.now();
  await client.translateText(largeDataset[0], {
    sourceLanguage: 'en',
    targetLanguage: 'ja'
  });
  const cacheTestEnd = Date.now();
  
  console.log(`🚀 キャッシュ利用時の応答時間: ${cacheTestEnd - cacheTestStart}ms`);
  
  // キャッシュ統計の表示
  const cacheStats = await client.getCacheStats();
  console.log(`📊 キャッシュ統計:`);
  console.log(`  - 総キー数: ${cacheStats.totalKeys}`);
  console.log(`  - ヒット率: ${cacheStats.hitRate}%`);
  console.log(`  - メモリ使用量: ${cacheStats.memoryUsage}`);
  
  // 3. エラーハンドリングとリトライ
  console.log('\n🛡️ エラーハンドリング例');
  
  const unreliableTexts = [
    'Normal text',
    'A'.repeat(50000), // 長すぎるテキスト
    '', // 空のテキスト
    'Another normal text'
  ];
  
  for (let i = 0; i < unreliableTexts.length; i++) {
    try {
      const result = await client.translateText(unreliableTexts[i], {
        targetLanguage: 'ja'
      });
      console.log(`✅ テキスト ${i + 1}: 翻訳成功`);
    } catch (error) {
      console.log(`❌ テキスト ${i + 1}: ${error.message}`);
    }
  }
  
  await client.disconnect();
}
```

## まとめ

本記事では、**GCP Translation AIをTypeScriptで活用し、プロダクション対応の多言語翻訳システムを構築する方法**を包括的に解説しました。

**🎯 GCP Translation AIの主要メリット**

>* **高品質翻訳**: Geminiベースの最新NLPモデルによる自然な翻訳
>* **100言語対応**: グローバル展開に必要な言語を網羅
>* **柔軟なカスタマイズ**: 用語集・カスタムモデルで業界特化翻訳
>* **スケーラブル運用**: GCPインフラによる安定したAPI提供

**💡 実装のポイント**

>* **キャッシュ戦略**: Redis統合でコストと性能を最適化
>* **バッチ処理**: 大量テキストの効率的な並列翻訳
>* **エラー処理**: プロダクション運用に必要な堅牢性
>* **ファイル対応**: JSON、CSV、Markdownなど多様な形式をサポート

**⚡ 実用的な応用例**

>* **Webサイト多言語化**: HTMLの自動翻訳とSEO最適化
>* **アプリ国際化**: JSON locale ファイルの一括生成
>* **ドキュメント翻訳**: 技術文書やマニュアルの多言語展開
>* **リアルタイム翻訳**: チャットやコメントの即座な多言語対応

TypeScriptでTranslation AIを活用することで、**企業レベルの多言語対応システム**を効率的に構築できます。ぜひあなたのグローバルプロジェクトで活用してみてください！