# もう動画生成で迷わない！Google DeepMind VeoをTypeScriptで完全制御する次世代AI動画制作ガイド

## はじめに

「テキストから高品質な動画を生成したい」
「プロダクションレベルの動画生成AIを実装したい」
「TypeScriptでVeo APIを効率的に扱いたい」

そんなあなたの悩み、Google DeepMind Veoで解決しましょう！

2024年5月にGoogle DeepMindから発表されたVeoは、テキストプロンプトから高品質な動画を生成できる最先端のAIモデルです。2024年12月にはVeo 2が、そして2025年にはVeo 3がリリースされ、4K解像度や音声生成機能も追加されました。

この記事では、Veo APIをTypeScriptで完全制御する実装方法を、基本的なセットアップから実際の動画生成、エラーハンドリングまで徹底解説します。

## Google DeepMind Veoとは

**🎬 高品質動画生成**

>* 8秒間の720p（Veo 3）/ 4K解像度（Veo 2）動画を生成
>* リアルな物理法則と映画的な表現が可能
>* テキストから動画、画像から動画の両方に対応

**🎵 ネイティブ音声生成（Veo 3）**

>* 映像に同期した音声を自動生成
>* 効果音、BGM、ナレーションに対応
>* セリフや環境音の細かな制御が可能

**⚡ 高速生成**

>* Veo 3：11秒〜6分で生成完了
>* Veo 3 Fast：より高速な軽量版も提供
>* バッチ処理による大量生成も可能

| バージョン | リリース | 解像度 | 音声 | 特徴 |
|:---:|:---:|:---:|:---:|:---|
| Veo | 2024年5月 | 720p | ❌ | 初期版、基本的なテキスト→動画 |
| Veo 2 | 2024年12月 | 4K | ❌ | 解像度向上、物理法則の改善 |
| Veo 3 | 2025年 | 720p | ✅ | 音声生成、SynthID透かし |

## 環境構築とセットアップ

**Google Cloud Projectの設定**

```bash
# Google Cloud CLIのインストール（Windows）
winget install Google.CloudSDK

# 認証設定
gcloud auth login
gcloud config set project YOUR_PROJECT_ID

# Vertex AI APIの有効化
gcloud services enable aiplatform.googleapis.com
```

**Node.jsプロジェクトの初期化**

```bash
# プロジェクト作成
mkdir veo-typescript-app
cd veo-typescript-app
npm init -y

# TypeScript環境構築
npm install -D typescript @types/node ts-node
npm install @google/genai dotenv
```

**tsconfig.json**

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "commonjs",
    "moduleResolution": "node",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "outDir": "./dist",
    "rootDir": "./src"
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

**.env**
```env
GOOGLE_GENAI_API_KEY=your_api_key_here
PROJECT_ID=your_project_id
LOCATION=us-central1
```

## 基本実装：Veo TypeScriptクライアント

### 1\. 基本クライアントクラス

**src/veo-client.ts**
```typescript
import { GoogleGenAI, ModelPart, generateVideosResponse } from '@google/genai';
import * as fs from 'fs/promises';
import * as path from 'path';
import { config } from 'dotenv';

config();

export interface VeoConfig {
  aspectRatio?: '16:9' | '9:16';
  negativePrompt?: string;
  seed?: number;
}

export interface GenerationResult {
  videoPath: string;
  duration: number;
  metadata: {
    prompt: string;
    model: string;
    generatedAt: Date;
    config?: VeoConfig;
  };
}

export class VeoClient {
  private client: GoogleGenAI;
  private outputDir: string;

  constructor(outputDir = './generated-videos') {
    const apiKey = process.env.GOOGLE_GENAI_API_KEY;
    if (!apiKey) {
      throw new Error('GOOGLE_GENAI_API_KEY environment variable is required');
    }

    this.client = new GoogleGenAI({ 
      apiKey,
      project: process.env.PROJECT_ID,
      location: process.env.LOCATION || 'us-central1'
    });
    
    this.outputDir = outputDir;
    this.ensureOutputDirectory();
  }

  private async ensureOutputDirectory(): Promise<void> {
    try {
      await fs.mkdir(this.outputDir, { recursive: true });
    } catch (error) {
      console.error('Failed to create output directory:', error);
    }
  }

  async generateVideo(
    prompt: string, 
    config?: VeoConfig,
    model: string = 'veo-3.0-generate-preview'
  ): Promise<GenerationResult> {
    console.log(`🎬 Starting video generation with prompt: "${prompt}"`);
    
    try {
      // 動画生成リクエスト
      const operation = await this.client.models.generateVideos({
        model,
        prompt,
        config: {
          aspectRatio: config?.aspectRatio || '16:9',
          negativePrompt: config?.negativePrompt,
          seed: config?.seed
        }
      });

      console.log('⏳ Waiting for video generation to complete...');
      
      // 生成完了まで待機（ポーリング）
      let completedOperation = operation;
      while (!completedOperation.done) {
        await this.sleep(20000); // 20秒待機
        completedOperation = await this.client.operations.get(operation);
        console.log('📊 Generation status:', completedOperation.done ? 'Complete' : 'In Progress');
      }

      // 結果取得とファイル保存
      const result = completedOperation.result;
      if (!result?.generated_videos?.[0]?.video) {
        throw new Error('No video generated in response');
      }

      const generatedVideo = result.generated_videos[0];
      const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
      const filename = `veo_video_${timestamp}.mp4`;
      const videoPath = path.join(this.outputDir, filename);

      // ビデオファイルをダウンロード
      await this.client.files.download({
        file: generatedVideo.video,
        destination: videoPath
      });

      console.log(`✅ Video generated successfully: ${videoPath}`);

      return {
        videoPath,
        duration: 8, // Veoは8秒動画を生成
        metadata: {
          prompt,
          model,
          generatedAt: new Date(),
          config
        }
      };

    } catch (error) {
      console.error('❌ Video generation failed:', error);
      throw new Error(`Video generation failed: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  async generateVideoFromImage(
    prompt: string,
    imagePath: string,
    config?: VeoConfig,
    model: string = 'veo-3.0-generate-preview'
  ): Promise<GenerationResult> {
    console.log(`🖼️ Starting image-to-video generation: "${prompt}"`);
    
    try {
      // 画像を読み込み
      const imageData = await fs.readFile(imagePath);
      const imagePart: ModelPart = {
        inlineData: {
          data: imageData.toString('base64'),
          mimeType: this.getMimeType(imagePath)
        }
      };

      const operation = await this.client.models.generateVideos({
        model,
        prompt,
        config: {
          aspectRatio: config?.aspectRatio || '16:9',
          negativePrompt: config?.negativePrompt,
          seed: config?.seed
        },
        contents: [
          {
            parts: [imagePart, { text: prompt }]
          }
        ]
      });

      // 以降の処理は generateVideo と同じ
      console.log('⏳ Waiting for image-to-video generation to complete...');
      
      let completedOperation = operation;
      while (!completedOperation.done) {
        await this.sleep(20000);
        completedOperation = await this.client.operations.get(operation);
      }

      const result = completedOperation.result;
      if (!result?.generated_videos?.[0]?.video) {
        throw new Error('No video generated from image');
      }

      const generatedVideo = result.generated_videos[0];
      const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
      const filename = `veo_image_to_video_${timestamp}.mp4`;
      const videoPath = path.join(this.outputDir, filename);

      await this.client.files.download({
        file: generatedVideo.video,
        destination: videoPath
      });

      console.log(`✅ Image-to-video generated successfully: ${videoPath}`);

      return {
        videoPath,
        duration: 8,
        metadata: {
          prompt,
          model,
          generatedAt: new Date(),
          config
        }
      };

    } catch (error) {
      console.error('❌ Image-to-video generation failed:', error);
      throw new Error(`Image-to-video generation failed: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  private getMimeType(filePath: string): string {
    const ext = path.extname(filePath).toLowerCase();
    switch (ext) {
      case '.jpg':
      case '.jpeg':
        return 'image/jpeg';
      case '.png':
        return 'image/png';
      case '.webp':
        return 'image/webp';
      default:
        return 'image/jpeg';
    }
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

### 2\. バッチ生成機能

**src/batch-generator.ts**
```typescript
import { VeoClient, VeoConfig, GenerationResult } from './veo-client';
import * as fs from 'fs/promises';
import * as path from 'path';

export interface BatchJob {
  id: string;
  prompt: string;
  config?: VeoConfig;
  model?: string;
}

export interface BatchResult {
  jobId: string;
  success: boolean;
  result?: GenerationResult;
  error?: string;
  executionTime: number;
}

export class VeoBatchGenerator {
  private veoClient: VeoClient;
  private maxConcurrentJobs: number;

  constructor(outputDir?: string, maxConcurrentJobs = 3) {
    this.veoClient = new VeoClient(outputDir);
    this.maxConcurrentJobs = maxConcurrentJobs;
  }

  async processBatch(jobs: BatchJob[]): Promise<BatchResult[]> {
    console.log(`🔄 Starting batch processing of ${jobs.length} jobs`);
    const results: BatchResult[] = [];
    
    // 並行実行を制限しながら処理
    for (let i = 0; i < jobs.length; i += this.maxConcurrentJobs) {
      const chunk = jobs.slice(i, i + this.maxConcurrentJobs);
      const chunkPromises = chunk.map(job => this.processJob(job));
      const chunkResults = await Promise.allSettled(chunkPromises);
      
      chunkResults.forEach((result, index) => {
        const job = chunk[index];
        if (result.status === 'fulfilled') {
          results.push(result.value);
        } else {
          results.push({
            jobId: job.id,
            success: false,
            error: result.reason?.message || 'Unknown error',
            executionTime: 0
          });
        }
      });
    }

    await this.generateBatchReport(results);
    return results;
  }

  private async processJob(job: BatchJob): Promise<BatchResult> {
    const startTime = Date.now();
    
    try {
      console.log(`🎬 Processing job ${job.id}: "${job.prompt}"`);
      
      const result = await this.veoClient.generateVideo(
        job.prompt,
        job.config,
        job.model
      );

      const executionTime = Date.now() - startTime;
      
      return {
        jobId: job.id,
        success: true,
        result,
        executionTime
      };

    } catch (error) {
      const executionTime = Date.now() - startTime;
      
      return {
        jobId: job.id,
        success: false,
        error: error instanceof Error ? error.message : 'Unknown error',
        executionTime
      };
    }
  }

  private async generateBatchReport(results: BatchResult[]): Promise<void> {
    const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
    const reportPath = path.join('./generated-videos', `batch-report-${timestamp}.json`);
    
    const report = {
      generatedAt: new Date().toISOString(),
      totalJobs: results.length,
      successfulJobs: results.filter(r => r.success).length,
      failedJobs: results.filter(r => !r.success).length,
      averageExecutionTime: results.reduce((sum, r) => sum + r.executionTime, 0) / results.length,
      results
    };

    await fs.writeFile(reportPath, JSON.stringify(report, null, 2));
    console.log(`📊 Batch report generated: ${reportPath}`);
  }

  async loadJobsFromFile(filePath: string): Promise<BatchJob[]> {
    try {
      const content = await fs.readFile(filePath, 'utf-8');
      return JSON.parse(content) as BatchJob[];
    } catch (error) {
      throw new Error(`Failed to load jobs from ${filePath}: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }
}
```

## 実用的な実装例

### 1\. シンプルな動画生成

**src/examples/basic-generation.ts**
```typescript
import { VeoClient } from '../veo-client';

async function basicExample() {
  const veoClient = new VeoClient();

  try {
    // シンプルなテキスト→動画生成
    const result = await veoClient.generateVideo(
      "A majestic golden retriever playing in a field of sunflowers at sunset, cinematic shot with shallow depth of field"
    );

    console.log('生成された動画:', result.videoPath);
    console.log('メタデータ:', result.metadata);

  } catch (error) {
    console.error('エラーが発生しました:', error);
  }
}

basicExample();
```

### 2\. 高度な設定での動画生成

**src/examples/advanced-generation.ts**
```typescript
import { VeoClient, VeoConfig } from '../veo-client';

async function advancedExample() {
  const veoClient = new VeoClient('./my-videos');

  // 高度な設定
  const config: VeoConfig = {
    aspectRatio: '16:9',
    negativePrompt: 'blurry, low quality, distorted, cartoon style',
    seed: 12345 // 再現可能な生成のため
  };

  try {
    // 複数パターンの生成
    const prompts = [
      "A serene Japanese garden with cherry blossoms falling, peaceful morning light",
      "A bustling Tokyo street at night with neon signs reflecting on wet pavement",
      "A traditional Japanese tea ceremony in a wooden house, soft natural lighting"
    ];

    for (const prompt of prompts) {
      console.log(`\n🎬 Generating: ${prompt}`);
      
      const result = await veoClient.generateVideo(prompt, config);
      
      console.log(`✅ Complete: ${result.videoPath}`);
      console.log(`📊 Generated at: ${result.metadata.generatedAt}`);
    }

  } catch (error) {
    console.error('❌ Generation failed:', error);
  }
}

advancedExample();
```

### 3\. 画像からビデオ生成

**src/examples/image-to-video.ts**
```typescript
import { VeoClient } from '../veo-client';

async function imageToVideoExample() {
  const veoClient = new VeoClient();

  try {
    // 画像を動画に変換
    const result = await veoClient.generateVideoFromImage(
      "Make this image come to life with gentle movement and natural lighting changes",
      "./source-images/landscape.jpg",
      {
        aspectRatio: '16:9',
        negativePrompt: 'rapid motion, shaking, distortion'
      }
    );

    console.log('画像から生成された動画:', result.videoPath);

  } catch (error) {
    console.error('画像→動画変換エラー:', error);
  }
}

imageToVideoExample();
```

### 4\. バッチ処理の実装

**src/examples/batch-processing.ts**
```typescript
import { VeoBatchGenerator, BatchJob } from '../batch-generator';

async function batchProcessingExample() {
  const batchGenerator = new VeoBatchGenerator('./batch-videos', 2); // 同時実行数2

  // バッチジョブの定義
  const jobs: BatchJob[] = [
    {
      id: 'nature-001',
      prompt: "A peaceful mountain lake reflecting snow-capped peaks at sunrise",
      config: { aspectRatio: '16:9' }
    },
    {
      id: 'urban-001',
      prompt: "A busy city intersection with people crossing, birds eye view",
      config: { aspectRatio: '16:9', negativePrompt: 'cars, traffic' }
    },
    {
      id: 'animal-001',
      prompt: "A family of deer grazing in a meadow, soft morning light",
      config: { aspectRatio: '9:16' }
    }
  ];

  try {
    console.log('🔄 Starting batch video generation...');
    const results = await batchGenerator.processBatch(jobs);

    // 結果の表示
    results.forEach(result => {
      if (result.success) {
        console.log(`✅ ${result.jobId}: Success (${result.executionTime}ms)`);
        console.log(`   Video: ${result.result?.videoPath}`);
      } else {
        console.log(`❌ ${result.jobId}: Failed - ${result.error}`);
      }
    });

  } catch (error) {
    console.error('バッチ処理エラー:', error);
  }
}

batchProcessingExample();
```

## エラーハンドリングとモニタリング

### 1\. 包括的エラーハンドリング

**src/error-handler.ts**
```typescript
export enum VeoErrorType {
  AUTHENTICATION = 'AUTHENTICATION',
  QUOTA_EXCEEDED = 'QUOTA_EXCEEDED',
  INVALID_PROMPT = 'INVALID_PROMPT',
  GENERATION_FAILED = 'GENERATION_FAILED',
  FILE_OPERATION = 'FILE_OPERATION',
  NETWORK = 'NETWORK'
}

export class VeoError extends Error {
  constructor(
    public type: VeoErrorType,
    message: string,
    public originalError?: Error,
    public context?: Record<string, any>
  ) {
    super(message);
    this.name = 'VeoError';
  }
}

export function handleVeoError(error: unknown, context?: Record<string, any>): VeoError {
  if (error instanceof VeoError) {
    return error;
  }

  if (error instanceof Error) {
    // Google API固有のエラーを判定
    if (error.message.includes('authentication')) {
      return new VeoError(
        VeoErrorType.AUTHENTICATION,
        'Authentication failed. Check your API key and permissions.',
        error,
        context
      );
    }

    if (error.message.includes('quota') || error.message.includes('rate limit')) {
      return new VeoError(
        VeoErrorType.QUOTA_EXCEEDED,
        'API quota exceeded. Please wait before retrying.',
        error,
        context
      );
    }

    if (error.message.includes('prompt') || error.message.includes('invalid')) {
      return new VeoError(
        VeoErrorType.INVALID_PROMPT,
        'Invalid prompt or parameters provided.',
        error,
        context
      );
    }

    if (error.message.includes('ENOENT') || error.message.includes('file')) {
      return new VeoError(
        VeoErrorType.FILE_OPERATION,
        'File operation failed. Check file paths and permissions.',
        error,
        context
      );
    }

    if (error.message.includes('network') || error.message.includes('timeout')) {
      return new VeoError(
        VeoErrorType.NETWORK,
        'Network error occurred. Check your connection and try again.',
        error,
        context
      );
    }

    return new VeoError(
      VeoErrorType.GENERATION_FAILED,
      `Video generation failed: ${error.message}`,
      error,
      context
    );
  }

  return new VeoError(
    VeoErrorType.GENERATION_FAILED,
    'An unknown error occurred during video generation.',
    undefined,
    context
  );
}
```

### 2\. リトライ機能付きクライアント

**src/resilient-veo-client.ts**
```typescript
import { VeoClient, VeoConfig, GenerationResult } from './veo-client';
import { handleVeoError, VeoError, VeoErrorType } from './error-handler';

export interface RetryConfig {
  maxRetries: number;
  baseDelay: number;
  maxDelay: number;
  retryableErrors: VeoErrorType[];
}

export class ResilientVeoClient extends VeoClient {
  private retryConfig: RetryConfig;

  constructor(
    outputDir?: string,
    retryConfig: Partial<RetryConfig> = {}
  ) {
    super(outputDir);
    
    this.retryConfig = {
      maxRetries: 3,
      baseDelay: 1000,
      maxDelay: 30000,
      retryableErrors: [
        VeoErrorType.NETWORK,
        VeoErrorType.QUOTA_EXCEEDED,
        VeoErrorType.GENERATION_FAILED
      ],
      ...retryConfig
    };
  }

  async generateVideoWithRetry(
    prompt: string,
    config?: VeoConfig,
    model?: string
  ): Promise<GenerationResult> {
    let lastError: VeoError | null = null;

    for (let attempt = 0; attempt <= this.retryConfig.maxRetries; attempt++) {
      try {
        console.log(`🎬 Attempt ${attempt + 1}/${this.retryConfig.maxRetries + 1}: "${prompt}"`);
        
        return await this.generateVideo(prompt, config, model);

      } catch (error) {
        lastError = handleVeoError(error, { 
          attempt, 
          prompt: prompt.substring(0, 100),
          model 
        });

        console.warn(`⚠️ Attempt ${attempt + 1} failed:`, lastError.message);

        // 最後の試行の場合はエラーを投げる
        if (attempt === this.retryConfig.maxRetries) {
          break;
        }

        // リトライ可能なエラーかチェック
        if (!this.retryConfig.retryableErrors.includes(lastError.type)) {
          console.error(`❌ Non-retryable error: ${lastError.type}`);
          break;
        }

        // 指数バックオフで待機
        const delay = Math.min(
          this.retryConfig.baseDelay * Math.pow(2, attempt),
          this.retryConfig.maxDelay
        );
        
        console.log(`⏱️ Retrying in ${delay}ms...`);
        await this.sleep(delay);
      }
    }

    throw lastError || new VeoError(
      VeoErrorType.GENERATION_FAILED,
      'Video generation failed after all retries'
    );
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## パフォーマンス最適化とベストプラクティス

### 1\. プロンプト最適化

**良いプロンプトの例：**
```typescript
const optimizedPrompts = {
  // ❌ 悪い例：曖昧で不具体
  bad: "A dog playing",
  
  // ✅ 良い例：具体的で詳細
  good: "A golden retriever playing fetch in a sunny park, slow motion shot with cinematic lighting, shallow depth of field, professional cinematography",
  
  // ✅ 音声込みの例（Veo 3）
  withAudio: "A jazz pianist performing in a dimly lit club, camera slowly zooming in, soft piano melody with ambient club sounds, warm golden lighting"
};
```

### 2\. コスト最適化

**src/cost-optimizer.ts**
```typescript
export class VeoCostOptimizer {
  private readonly VEO_3_COST_PER_SECOND = 0.75; // $0.75 per second
  
  calculateCost(videoDurationSeconds: number, videoCount: number): number {
    return videoDurationSeconds * videoCount * this.VEO_3_COST_PER_SECOND;
  }

  estimateBatchCost(prompts: string[]): number {
    // Veoは8秒動画を生成
    return this.calculateCost(8, prompts.length);
  }

  suggestOptimization(estimatedCost: number): string[] {
    const suggestions: string[] = [];
    
    if (estimatedCost > 50) {
      suggestions.push('💰 Consider using Veo 3 Fast for lower costs');
      suggestions.push('🎯 Test prompts with single generation before batch processing');
    }
    
    if (estimatedCost > 100) {
      suggestions.push('⏳ Process videos in smaller batches to manage costs');
      suggestions.push('🔄 Implement caching to avoid regenerating similar content');
    }

    return suggestions;
  }
}
```

### 3\. キャッシュ機能

**src/video-cache.ts**
```typescript
import * as crypto from 'crypto';
import * as fs from 'fs/promises';
import * as path from 'path';
import { GenerationResult, VeoConfig } from './veo-client';

interface CacheEntry {
  prompt: string;
  config: VeoConfig;
  result: GenerationResult;
  cachedAt: Date;
}

export class VideoCache {
  private cacheDir: string;
  private maxCacheAge: number; // ミリ秒

  constructor(cacheDir = './video-cache', maxCacheAgeHours = 24) {
    this.cacheDir = cacheDir;
    this.maxCacheAge = maxCacheAgeHours * 60 * 60 * 1000;
    this.ensureCacheDirectory();
  }

  private async ensureCacheDirectory(): Promise<void> {
    try {
      await fs.mkdir(this.cacheDir, { recursive: true });
    } catch (error) {
      console.error('Failed to create cache directory:', error);
    }
  }

  private generateCacheKey(prompt: string, config?: VeoConfig): string {
    const content = JSON.stringify({ prompt, config });
    return crypto.createHash('md5').update(content).digest('hex');
  }

  async get(prompt: string, config?: VeoConfig): Promise<GenerationResult | null> {
    const cacheKey = this.generateCacheKey(prompt, config);
    const cacheFile = path.join(this.cacheDir, `${cacheKey}.json`);

    try {
      const cacheData = await fs.readFile(cacheFile, 'utf-8');
      const entry: CacheEntry = JSON.parse(cacheData);

      // キャッシュの有効期限をチェック
      const age = Date.now() - new Date(entry.cachedAt).getTime();
      if (age > this.maxCacheAge) {
        await this.delete(cacheKey);
        return null;
      }

      // ビデオファイルが存在するかチェック
      try {
        await fs.access(entry.result.videoPath);
        console.log(`📦 Cache hit for prompt: ${prompt.substring(0, 50)}...`);
        return entry.result;
      } catch {
        // ビデオファイルが削除されている場合はキャッシュを無効化
        await this.delete(cacheKey);
        return null;
      }

    } catch {
      return null;
    }
  }

  async set(prompt: string, config: VeoConfig | undefined, result: GenerationResult): Promise<void> {
    const cacheKey = this.generateCacheKey(prompt, config);
    const cacheFile = path.join(this.cacheDir, `${cacheKey}.json`);

    const entry: CacheEntry = {
      prompt,
      config: config || {},
      result,
      cachedAt: new Date()
    };

    try {
      await fs.writeFile(cacheFile, JSON.stringify(entry, null, 2));
      console.log(`💾 Cached result for prompt: ${prompt.substring(0, 50)}...`);
    } catch (error) {
      console.error('Failed to write cache:', error);
    }
  }

  private async delete(cacheKey: string): Promise<void> {
    const cacheFile = path.join(this.cacheDir, `${cacheKey}.json`);
    try {
      await fs.unlink(cacheFile);
    } catch {
      // ファイルが存在しない場合は無視
    }
  }

  async cleanup(): Promise<void> {
    try {
      const files = await fs.readdir(this.cacheDir);
      const now = Date.now();

      for (const file of files) {
        if (!file.endsWith('.json')) continue;

        const filePath = path.join(this.cacheDir, file);
        try {
          const content = await fs.readFile(filePath, 'utf-8');
          const entry: CacheEntry = JSON.parse(content);
          
          const age = now - new Date(entry.cachedAt).getTime();
          if (age > this.maxCacheAge) {
            await fs.unlink(filePath);
            console.log(`🗑️ Cleaned up expired cache: ${file}`);
          }
        } catch {
          // 壊れたキャッシュファイルを削除
          await fs.unlink(filePath);
        }
      }
    } catch (error) {
      console.error('Cache cleanup failed:', error);
    }
  }
}
```

## 実用的なワークフロー例

### 1\. CLI ツール

**src/cli/veo-cli.ts**
```typescript
#!/usr/bin/env node
import { Command } from 'commander';
import { VeoClient } from '../veo-client';
import { VeoBatchGenerator } from '../batch-generator';
import { VeoCostOptimizer } from '../cost-optimizer';
import { VideoCache } from '../video-cache';

const program = new Command();

program
  .name('veo-cli')
  .description('Google DeepMind Veo video generation CLI')
  .version('1.0.0');

// 単一動画生成コマンド
program
  .command('generate')
  .description('Generate a single video from text prompt')
  .argument('<prompt>', 'Text prompt for video generation')
  .option('-o, --output <dir>', 'Output directory', './generated-videos')
  .option('-r, --ratio <ratio>', 'Aspect ratio (16:9 or 9:16)', '16:9')
  .option('-n, --negative <prompt>', 'Negative prompt')
  .option('-m, --model <model>', 'Model to use', 'veo-3.0-generate-preview')
  .option('--no-cache', 'Disable cache')
  .action(async (prompt, options) => {
    try {
      console.log('🎬 Starting video generation...');
      
      const cache = options.cache ? new VideoCache() : null;
      
      // キャッシュチェック
      if (cache) {
        const cached = await cache.get(prompt, {
          aspectRatio: options.ratio,
          negativePrompt: options.negative
        });
        if (cached) {
          console.log('📦 Using cached result:', cached.videoPath);
          return;
        }
      }

      const veoClient = new VeoClient(options.output);
      const result = await veoClient.generateVideo(prompt, {
        aspectRatio: options.ratio,
        negativePrompt: options.negative
      }, options.model);

      // キャッシュに保存
      if (cache) {
        await cache.set(prompt, {
          aspectRatio: options.ratio,
          negativePrompt: options.negative
        }, result);
      }

      console.log('✅ Generation completed!');
      console.log('📁 Video path:', result.videoPath);
      
      // コスト計算
      const costOptimizer = new VeoCostOptimizer();
      const cost = costOptimizer.calculateCost(8, 1);
      console.log('💰 Estimated cost:', `$${cost.toFixed(2)}`);

    } catch (error) {
      console.error('❌ Generation failed:', error);
      process.exit(1);
    }
  });

// バッチ処理コマンド
program
  .command('batch')
  .description('Generate multiple videos from job file')
  .argument('<jobFile>', 'JSON file containing batch jobs')
  .option('-o, --output <dir>', 'Output directory', './batch-videos')
  .option('-c, --concurrent <num>', 'Max concurrent jobs', '2')
  .action(async (jobFile, options) => {
    try {
      console.log('🔄 Starting batch processing...');
      
      const batchGenerator = new VeoBatchGenerator(
        options.output,
        parseInt(options.concurrent)
      );

      const jobs = await batchGenerator.loadJobsFromFile(jobFile);
      
      // コスト見積もり
      const costOptimizer = new VeoCostOptimizer();
      const estimatedCost = costOptimizer.estimateBatchCost(
        jobs.map(job => job.prompt)
      );
      
      console.log(`💰 Estimated total cost: $${estimatedCost.toFixed(2)}`);
      
      const suggestions = costOptimizer.suggestOptimization(estimatedCost);
      if (suggestions.length > 0) {
        console.log('💡 Cost optimization suggestions:');
        suggestions.forEach(suggestion => console.log(`   ${suggestion}`));
      }

      const results = await batchGenerator.processBatch(jobs);
      
      const successful = results.filter(r => r.success).length;
      console.log(`✅ Batch completed: ${successful}/${results.length} successful`);

    } catch (error) {
      console.error('❌ Batch processing failed:', error);
      process.exit(1);
    }
  });

program.parse();
```

### 2\. サンプルバッチジョブファイル

**batch-jobs.json**
```json
[
  {
    "id": "nature-sunrise",
    "prompt": "A serene mountain landscape at sunrise, golden light illuminating snow-capped peaks, misty valleys below, cinematic wide shot",
    "config": {
      "aspectRatio": "16:9",
      "negativePrompt": "people, buildings, text, watermarks"
    }
  },
  {
    "id": "city-timelapse",
    "prompt": "Tokyo cityscape time-lapse at night, bustling streets with light trails, neon signs reflecting on wet pavement, urban energy",
    "config": {
      "aspectRatio": "16:9",
      "negativePrompt": "daytime, static, blurry"
    }
  },
  {
    "id": "ocean-waves",
    "prompt": "Powerful ocean waves crashing against rocky cliffs, dramatic spray and foam, overcast sky with dramatic lighting",
    "config": {
      "aspectRatio": "9:16",
      "negativePrompt": "calm, peaceful, sunny"
    }
  }
]
```

### 3\. package.json スクリプト設定

**package.json**
```json
{
  "name": "veo-typescript-app",
  "version": "1.0.0",
  "description": "Google DeepMind Veo TypeScript implementation",
  "main": "dist/index.js",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "ts-node src/index.ts",
    "generate": "ts-node src/cli/veo-cli.ts generate",
    "batch": "ts-node src/cli/veo-cli.ts batch",
    "cache:cleanup": "ts-node -e \"import('./src/video-cache').then(m => new m.VideoCache().cleanup())\"",
    "cost:estimate": "ts-node src/examples/cost-estimation.ts"
  },
  "dependencies": {
    "@google/genai": "^0.3.0",
    "commander": "^11.1.0",
    "dotenv": "^16.3.0"
  },
  "devDependencies": {
    "@types/node": "^20.10.0",
    "ts-node": "^10.9.0",
    "typescript": "^5.3.0"
  }
}
```

## まとめ

Google DeepMind VeoをTypeScriptで完全制御する実装方法をご紹介しました。

**この記事で学んだこと：**

>* Veo APIの基本的な使い方とTypeScript実装
>* 画像からビデオへの変換機能
>* バッチ処理とエラーハンドリング
>* パフォーマンス最適化とキャッシュ機能
>* 実用的なCLIツールの構築

**次のステップ：**

>1\. 本記事のコードを参考に基本実装を試してみてください
>2\. 独自のプロンプト最適化を実験してみてください  
>3\. プロダクション環境でのモニタリング機能を追加してみてください

TypeScriptでVeoを使いこなすことで、高品質な動画生成を自動化し、効率的なコンテンツ制作環境を構築できますね。VeoのAPIを活用して、あなたのプロジェクトに革新的な動画生成機能を実装してみてください！