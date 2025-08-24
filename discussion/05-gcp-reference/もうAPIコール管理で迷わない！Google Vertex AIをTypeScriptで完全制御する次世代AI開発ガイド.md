# もうAPIコール管理で迷わない！Google Vertex AIをTypeScriptで完全制御する次世代AI開発ガイド

## はじめに

AI開発において、「どのプラットフォームを選べばいいかわからない」「APIの管理が複雑すぎる」といった悩みを抱えていませんか？

Google Vertex AIなら、統一されたプラットフォームで複数のAIモデルを管理でき、TypeScriptとの組み合わせで型安全なAI開発が実現できます。本記事では、Vertex AIの基本概念から実装まで、すぐに現場で使える実践的な内容をお届けします！

## Google Vertex AIとは

Google Vertex AIは、Googleが提供する統合AI・機械学習プラットフォームです。従来のAI開発で直面していた「複数のツールを使い分ける複雑さ」を解消し、一つのプラットフォームでAIアプリケーションのライフサイクル全体を管理できます。

**🎯 主な特徴**

>* **統合プラットフォーム**: データ準備からモデル訓練、デプロイメントまでワンストップ
>* **多様なモデル選択**: Gemini、Palm、Codeyなど豊富なAIモデル
>* **AutoML機能**: コードなしでカスタムモデルを構築可能
>* **スケーラブル**: 小規模プロトタイプから大規模本番環境まで対応
>* **エンタープライズ対応**: セキュリティとコンプライアンスを重視した設計

**従来のAI開発 vs Vertex AI**

| 従来のアプローチ | Google Vertex AI |
|---|---|
| 複数のサービスを組み合わせ | 統合プラットフォーム |
| インフラ管理が必要 | フルマネージド |
| モデル管理が煩雑 | 統一されたモデル管理 |
| スケールが困難 | 自動スケーリング |

## 環境セットアップ

### 1\. GCPプロジェクトの準備

まず、Google Cloud Consoleでプロジェクトを作成し、Vertex AI APIを有効化しましょう。

```bash
# gcloudコマンドでの設定例
gcloud config set project YOUR_PROJECT_ID
gcloud services enable aiplatform.googleapis.com
```

### 2\. 認証設定

```bash
# サービスアカウントキーの作成と設定
gcloud auth application-default login
```

### 3\. TypeScriptプロジェクトのセットアップ

```bash
npm init -y
npm install @google-cloud/aiplatform
npm install --save-dev typescript @types/node ts-node
```

**📌 補足**

>* Vertex AI Client Libraryは最新の機能に対応するため、定期的にアップデートすることをお勧めします。

## 基本的な実装パターン

### 1\. Vertex AI クライアントの初期化

```typescript:src/vertex-ai-client.ts
import { PredictionServiceClient } from '@google-cloud/aiplatform';
import { google } from '@google-cloud/aiplatform/build/protos/protos';

export class VertexAIClient {
  private client: PredictionServiceClient;
  private projectId: string;
  private location: string;

  constructor(projectId: string, location: string = 'us-central1') {
    this.client = new PredictionServiceClient({
      apiEndpoint: `${location}-aiplatform.googleapis.com`
    });
    this.projectId = projectId;
    this.location = location;
  }

  // エンドポイントパスの生成
  private getEndpointPath(modelName: string): string {
    return this.client.endpointPath(
      this.projectId,
      this.location,
      modelName
    );
  }
}
```

### 2\. テキスト生成の実装

```typescript:src/text-generation.ts
import { VertexAIClient } from './vertex-ai-client';

interface TextGenerationOptions {
  maxTokens?: number;
  temperature?: number;
  topP?: number;
  topK?: number;
}

export class TextGenerator extends VertexAIClient {
  async generateText(
    prompt: string,
    options: TextGenerationOptions = {}
  ): Promise<string> {
    const {
      maxTokens = 1024,
      temperature = 0.7,
      topP = 0.8,
      topK = 40
    } = options;

    try {
      const instanceValue = {
        prompt: prompt,
        max_output_tokens: maxTokens,
        temperature: temperature,
        top_p: topP,
        top_k: topK
      };

      const instance = google.protobuf.Value.fromObject(instanceValue);
      const instances = [instance];

      const parameter = {
        temperature: temperature,
        max_output_tokens: maxTokens,
        top_p: topP,
        top_k: topK
      };
      const parameters = google.protobuf.Value.fromObject(parameter);

      const request = {
        endpoint: this.getEndpointPath('text-bison'),
        instances,
        parameters
      };

      const [response] = await this.client.predict(request);
      
      if (response.predictions && response.predictions[0]) {
        const prediction = response.predictions[0];
        return prediction.structValue?.fields?.content?.stringValue || '';
      }
      
      throw new Error('予期しないレスポンス形式です');
    } catch (error) {
      console.error('テキスト生成エラー:', error);
      throw error;
    }
  }
}
```

### 3\. チャットボット実装

```typescript:src/chatbot.ts
import { VertexAIClient } from './vertex-ai-client';

interface Message {
  role: 'user' | 'assistant';
  content: string;
  timestamp: Date;
}

interface ChatSession {
  id: string;
  messages: Message[];
  context: string;
}

export class VertexChatBot extends VertexAIClient {
  private sessions: Map<string, ChatSession> = new Map();

  // チャットセッションの作成
  createSession(sessionId: string, systemContext: string = ''): ChatSession {
    const session: ChatSession = {
      id: sessionId,
      messages: [],
      context: systemContext
    };
    
    this.sessions.set(sessionId, session);
    return session;
  }

  // メッセージの送信
  async sendMessage(sessionId: string, userMessage: string): Promise<string> {
    const session = this.sessions.get(sessionId);
    if (!session) {
      throw new Error(`セッション ${sessionId} が見つかりません`);
    }

    // ユーザーメッセージを履歴に追加
    session.messages.push({
      role: 'user',
      content: userMessage,
      timestamp: new Date()
    });

    try {
      // コンテキストを含むプロンプトの構築
      const conversationHistory = this.buildConversationPrompt(session);
      const response = await this.generateChatResponse(conversationHistory);

      // アシスタントの返答を履歴に追加
      session.messages.push({
        role: 'assistant',
        content: response,
        timestamp: new Date()
      });

      return response;
    } catch (error) {
      console.error(`チャット生成エラー (セッション: ${sessionId}):`, error);
      throw error;
    }
  }

  // 会話履歴からプロンプトを構築
  private buildConversationPrompt(session: ChatSession): string {
    let prompt = session.context ? `システム: ${session.context}\n\n` : '';
    
    session.messages.forEach(msg => {
      const role = msg.role === 'user' ? 'ユーザー' : 'アシスタント';
      prompt += `${role}: ${msg.content}\n`;
    });
    
    prompt += 'アシスタント: ';
    return prompt;
  }

  // チャット専用の生成ロジック
  private async generateChatResponse(prompt: string): Promise<string> {
    const instanceValue = {
      messages: [
        {
          author: 'user',
          content: prompt
        }
      ],
      context: '',
      examples: [],
      temperature: 0.3,
      maxOutputTokens: 1024,
      topP: 0.8,
      topK: 40
    };

    const instance = google.protobuf.Value.fromObject(instanceValue);
    const instances = [instance];

    const request = {
      endpoint: this.getEndpointPath('chat-bison'),
      instances
    };

    const [response] = await this.client.predict(request);
    
    if (response.predictions && response.predictions[0]) {
      const prediction = response.predictions[0];
      const candidates = prediction.structValue?.fields?.candidates?.listValue?.values;
      
      if (candidates && candidates.length > 0) {
        return candidates[0].structValue?.fields?.content?.stringValue || '';
      }
    }
    
    throw new Error('チャット応答の生成に失敗しました');
  }
}
```

## 実用的な使用例

### 1\. ドキュメント要約システム

```typescript:src/document-summarizer.ts
import { TextGenerator } from './text-generation';

export class DocumentSummarizer extends TextGenerator {
  async summarizeDocument(
    document: string,
    maxSummaryLength: number = 500
  ): Promise<string> {
    const prompt = `
以下の文書を${maxSummaryLength}文字以内で要約してください。
重要なポイントを箇条書きで整理し、簡潔で理解しやすい日本語でまとめてください。

【文書】
${document}

【要約】
`;

    return await this.generateText(prompt, {
      maxTokens: maxSummaryLength * 2, // 日本語を考慮してバッファを設ける
      temperature: 0.3 // 要約は一貫性を重視
    });
  }

  // 複数文書の一括要約
  async summarizeMultipleDocuments(documents: string[]): Promise<string[]> {
    const summaryPromises = documents.map(doc => 
      this.summarizeDocument(doc)
    );
    
    return await Promise.all(summaryPromises);
  }
}
```

### 2\. コード解析・レビューアシスタント

```typescript:src/code-reviewer.ts
import { TextGenerator } from './text-generation';

interface CodeReviewResult {
  summary: string;
  issues: Array<{
    severity: 'low' | 'medium' | 'high';
    description: string;
    line?: number;
  }>;
  suggestions: string[];
}

export class CodeReviewer extends TextGenerator {
  async reviewCode(
    code: string,
    language: string = 'typescript'
  ): Promise<CodeReviewResult> {
    const prompt = `
あなたは経験豊富な${language}開発者です。以下のコードをレビューし、JSON形式で結果を返してください。

コード:
\`\`\`${language}
${code}
\`\`\`

以下のJSON形式で回答してください：
{
  "summary": "コード全体の簡潔な評価",
  "issues": [
    {
      "severity": "low|medium|high",
      "description": "問題の説明",
      "line": 行番号（わかる場合）
    }
  ],
  "suggestions": [
    "改善提案1",
    "改善提案2"
  ]
}
`;

    try {
      const response = await this.generateText(prompt, {
        temperature: 0.2, // コードレビューは一貫性を重視
        maxTokens: 2048
      });

      // JSONパースを試行
      const cleanResponse = this.extractJson(response);
      return JSON.parse(cleanResponse) as CodeReviewResult;
    } catch (error) {
      console.error('コードレビュー解析エラー:', error);
      throw new Error('コードレビューの解析に失敗しました');
    }
  }

  private extractJson(text: string): string {
    const jsonMatch = text.match(/\{[\s\S]*\}/);
    return jsonMatch ? jsonMatch[0] : text;
  }
}
```

## エラーハンドリングとベストプラクティス

### 1\. 包括的エラーハンドリング

```typescript:src/error-handler.ts
export enum VertexAIErrorType {
  QUOTA_EXCEEDED = 'QUOTA_EXCEEDED',
  INVALID_REQUEST = 'INVALID_REQUEST',
  MODEL_ERROR = 'MODEL_ERROR',
  NETWORK_ERROR = 'NETWORK_ERROR',
  AUTHENTICATION_ERROR = 'AUTHENTICATION_ERROR'
}

export class VertexAIError extends Error {
  constructor(
    public type: VertexAIErrorType,
    message: string,
    public originalError?: Error
  ) {
    super(message);
    this.name = 'VertexAIError';
  }
}

export class ErrorHandler {
  static handleApiError(error: any): VertexAIError {
    // Google APIのエラーコードに基づく分類
    if (error.code) {
      switch (error.code) {
        case 8: // RESOURCE_EXHAUSTED
          return new VertexAIError(
            VertexAIErrorType.QUOTA_EXCEEDED,
            'APIクォータが上限に達しました',
            error
          );
        case 3: // INVALID_ARGUMENT
          return new VertexAIError(
            VertexAIErrorType.INVALID_REQUEST,
            'リクエストパラメータが不正です',
            error
          );
        case 16: // UNAUTHENTICATED
          return new VertexAIError(
            VertexAIErrorType.AUTHENTICATION_ERROR,
            '認証に失敗しました',
            error
          );
        default:
          return new VertexAIError(
            VertexAIErrorType.MODEL_ERROR,
            `APIエラー: ${error.message}`,
            error
          );
      }
    }

    return new VertexAIError(
      VertexAIErrorType.NETWORK_ERROR,
      'ネットワークエラーが発生しました',
      error
    );
  }
}
```

### 2\. リトライロジック付きクライアント

```typescript:src/resilient-client.ts
import { VertexAIClient } from './vertex-ai-client';
import { ErrorHandler, VertexAIErrorType } from './error-handler';

interface RetryOptions {
  maxRetries: number;
  initialDelay: number;
  backoffMultiplier: number;
}

export class ResilientVertexAIClient extends VertexAIClient {
  private retryOptions: RetryOptions;

  constructor(
    projectId: string,
    location: string = 'us-central1',
    retryOptions: Partial<RetryOptions> = {}
  ) {
    super(projectId, location);
    this.retryOptions = {
      maxRetries: 3,
      initialDelay: 1000,
      backoffMultiplier: 2,
      ...retryOptions
    };
  }

  async executeWithRetry<T>(
    operation: () => Promise<T>
  ): Promise<T> {
    let lastError: Error;
    
    for (let attempt = 0; attempt <= this.retryOptions.maxRetries; attempt++) {
      try {
        return await operation();
      } catch (error) {
        const handledError = ErrorHandler.handleApiError(error);
        lastError = handledError;

        // リトライ不可能なエラーの場合は即座に失敗
        if (this.shouldNotRetry(handledError.type)) {
          throw handledError;
        }

        // 最後の試行の場合はエラーを投げる
        if (attempt === this.retryOptions.maxRetries) {
          break;
        }

        // エクスポネンシャルバックオフでリトライ
        const delay = this.retryOptions.initialDelay * 
          Math.pow(this.retryOptions.backoffMultiplier, attempt);
        
        console.warn(
          `リトライ ${attempt + 1}/${this.retryOptions.maxRetries} ` +
          `(${delay}ms後): ${handledError.message}`
        );
        
        await this.sleep(delay);
      }
    }

    throw lastError!;
  }

  private shouldNotRetry(errorType: VertexAIErrorType): boolean {
    return [
      VertexAIErrorType.INVALID_REQUEST,
      VertexAIErrorType.AUTHENTICATION_ERROR
    ].includes(errorType);
  }

  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

## 実装サンプル：完成版AIアシスタント

```typescript:src/ai-assistant.ts
import { ResilientVertexAIClient } from './resilient-client';
import { DocumentSummarizer } from './document-summarizer';
import { CodeReviewer } from './code-reviewer';
import { VertexChatBot } from './chatbot';

export class AIAssistant {
  private client: ResilientVertexAIClient;
  private summarizer: DocumentSummarizer;
  private codeReviewer: CodeReviewer;
  private chatBot: VertexChatBot;

  constructor(projectId: string, location: string = 'us-central1') {
    this.client = new ResilientVertexAIClient(projectId, location);
    this.summarizer = new DocumentSummarizer(projectId, location);
    this.codeReviewer = new CodeReviewer(projectId, location);
    this.chatBot = new VertexChatBot(projectId, location);
  }

  // 統合されたAI処理メソッド
  async processRequest(request: {
    type: 'summarize' | 'review' | 'chat';
    content: string;
    sessionId?: string;
    options?: any;
  }): Promise<any> {
    return this.client.executeWithRetry(async () => {
      switch (request.type) {
        case 'summarize':
          return await this.summarizer.summarizeDocument(
            request.content,
            request.options?.maxLength
          );

        case 'review':
          return await this.codeReviewer.reviewCode(
            request.content,
            request.options?.language || 'typescript'
          );

        case 'chat':
          if (!request.sessionId) {
            throw new Error('チャット処理にはsessionIdが必要です');
          }
          return await this.chatBot.sendMessage(
            request.sessionId,
            request.content
          );

        default:
          throw new Error(`未対応のリクエストタイプ: ${request.type}`);
      }
    });
  }
}

// 使用例
async function example() {
  const assistant = new AIAssistant('your-project-id', 'us-central1');

  try {
    // ドキュメント要約
    const summary = await assistant.processRequest({
      type: 'summarize',
      content: '長いドキュメントテキスト...',
      options: { maxLength: 300 }
    });

    // コードレビュー
    const reviewResult = await assistant.processRequest({
      type: 'review',
      content: 'const x = 1; // レビュー対象のコード',
      options: { language: 'typescript' }
    });

    // チャット機能
    const chatBot = new VertexChatBot('your-project-id');
    chatBot.createSession('session-1', 'あなたは親切なプログラミングアシスタントです');
    
    const chatResponse = await assistant.processRequest({
      type: 'chat',
      content: 'TypeScriptの型について教えて',
      sessionId: 'session-1'
    });

    console.log('要約:', summary);
    console.log('コードレビュー:', reviewResult);
    console.log('チャット応答:', chatResponse);
  } catch (error) {
    console.error('処理エラー:', error);
  }
}
```

## パフォーマンス最適化

### 1\. バッチ処理とストリーミング

```typescript:src/batch-processor.ts
export class BatchProcessor extends ResilientVertexAIClient {
  async processBatch<T, R>(
    items: T[],
    processor: (item: T) => Promise<R>,
    batchSize: number = 5,
    concurrency: number = 3
  ): Promise<R[]> {
    const results: R[] = [];
    const chunks = this.chunkArray(items, batchSize);

    for (const chunk of chunks) {
      const promises = chunk.map(item => 
        this.executeWithRetry(() => processor(item))
      );
      
      const chunkResults = await Promise.all(promises);
      results.push(...chunkResults);
      
      // レート制限を考慮した遅延
      if (chunks.indexOf(chunk) < chunks.length - 1) {
        await this.sleep(1000);
      }
    }

    return results;
  }

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

```typescript:src/cached-client.ts
import { LRUCache } from 'lru-cache';
import { createHash } from 'crypto';

export class CachedVertexAIClient extends ResilientVertexAIClient {
  private cache: LRUCache<string, any>;

  constructor(
    projectId: string,
    location: string = 'us-central1',
    cacheOptions: { maxSize?: number; ttl?: number } = {}
  ) {
    super(projectId, location);
    this.cache = new LRUCache({
      max: cacheOptions.maxSize || 1000,
      ttl: cacheOptions.ttl || 1000 * 60 * 60 // 1時間
    });
  }

  async cachedExecute<T>(
    key: string,
    operation: () => Promise<T>
  ): Promise<T> {
    const cacheKey = this.generateCacheKey(key);
    const cached = this.cache.get(cacheKey);
    
    if (cached) {
      console.log(`キャッシュヒット: ${cacheKey}`);
      return cached;
    }

    const result = await this.executeWithRetry(operation);
    this.cache.set(cacheKey, result);
    
    return result;
  }

  private generateCacheKey(input: string): string {
    return createHash('sha256').update(input).digest('hex').substring(0, 16);
  }
}
```

## 本番環境での運用

**モニタリング・ログ**

```typescript:src/monitoring.ts
export class VertexAIMonitor {
  private metrics: Map<string, number> = new Map();
  private errors: Array<{ timestamp: Date; error: Error }> = [];

  recordApiCall(operation: string, duration: number, success: boolean): void {
    const key = `${operation}_${success ? 'success' : 'error'}`;
    this.metrics.set(key, (this.metrics.get(key) || 0) + 1);
    
    if (!success) {
      this.errors.push({
        timestamp: new Date(),
        error: new Error(`${operation} failed`)
      });
    }

    // メトリクス出力（本番環境ではDatadogやStackdriver等に送信）
    console.log(`[METRICS] ${key}: ${this.metrics.get(key)}, Duration: ${duration}ms`);
  }

  getMetrics(): { [key: string]: number } {
    return Object.fromEntries(this.metrics);
  }

  getRecentErrors(minutes: number = 60): Array<{ timestamp: Date; error: Error }> {
    const cutoff = new Date(Date.now() - minutes * 60 * 1000);
    return this.errors.filter(e => e.timestamp > cutoff);
  }
}
```

**設定管理**

```typescript:src/config.ts
export interface VertexAIConfig {
  projectId: string;
  location: string;
  models: {
    textGeneration: string;
    chat: string;
    embedding: string;
  };
  defaults: {
    temperature: number;
    maxTokens: number;
    topP: number;
    topK: number;
  };
  retry: {
    maxRetries: number;
    initialDelay: number;
    backoffMultiplier: number;
  };
  cache: {
    enabled: boolean;
    maxSize: number;
    ttl: number;
  };
}

export const defaultConfig: VertexAIConfig = {
  projectId: process.env.GOOGLE_CLOUD_PROJECT || '',
  location: process.env.VERTEX_AI_LOCATION || 'us-central1',
  models: {
    textGeneration: 'text-bison',
    chat: 'chat-bison',
    embedding: 'textembedding-gecko'
  },
  defaults: {
    temperature: 0.7,
    maxTokens: 1024,
    topP: 0.8,
    topK: 40
  },
  retry: {
    maxRetries: 3,
    initialDelay: 1000,
    backoffMultiplier: 2
  },
  cache: {
    enabled: true,
    maxSize: 1000,
    ttl: 3600000 // 1時間
  }
};
```

## まとめ

Google Vertex AIとTypeScriptを組み合わせることで、以下のメリットが得られます：

>* **統合プラットフォーム**: 複数のAI機能を一つのAPIで管理
>* **型安全性**: TypeScriptによる堅牢なコード品質
>* **スケーラビリティ**: 小規模から大規模まで柔軟に対応
>* **エンタープライズ対応**: セキュリティとコンプライアンスを重視
>* **豊富なモデル選択**: 用途に応じた最適なAIモデルを選択可能

本記事で紹介した実装パターンを活用して、次世代のAI開発プロジェクトを成功させてください！

**🔍 次のステップ**

>* モデルファインチューニングでカスタマイズ
>* Vector Searchとの連携でRAGシステム構築  
>* MLOpsパイプラインの構築と自動化
>* 本番環境でのパフォーマンス監視とチューニング

Vertex AIの可能性は無限大です。ぜひ実際にコードを動かして、AIの力を体感してみてください！