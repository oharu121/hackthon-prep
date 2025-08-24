# もう情報検索で迷わない！GCP Vector SearchをTypeScriptで完全制御するRAGシステム構築ガイド

## はじめに

「大量のドキュメントから関連情報を素早く見つけたい」「チャットボットに自社の知識を組み込みたい」そんな悩みを抱えていませんか？

GCP Vector Searchなら、ベクトル埋め込みを使った高度な意味検索が実現できます。さらにRAG（Retrieval-Augmented Generation）システムと組み合わせることで、自社データに基づく正確なAI回答システムが構築可能です。本記事では、Vector Searchの基本概念から実装まで、すぐに現場で使える実践的な内容をお届けします！

## GCP Vector Searchとは

GCP Vector Search（旧 Vertex AI Matching Engine）は、大規模なベクトルデータの近似最近傍検索を高速で実行できるマネージドサービスです。テキスト、画像、音声などをベクトル化し、意味的類似性に基づいた検索が可能になります。

**🎯 主な特徴**

>* **高速検索**: 数十億個のベクトルでもミリ秒単位で検索
>* **スケーラブル**: 自動スケーリングによる柔軟な負荷対応
>* **多様な距離計算**: コサイン、ユークリッド、内積による類似度計算
>* **リアルタイム更新**: データの追加・更新がリアルタイムで反映
>* **統合環境**: Vertex AIエコシステムとの完全統合

**従来の検索 vs Vector Search**

| 従来のキーワード検索 | Vector Search |
|---|---|
| 完全一致・部分一致 | 意味的類似性 |
| 語順や表記に依存 | 文脈を理解 |
| 同義語の検索が困難 | 類似概念も検索可能 |
| 構造化クエリ | 自然言語クエリ |

## 環境セットアップ

### 1\. GCPプロジェクトとAPIの有効化

```bash
# 必要なAPIを有効化
gcloud services enable aiplatform.googleapis.com
gcloud services enable storage.googleapis.com
gcloud services enable compute.googleapis.com
```

### 2\. TypeScriptプロジェクトのセットアップ

```bash
npm init -y
npm install @google-cloud/aiplatform @google-cloud/storage
npm install openai @huggingface/transformers
npm install --save-dev typescript @types/node ts-node
```

### 3\. 認証設定

```bash
# Vector Search用のサービスアカウント作成
gcloud iam service-accounts create vector-search-sa
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:vector-search-sa@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
```

## Vector Search基盤クライアントの実装

```typescript:src/vector-search-client.ts
import { IndexServiceClient, IndexEndpointServiceClient } from '@google-cloud/aiplatform';
import { Storage } from '@google-cloud/storage';

export interface VectorSearchConfig {
  projectId: string;
  location: string;
  bucketName: string;
  keyFilename?: string;
}

export interface VectorData {
  id: string;
  embedding: number[];
  metadata?: { [key: string]: any };
}

export interface SearchResult {
  id: string;
  distance: number;
  metadata?: { [key: string]: any };
}

export class VectorSearchClient {
  private indexClient: IndexServiceClient;
  private endpointClient: IndexEndpointServiceClient;
  private storage: Storage;
  private projectId: string;
  private location: string;
  private bucketName: string;

  constructor(config: VectorSearchConfig) {
    this.projectId = config.projectId;
    this.location = config.location;
    this.bucketName = config.bucketName;

    this.indexClient = new IndexServiceClient({
      keyFilename: config.keyFilename
    });

    this.endpointClient = new IndexEndpointServiceClient({
      keyFilename: config.keyFilename
    });

    this.storage = new Storage({
      projectId: config.projectId,
      keyFilename: config.keyFilename
    });
  }

  // プロジェクトのロケーションパスを取得
  private getLocationPath(): string {
    return this.indexClient.locationPath(this.projectId, this.location);
  }

  // インデックスパスを取得
  private getIndexPath(indexId: string): string {
    return this.indexClient.indexPath(this.projectId, this.location, indexId);
  }

  // エンドポイントパスを取得
  private getEndpointPath(endpointId: string): string {
    return this.endpointClient.indexEndpointPath(
      this.projectId, 
      this.location, 
      endpointId
    );
  }
}
```

## Vector Indexの管理

**インデックス作成・管理機能**

```typescript:src/vector-index-manager.ts
import { VectorSearchClient, VectorSearchConfig, VectorData } from './vector-search-client';
import * as fs from 'fs';
import * as path from 'path';

interface IndexConfig {
  displayName: string;
  description?: string;
  dimensions: number;
  approximateNeighborsCount?: number;
  distanceMeasureType: 'DOT_PRODUCT_DISTANCE' | 'COSINE_DISTANCE' | 'SQUARED_L2_DISTANCE';
  algorithm?: 'TREE_AH' | 'BRUTE_FORCE';
}

interface IndexDeploymentConfig {
  minReplicaCount: number;
  maxReplicaCount: number;
  machineType?: string;
}

export class VectorIndexManager extends VectorSearchClient {
  constructor(config: VectorSearchConfig) {
    super(config);
  }

  // Vector Indexの作成
  async createIndex(config: IndexConfig): Promise<string> {
    try {
      const locationPath = this.getLocationPath();

      const index = {
        displayName: config.displayName,
        description: config.description || '',
        metadata: {
          contentsDeltaUri: `gs://${this.bucketName}/vector-data/`,
          config: {
            dimensions: config.dimensions,
            approximateNeighborsCount: config.approximateNeighborsCount || 150,
            distanceMeasureType: config.distanceMeasureType,
            algorithm: config.algorithm || 'TREE_AH'
          }
        }
      };

      console.log('Vector Index作成中...');
      const [operation] = await this.indexClient.createIndex({
        parent: locationPath,
        index: index
      });

      const [response] = await operation.promise();
      const indexId = response.name?.split('/').pop();
      
      console.log(`Vector Index作成完了: ${indexId}`);
      return indexId!;
    } catch (error) {
      throw new Error(`Index作成エラー: ${error}`);
    }
  }

  // Index Endpointの作成
  async createIndexEndpoint(displayName: string): Promise<string> {
    try {
      const locationPath = this.getLocationPath();

      const endpoint = {
        displayName: displayName,
        description: 'Vector Search endpoint for similarity search'
      };

      console.log('Index Endpoint作成中...');
      const [operation] = await this.endpointClient.createIndexEndpoint({
        parent: locationPath,
        indexEndpoint: endpoint
      });

      const [response] = await operation.promise();
      const endpointId = response.name?.split('/').pop();
      
      console.log(`Index Endpoint作成完了: ${endpointId}`);
      return endpointId!;
    } catch (error) {
      throw new Error(`Endpoint作成エラー: ${error}`);
    }
  }

  // インデックスをエンドポイントにデプロイ
  async deployIndex(
    indexId: string,
    endpointId: string,
    deploymentConfig: IndexDeploymentConfig,
    deploymentId: string = 'default-deployment'
  ): Promise<void> {
    try {
      const endpointPath = this.getEndpointPath(endpointId);
      const indexPath = this.getIndexPath(indexId);

      const deployedIndex = {
        id: deploymentId,
        index: indexPath,
        displayName: `${deploymentId}-deployment`,
        automaticResources: {
          minReplicaCount: deploymentConfig.minReplicaCount,
          maxReplicaCount: deploymentConfig.maxReplicaCount
        }
      };

      console.log('Index デプロイ中...');
      const [operation] = await this.endpointClient.deployIndex({
        indexEndpoint: endpointPath,
        deployedIndex: deployedIndex
      });

      await operation.promise();
      console.log('Index デプロイ完了');
    } catch (error) {
      throw new Error(`Index デプロイエラー: ${error}`);
    }
  }

  // ベクトルデータのアップロード
  async uploadVectorData(
    vectorData: VectorData[],
    indexId: string
  ): Promise<void> {
    try {
      // JSONL形式でベクトルデータを準備
      const jsonlContent = vectorData.map(data => 
        JSON.stringify({
          id: data.id,
          embedding: data.embedding,
          restricts: data.metadata ? [{ namespace: 'metadata', allowList: [JSON.stringify(data.metadata)] }] : []
        })
      ).join('\n');

      // ローカルファイルに保存
      const localFilePath = `/tmp/vector_data_${indexId}.jsonl`;
      fs.writeFileSync(localFilePath, jsonlContent);

      // Cloud Storageにアップロード
      const gcsPath = `vector-data/${indexId}/data.jsonl`;
      const bucket = this.storage.bucket(this.bucketName);
      
      await bucket.upload(localFilePath, {
        destination: gcsPath
      });

      // ローカルファイルを削除
      fs.unlinkSync(localFilePath);

      console.log(`ベクトルデータアップロード完了: gs://${this.bucketName}/${gcsPath}`);
    } catch (error) {
      throw new Error(`ベクトルデータアップロードエラー: ${error}`);
    }
  }
}
```

## Embedding生成器の実装

### 1\. OpenAI Embeddings

```typescript:src/embedding-generator.ts
import OpenAI from 'openai';

export interface EmbeddingConfig {
  provider: 'openai' | 'huggingface' | 'vertex';
  model?: string;
  apiKey?: string;
}

export abstract class BaseEmbeddingGenerator {
  abstract generateEmbedding(text: string): Promise<number[]>;
  abstract generateBatchEmbeddings(texts: string[]): Promise<number[][]>;
  abstract getDimensions(): number;
}

export class OpenAIEmbeddingGenerator extends BaseEmbeddingGenerator {
  private client: OpenAI;
  private model: string;

  constructor(apiKey: string, model: string = 'text-embedding-3-small') {
    super();
    this.client = new OpenAI({ apiKey });
    this.model = model;
  }

  async generateEmbedding(text: string): Promise<number[]> {
    try {
      const response = await this.client.embeddings.create({
        model: this.model,
        input: text
      });

      return response.data[0].embedding;
    } catch (error) {
      throw new Error(`OpenAI embedding生成エラー: ${error}`);
    }
  }

  async generateBatchEmbeddings(texts: string[]): Promise<number[][]> {
    try {
      const response = await this.client.embeddings.create({
        model: this.model,
        input: texts
      });

      return response.data.map(item => item.embedding);
    } catch (error) {
      throw new Error(`バッチembedding生成エラー: ${error}`);
    }
  }

  getDimensions(): number {
    // text-embedding-3-small: 1536次元
    // text-embedding-3-large: 3072次元
    return this.model.includes('large') ? 3072 : 1536;
  }
}
```

### 2\. Vertex AI Embeddings

```typescript:src/vertex-embedding-generator.ts
import { PredictionServiceClient } from '@google-cloud/aiplatform';
import { BaseEmbeddingGenerator } from './embedding-generator';

export class VertexEmbeddingGenerator extends BaseEmbeddingGenerator {
  private client: PredictionServiceClient;
  private projectId: string;
  private location: string;
  private model: string;

  constructor(projectId: string, location: string = 'us-central1', model: string = 'textembedding-gecko') {
    super();
    this.client = new PredictionServiceClient({
      apiEndpoint: `${location}-aiplatform.googleapis.com`
    });
    this.projectId = projectId;
    this.location = location;
    this.model = model;
  }

  async generateEmbedding(text: string): Promise<number[]> {
    try {
      const endpoint = `projects/${this.projectId}/locations/${this.location}/publishers/google/models/${this.model}`;

      const instanceValue = {
        content: text
      };

      const instance = {
        structValue: {
          fields: {
            content: { stringValue: text }
          }
        }
      };

      const request = {
        endpoint: endpoint,
        instances: [instance]
      };

      const [response] = await this.client.predict(request);

      if (response.predictions && response.predictions[0]) {
        const prediction = response.predictions[0];
        const embedding = prediction.structValue?.fields?.embeddings?.structValue?.fields?.values?.listValue?.values;
        
        if (embedding) {
          return embedding.map((val: any) => val.numberValue);
        }
      }

      throw new Error('予期しないレスポンス形式');
    } catch (error) {
      throw new Error(`Vertex AI embedding生成エラー: ${error}`);
    }
  }

  async generateBatchEmbeddings(texts: string[]): Promise<number[][]> {
    const embeddings: number[][] = [];
    
    for (const text of texts) {
      const embedding = await this.generateEmbedding(text);
      embeddings.push(embedding);
    }
    
    return embeddings;
  }

  getDimensions(): number {
    // textembedding-gecko: 768次元
    // textembedding-gecko-multilingual: 768次元
    return 768;
  }
}
```

## Vector Search実行エンジン

```typescript:src/vector-search-engine.ts
import { VectorSearchClient, VectorSearchConfig, SearchResult } from './vector-search-client';
import { BaseEmbeddingGenerator } from './embedding-generator';

interface SearchOptions {
  topK?: number;
  includeMetadata?: boolean;
  filters?: { [key: string]: any };
}

export class VectorSearchEngine extends VectorSearchClient {
  private embeddingGenerator: BaseEmbeddingGenerator;

  constructor(config: VectorSearchConfig, embeddingGenerator: BaseEmbeddingGenerator) {
    super(config);
    this.embeddingGenerator = embeddingGenerator;
  }

  // テキスト検索の実行
  async searchSimilarTexts(
    query: string,
    endpointId: string,
    deploymentId: string = 'default-deployment',
    options: SearchOptions = {}
  ): Promise<SearchResult[]> {
    try {
      // クエリテキストをベクトル化
      const queryEmbedding = await this.embeddingGenerator.generateEmbedding(query);
      
      return await this.searchByVector(
        queryEmbedding,
        endpointId,
        deploymentId,
        options
      );
    } catch (error) {
      throw new Error(`テキスト検索エラー: ${error}`);
    }
  }

  // ベクトルによる直接検索
  async searchByVector(
    queryVector: number[],
    endpointId: string,
    deploymentId: string = 'default-deployment',
    options: SearchOptions = {}
  ): Promise<SearchResult[]> {
    try {
      const endpointPath = this.getEndpointPath(endpointId);
      
      const { topK = 10, includeMetadata = true, filters } = options;

      // 検索クエリの構築
      const query = {
        embedding: {
          value: queryVector
        },
        neighborCount: topK,
        returnsEmbeddings: false
      };

      const request = {
        indexEndpoint: endpointPath,
        deployedIndexId: deploymentId,
        queries: [query]
      };

      const [response] = await this.endpointClient.findNeighbors(request);

      // 検索結果の解析
      const searchResults: SearchResult[] = [];
      
      if (response.nearestNeighbors && response.nearestNeighbors[0]) {
        const neighbors = response.nearestNeighbors[0].neighbors || [];
        
        for (const neighbor of neighbors) {
          const result: SearchResult = {
            id: neighbor.datapoint?.datapointId || '',
            distance: neighbor.distance || 0
          };

          // メタデータの解析
          if (includeMetadata && neighbor.datapoint?.restricts) {
            try {
              const metadataRestrict = neighbor.datapoint.restricts.find(
                r => r.namespace === 'metadata'
              );
              
              if (metadataRestrict && metadataRestrict.allowList?.[0]) {
                result.metadata = JSON.parse(metadataRestrict.allowList[0]);
              }
            } catch (error) {
              console.warn('メタデータ解析エラー:', error);
            }
          }

          searchResults.push(result);
        }
      }

      return searchResults;
    } catch (error) {
      throw new Error(`Vector検索エラー: ${error}`);
    }
  }

  // バッチ検索の実行
  async batchSearch(
    queries: string[],
    endpointId: string,
    deploymentId: string = 'default-deployment',
    options: SearchOptions = {}
  ): Promise<SearchResult[][]> {
    const results: SearchResult[][] = [];
    
    for (const query of queries) {
      const searchResults = await this.searchSimilarTexts(
        query,
        endpointId,
        deploymentId,
        options
      );
      results.push(searchResults);
    }
    
    return results;
  }
}
```

## RAGシステムの実装

### 1\. 文書処理・分割機能

```typescript:src/document-processor.ts
export interface DocumentChunk {
  id: string;
  content: string;
  metadata: {
    source: string;
    chunkIndex: number;
    totalChunks: number;
    timestamp: Date;
    [key: string]: any;
  };
}

export class DocumentProcessor {
  // テキストをチャンクに分割
  static chunkText(
    text: string,
    chunkSize: number = 1000,
    overlapSize: number = 200,
    source: string = 'unknown'
  ): DocumentChunk[] {
    const chunks: DocumentChunk[] = [];
    const sentences = text.split(/[.!?。！？]\s+/);
    
    let currentChunk = '';
    let chunkIndex = 0;
    
    for (let i = 0; i < sentences.length; i++) {
      const sentence = sentences[i].trim();
      
      if (currentChunk.length + sentence.length > chunkSize) {
        if (currentChunk.trim()) {
          chunks.push({
            id: `${source}-chunk-${chunkIndex}`,
            content: currentChunk.trim(),
            metadata: {
              source: source,
              chunkIndex: chunkIndex,
              totalChunks: 0, // 後で更新
              timestamp: new Date()
            }
          });
          chunkIndex++;
        }
        
        // オーバーラップの処理
        if (overlapSize > 0 && currentChunk.length > overlapSize) {
          const words = currentChunk.split(/\s+/);
          const overlapWords = words.slice(-Math.floor(overlapSize / 10));
          currentChunk = overlapWords.join(' ') + ' ' + sentence;
        } else {
          currentChunk = sentence;
        }
      } else {
        currentChunk += (currentChunk ? ' ' : '') + sentence;
      }
    }
    
    // 最後のチャンクを追加
    if (currentChunk.trim()) {
      chunks.push({
        id: `${source}-chunk-${chunkIndex}`,
        content: currentChunk.trim(),
        metadata: {
          source: source,
          chunkIndex: chunkIndex,
          totalChunks: 0,
          timestamp: new Date()
        }
      });
    }

    // totalChunksを更新
    chunks.forEach(chunk => {
      chunk.metadata.totalChunks = chunks.length;
    });

    return chunks;
  }

  // Markdownファイルの処理
  static processMarkdownFile(
    filePath: string,
    chunkSize: number = 1000
  ): DocumentChunk[] {
    const fs = require('fs');
    const path = require('path');
    
    const content = fs.readFileSync(filePath, 'utf-8');
    const fileName = path.basename(filePath, '.md');
    
    // Markdownのヘッダーを考慮した分割
    const sections = content.split(/^#{1,6}\s+/m);
    const chunks: DocumentChunk[] = [];
    
    sections.forEach((section, index) => {
      if (section.trim()) {
        const sectionChunks = this.chunkText(
          section,
          chunkSize,
          200,
          `${fileName}-section-${index}`
        );
        chunks.push(...sectionChunks);
      }
    });
    
    return chunks;
  }

  // PDFファイルの処理（簡易版）
  static async processPDFFile(
    filePath: string,
    chunkSize: number = 1000
  ): Promise<DocumentChunk[]> {
    // 実際の実装では pdf2pic や pdf-parse等を使用
    throw new Error('PDF処理は外部ライブラリが必要です');
  }
}
```

### 2\. RAGシステム本体

```typescript:src/rag-system.ts
import { VectorSearchEngine } from './vector-search-engine';
import { BaseEmbeddingGenerator } from './embedding-generator';
import { DocumentProcessor, DocumentChunk } from './document-processor';
import { VectorIndexManager } from './vector-index-manager';
import OpenAI from 'openai';

interface RAGConfig {
  vectorSearchConfig: any;
  embeddingGenerator: BaseEmbeddingGenerator;
  openaiApiKey: string;
  indexId: string;
  endpointId: string;
  deploymentId?: string;
}

interface RAGResponse {
  answer: string;
  sources: Array<{
    content: string;
    source: string;
    relevanceScore: number;
  }>;
  context: string;
  confidence: number;
}

export class RAGSystem {
  private vectorSearchEngine: VectorSearchEngine;
  private indexManager: VectorIndexManager;
  private openai: OpenAI;
  private config: RAGConfig;

  constructor(config: RAGConfig) {
    this.config = config;
    this.vectorSearchEngine = new VectorSearchEngine(
      config.vectorSearchConfig,
      config.embeddingGenerator
    );
    this.indexManager = new VectorIndexManager(config.vectorSearchConfig);
    this.openai = new OpenAI({ apiKey: config.openaiApiKey });
  }

  // 文書インデックスの構築
  async buildIndex(documents: DocumentChunk[]): Promise<void> {
    console.log(`📚 ${documents.length}個の文書チャンクをインデックス化中...`);
    
    try {
      // バッチでembeddingを生成
      const batchSize = 100;
      const vectorData = [];

      for (let i = 0; i < documents.length; i += batchSize) {
        const batch = documents.slice(i, i + batchSize);
        const texts = batch.map(doc => doc.content);
        
        console.log(`バッチ ${Math.floor(i / batchSize) + 1} 処理中...`);
        const embeddings = await this.config.embeddingGenerator.generateBatchEmbeddings(texts);
        
        for (let j = 0; j < batch.length; j++) {
          vectorData.push({
            id: batch[j].id,
            embedding: embeddings[j],
            metadata: {
              content: batch[j].content,
              ...batch[j].metadata
            }
          });
        }
      }

      // Vector Searchにアップロード
      await this.indexManager.uploadVectorData(vectorData, this.config.indexId);
      console.log('✅ インデックス構築完了');
    } catch (error) {
      throw new Error(`インデックス構築エラー: ${error}`);
    }
  }

  // ファイルからインデックス構築
  async buildIndexFromFiles(filePaths: string[]): Promise<void> {
    const allChunks: DocumentChunk[] = [];

    for (const filePath of filePaths) {
      try {
        let chunks: DocumentChunk[];
        
        if (filePath.endsWith('.md')) {
          chunks = DocumentProcessor.processMarkdownFile(filePath);
        } else if (filePath.endsWith('.txt')) {
          const fs = require('fs');
          const content = fs.readFileSync(filePath, 'utf-8');
          chunks = DocumentProcessor.chunkText(
            content,
            1000,
            200,
            filePath
          );
        } else {
          console.warn(`未対応のファイル形式: ${filePath}`);
          continue;
        }

        allChunks.push(...chunks);
        console.log(`📄 ${filePath}: ${chunks.length}チャンク生成`);
      } catch (error) {
        console.error(`ファイル処理エラー ${filePath}:`, error);
      }
    }

    await this.buildIndex(allChunks);
  }

  // RAG質問応答の実行
  async query(
    question: string,
    options: {
      topK?: number;
      temperature?: number;
      maxTokens?: number;
      includeContext?: boolean;
    } = {}
  ): Promise<RAGResponse> {
    const {
      topK = 5,
      temperature = 0.7,
      maxTokens = 1000,
      includeContext = true
    } = options;

    try {
      console.log(`🔍 関連文書を検索中: "${question}"`);

      // 関連文書の検索
      const searchResults = await this.vectorSearchEngine.searchSimilarTexts(
        question,
        this.config.endpointId,
        this.config.deploymentId || 'default-deployment',
        { topK, includeMetadata: true }
      );

      if (searchResults.length === 0) {
        return {
          answer: '関連する情報が見つかりませんでした。',
          sources: [],
          context: '',
          confidence: 0
        };
      }

      // コンテキストの構築
      const contextParts = searchResults.map((result, index) => {
        const content = result.metadata?.content || '';
        const source = result.metadata?.source || 'unknown';
        return `[文書${index + 1}] (出典: ${source})\n${content}`;
      });

      const context = contextParts.join('\n\n---\n\n');

      // GPTプロンプトの構築
      const systemPrompt = `あなたは提供されたコンテキスト情報に基づいて質問に答えるアシスタントです。

重要な指示:
1. 提供されたコンテキストの情報のみを使用して回答してください
2. コンテキストに情報がない場合は、「提供された情報では回答できません」と答えてください
3. 回答は正確で具体的にしてください
4. 可能な限り出典を明記してください

コンテキスト:
${context}`;

      const userPrompt = `質問: ${question}

上記のコンテキスト情報に基づいて、この質問に回答してください。`;

      // GPTによる回答生成
      console.log('🤖 AI回答生成中...');
      const completion = await this.openai.chat.completions.create({
        model: 'gpt-4',
        messages: [
          { role: 'system', content: systemPrompt },
          { role: 'user', content: userPrompt }
        ],
        temperature: temperature,
        max_tokens: maxTokens
      });

      const answer = completion.choices[0]?.message?.content || '回答を生成できませんでした。';

      // 信頼度の計算（簡易版）
      const averageDistance = searchResults.reduce((sum, result) => sum + result.distance, 0) / searchResults.length;
      const confidence = Math.max(0, Math.min(1, 1 - averageDistance));

      return {
        answer: answer,
        sources: searchResults.map(result => ({
          content: result.metadata?.content || '',
          source: result.metadata?.source || 'unknown',
          relevanceScore: 1 - result.distance
        })),
        context: includeContext ? context : '',
        confidence: confidence
      };
    } catch (error) {
      throw new Error(`RAG質問応答エラー: ${error}`);
    }
  }

  // 対話形式のRAGチャット
  async chatWithHistory(
    question: string,
    conversationHistory: Array<{ role: 'user' | 'assistant'; content: string }> = [],
    options: any = {}
  ): Promise<RAGResponse> {
    // 会話履歴を考慮したクエリの改良
    let enhancedQuery = question;
    
    if (conversationHistory.length > 0) {
      const recentContext = conversationHistory
        .slice(-4) // 直近の4つのやり取り
        .map(msg => `${msg.role}: ${msg.content}`)
        .join('\n');
      
      enhancedQuery = `会話の流れ:\n${recentContext}\n\n現在の質問: ${question}`;
    }

    return await this.query(enhancedQuery, options);
  }
}
```

## 実用的な使用例

### 1\. 企業FAQ検索システム

```typescript:src/faq-search-system.ts
import { RAGSystem } from './rag-system';
import { OpenAIEmbeddingGenerator } from './embedding-generator';
import { DocumentProcessor } from './document-processor';

interface FAQData {
  question: string;
  answer: string;
  category: string;
  tags: string[];
  lastUpdated: Date;
}

export class FAQSearchSystem {
  private ragSystem: RAGSystem;
  private faqDatabase: Map<string, FAQData> = new Map();

  constructor(
    vectorSearchConfig: any,
    openaiApiKey: string,
    indexId: string,
    endpointId: string
  ) {
    const embeddingGenerator = new OpenAIEmbeddingGenerator(openaiApiKey);
    
    this.ragSystem = new RAGSystem({
      vectorSearchConfig,
      embeddingGenerator,
      openaiApiKey,
      indexId,
      endpointId
    });
  }

  // FAQデータの登録
  async addFAQs(faqs: FAQData[]): Promise<void> {
    console.log(`📋 ${faqs.length}件のFAQを登録中...`);

    const documents = faqs.map((faq, index) => {
      const id = `faq-${index}`;
      this.faqDatabase.set(id, faq);

      return {
        id: id,
        content: `質問: ${faq.question}\n回答: ${faq.answer}`,
        metadata: {
          source: 'FAQ',
          category: faq.category,
          tags: faq.tags,
          question: faq.question,
          answer: faq.answer,
          lastUpdated: faq.lastUpdated,
          timestamp: new Date()
        }
      };
    });

    await this.ragSystem.buildIndex(documents);
    console.log('✅ FAQ登録完了');
  }

  // FAQ検索の実行
  async searchFAQ(
    userQuery: string,
    categoryFilter?: string
  ): Promise<{
    answer: string;
    relatedFAQs: Array<{
      question: string;
      answer: string;
      category: string;
      relevanceScore: number;
    }>;
    confidence: number;
  }> {
    try {
      const response = await this.ragSystem.query(userQuery, {
        topK: 5,
        temperature: 0.3 // FAQは一貫した回答が必要
      });

      const relatedFAQs = response.sources
        .filter(source => !categoryFilter || source.source === categoryFilter)
        .map(source => ({
          question: source.source || '',
          answer: source.content,
          category: 'General',
          relevanceScore: source.relevanceScore
        }));

      return {
        answer: response.answer,
        relatedFAQs: relatedFAQs,
        confidence: response.confidence
      };
    } catch (error) {
      throw new Error(`FAQ検索エラー: ${error}`);
    }
  }

  // カテゴリ別FAQ統計
  getFAQStats(): { [category: string]: number } {
    const stats: { [category: string]: number } = {};
    
    for (const faq of this.faqDatabase.values()) {
      stats[faq.category] = (stats[faq.category] || 0) + 1;
    }
    
    return stats;
  }
}
```

### 2\. ドキュメント検索・要約システム

```typescript:src/document-search-system.ts
import { RAGSystem } from './rag-system';
import { VertexEmbeddingGenerator } from './vertex-embedding-generator';
import * as fs from 'fs';
import * as path from 'path';

export class DocumentSearchSystem {
  private ragSystem: RAGSystem;
  private indexedDocuments: Set<string> = new Set();

  constructor(
    projectId: string,
    vectorSearchConfig: any,
    openaiApiKey: string,
    indexId: string,
    endpointId: string
  ) {
    const embeddingGenerator = new VertexEmbeddingGenerator(projectId);
    
    this.ragSystem = new RAGSystem({
      vectorSearchConfig,
      embeddingGenerator,
      openaiApiKey,
      indexId,
      endpointId
    });
  }

  // ディレクトリ内の文書を一括インデックス化
  async indexDocumentDirectory(dirPath: string): Promise<void> {
    console.log(`📂 ディレクトリをインデックス化: ${dirPath}`);

    const files = this.getAllFiles(dirPath);
    const supportedFiles = files.filter(file => 
      file.endsWith('.md') || file.endsWith('.txt')
    );

    console.log(`📄 対象ファイル数: ${supportedFiles.length}`);

    await this.ragSystem.buildIndexFromFiles(supportedFiles);
    
    supportedFiles.forEach(file => this.indexedDocuments.add(file));
    console.log('✅ ディレクトリインデックス化完了');
  }

  // 文書内容の検索・要約
  async searchAndSummarize(
    query: string,
    summaryLength: 'short' | 'medium' | 'long' = 'medium'
  ): Promise<{
    summary: string;
    keyPoints: string[];
    relevantSections: Array<{
      content: string;
      source: string;
      relevance: number;
    }>;
  }> {
    try {
      // 関連文書の検索
      const searchResponse = await this.ragSystem.query(query, {
        topK: 8,
        temperature: 0.5
      });

      // 要約の生成
      const summaryPrompt = this.buildSummaryPrompt(query, searchResponse.context, summaryLength);
      
      // キーポイントの抽出
      const keyPointsPrompt = `以下のコンテキストから、「${query}」に関する重要なポイントを3-5個の箇条書きで抽出してください：

${searchResponse.context}

重要なポイント（箇条書き）:`;

      // 並行して処理
      const [summaryResponse, keyPointsResponse] = await Promise.all([
        this.generateResponse(summaryPrompt),
        this.generateResponse(keyPointsPrompt)
      ]);

      // キーポイントの整形
      const keyPoints = keyPointsResponse
        .split('\n')
        .filter(line => line.trim().startsWith('-') || line.trim().startsWith('•'))
        .map(line => line.trim().replace(/^[-•]\s*/, ''));

      return {
        summary: summaryResponse,
        keyPoints: keyPoints,
        relevantSections: searchResponse.sources.map(source => ({
          content: source.content.substring(0, 300) + '...',
          source: source.source,
          relevance: source.relevanceScore
        }))
      };
    } catch (error) {
      throw new Error(`文書検索・要約エラー: ${error}`);
    }
  }

  // 対話型文書質問応答
  async askDocument(
    question: string,
    conversationHistory: Array<{ role: 'user' | 'assistant'; content: string }> = []
  ): Promise<string> {
    const response = await this.ragSystem.chatWithHistory(
      question,
      conversationHistory,
      { topK: 6, temperature: 0.7 }
    );

    return response.answer;
  }

  // 文書の類似性分析
  async findSimilarDocuments(
    referenceText: string,
    topK: number = 5
  ): Promise<Array<{
    source: string;
    similarityScore: number;
    excerpt: string;
  }>> {
    try {
      const searchResults = await this.ragSystem.vectorSearchEngine.searchSimilarTexts(
        referenceText,
        this.ragSystem['config'].endpointId,
        this.ragSystem['config'].deploymentId || 'default-deployment',
        { topK, includeMetadata: true }
      );

      return searchResults.map(result => ({
        source: result.metadata?.source || 'unknown',
        similarityScore: 1 - result.distance,
        excerpt: result.metadata?.content?.substring(0, 200) + '...' || ''
      }));
    } catch (error) {
      throw new Error(`類似文書検索エラー: ${error}`);
    }
  }

  // プライベートメソッド
  private getAllFiles(dir: string): string[] {
    const files: string[] = [];
    const items = fs.readdirSync(dir);

    for (const item of items) {
      const fullPath = path.join(dir, item);
      const stat = fs.statSync(fullPath);

      if (stat.isDirectory()) {
        files.push(...this.getAllFiles(fullPath));
      } else {
        files.push(fullPath);
      }
    }

    return files;
  }

  private buildSummaryPrompt(query: string, context: string, length: 'short' | 'medium' | 'long'): string {
    const lengthInstructions = {
      short: '100-150語で簡潔に',
      medium: '200-300語で詳細に',
      long: '400-500語で包括的に'
    };

    return `以下のコンテキスト情報に基づいて、「${query}」について${lengthInstructions[length]}要約してください。

重要な情報を網羅し、正確で理解しやすい日本語で記述してください。

コンテキスト:
${context}

要約:`;
  }

  private async generateResponse(prompt: string): Promise<string> {
    const completion = await this.ragSystem['openai'].chat.completions.create({
      model: 'gpt-4',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.7,
      max_tokens: 1000
    });

    return completion.choices[0]?.message?.content || '';
  }
}
```

## パフォーマンス最適化・監視

```typescript:src/vector-search-monitor.ts
export class VectorSearchMonitor {
  private metrics: Map<string, any> = new Map();
  private searchHistory: Array<{
    query: string;
    resultCount: number;
    responseTime: number;
    timestamp: Date;
  }> = [];

  // 検索パフォーマンスの記録
  recordSearch(
    query: string,
    resultCount: number,
    responseTime: number
  ): void {
    this.searchHistory.push({
      query,
      resultCount,
      responseTime,
      timestamp: new Date()
    });

    // メトリクス更新
    const currentStats = this.metrics.get('search_stats') || {
      totalSearches: 0,
      averageResponseTime: 0,
      averageResultCount: 0
    };

    currentStats.totalSearches++;
    currentStats.averageResponseTime = 
      (currentStats.averageResponseTime + responseTime) / 2;
    currentStats.averageResultCount = 
      (currentStats.averageResultCount + resultCount) / 2;

    this.metrics.set('search_stats', currentStats);
  }

  // パフォーマンスレポートの生成
  generatePerformanceReport(): {
    totalSearches: number;
    averageResponseTime: number;
    averageResultCount: number;
    recentSearches: Array<{
      query: string;
      responseTime: number;
      timestamp: Date;
    }>;
    slowQueries: Array<{
      query: string;
      responseTime: number;
    }>;
  } {
    const stats = this.metrics.get('search_stats') || {};
    
    const recentSearches = this.searchHistory
      .slice(-10)
      .map(search => ({
        query: search.query,
        responseTime: search.responseTime,
        timestamp: search.timestamp
      }));

    const slowQueries = this.searchHistory
      .filter(search => search.responseTime > 2000) // 2秒以上
      .sort((a, b) => b.responseTime - a.responseTime)
      .slice(0, 5)
      .map(search => ({
        query: search.query,
        responseTime: search.responseTime
      }));

    return {
      totalSearches: stats.totalSearches || 0,
      averageResponseTime: stats.averageResponseTime || 0,
      averageResultCount: stats.averageResultCount || 0,
      recentSearches,
      slowQueries
    };
  }

  // 検索品質の分析
  analyzeSearchQuality(): {
    lowResultQueries: string[];
    highVarianceQueries: string[];
    recommendations: string[];
  } {
    const lowResultQueries = this.searchHistory
      .filter(search => search.resultCount < 3)
      .map(search => search.query);

    const queryGroups = new Map<string, number[]>();
    this.searchHistory.forEach(search => {
      if (!queryGroups.has(search.query)) {
        queryGroups.set(search.query, []);
      }
      queryGroups.get(search.query)!.push(search.responseTime);
    });

    const highVarianceQueries: string[] = [];
    for (const [query, times] of queryGroups.entries()) {
      if (times.length > 1) {
        const mean = times.reduce((a, b) => a + b) / times.length;
        const variance = times.reduce((a, b) => a + Math.pow(b - mean, 2), 0) / times.length;
        if (variance > 1000000) { // 高い分散
          highVarianceQueries.push(query);
        }
      }
    }

    const recommendations = [
      lowResultQueries.length > 0 ? 'インデックスの充実化を検討' : '',
      highVarianceQueries.length > 0 ? 'クエリの最適化を検討' : '',
      'embedding品質の定期的な評価を推奨'
    ].filter(r => r);

    return {
      lowResultQueries: [...new Set(lowResultQueries)],
      highVarianceQueries,
      recommendations
    };
  }
}
```

## まとめ

GCP Vector SearchとTypeScriptを組み合わせることで、以下のメリットが実現できます：

>* **高精度検索**: 意味的類似性に基づく自然な情報検索
>* **スケーラブル**: 大量のデータでも高速な検索性能
>* **RAGシステム**: 自社データに基づく正確なAI回答
>* **型安全性**: TypeScriptによる堅牢な開発環境
>* **統合環境**: GCPエコシステムとの完全連携

本記事で紹介した実装パターンを活用して、次世代の情報検索システムを構築してください！

**🔍 次のステップ**

>* マルチモーダル検索（テキスト+画像）の実装
>* リアルタイムデータ更新システムの構築
>* A/Bテストによる検索品質の継続的改善
>* 企業向けセキュリティ・プライバシー対応の強化

Vector Searchの力で、あなたの組織の知識活用を革新しましょう。まずは小さなデータセットから始めて、情報検索の新たな可能性を体感してください！