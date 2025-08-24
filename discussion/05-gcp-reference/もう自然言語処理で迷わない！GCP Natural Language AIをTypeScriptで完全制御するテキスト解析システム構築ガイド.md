## もう自然言語処理で迷わない！GCP Natural Language AIをTypeScriptで完全制御するテキスト解析システム構築ガイド

## はじめに

テキストデータの分析や処理を行う際に、こんな悩みはありませんか？

> 「顧客レビューの感情分析をしたいけど、どう実装すれば...」
> 「大量のテキストから重要なエンティティを抽出したい」
> 「TypeScriptで自然言語処理を本格運用したい」

本記事では、**GCP Natural Language AIをTypeScriptで活用して、テキスト分析システムを構築する方法**を実際のコード例とともに解説します。感情分析、エンティティ抽出、構文解析まで、実務で即使える実装パターンをご紹介します。

## GCP Natural Language AIとは

GCP Natural Language AIは、**機械学習を使ってテキストの構造と意味を理解するためのサービス**です。

| 機能 | 説明 | 使用場面 |
|------|------|----------|
| **感情分析** | テキストの感情（ポジティブ/ネガティブ）を分析 | 顧客レビュー解析、SNS監視 |
| **エンティティ分析** | 人名、場所、組織などを抽出・分類 | 文書解析、情報抽出 |
| **構文解析** | 文の文法構造を解析 | 言語学習アプリ、翻訳システム |
| **テキスト分類** | カスタムカテゴリでテキストを分類 | コンテンツ分類、スパム検出 |
| **言語検出** | 使用されている言語を特定 | 多言語対応システム |

## TypeScript環境のセットアップ

### 1\. 必要なパッケージのインストール

```bash
npm install @google-cloud/language @google-cloud/storage
npm install @types/node dotenv express @types/express
npm install winston
```

### 2\. GCP認証設定

```typescript:src/config/gcp-config.ts
import * as dotenv from 'dotenv';

dotenv.config();

export const GCP_CONFIG = {
  projectId: process.env.GOOGLE_CLOUD_PROJECT_ID!,
  keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
  
  // Natural Language AI設定
  naturalLanguage: {
    apiEndpoint: 'language.googleapis.com'
  }
} as const;

// 環境変数の検証
export const validateConfig = (): void => {
  if (!GCP_CONFIG.projectId) {
    throw new Error('GOOGLE_CLOUD_PROJECT_ID is required');
  }
  
  if (!GCP_CONFIG.keyFilename) {
    throw new Error('GOOGLE_APPLICATION_CREDENTIALS is required');
  }
};
```

### 3\. 基本のクライアント設定

```typescript:src/services/natural-language-client.ts
import { LanguageServiceClient } from '@google-cloud/language';
import { GCP_CONFIG, validateConfig } from '../config/gcp-config';
import { Logger } from 'winston';
import { createLogger } from '../utils/logger';

export class NaturalLanguageClient {
  private client: LanguageServiceClient;
  private logger: Logger;

  constructor() {
    validateConfig();
    
    this.client = new LanguageServiceClient({
      projectId: GCP_CONFIG.projectId,
      keyFilename: GCP_CONFIG.keyFilename,
    });
    
    this.logger = createLogger('NaturalLanguageClient');
  }

  /**
   * サービスの接続確認
   */
  async testConnection(): Promise<boolean> {
    try {
      const request = {
        document: {
          content: 'Hello world',
          type: 'PLAIN_TEXT' as const
        },
        features: {
          extractSyntax: true
        }
      };

      await this.client.annotateText(request);
      this.logger.info('Natural Language API接続確認成功');
      return true;
    } catch (error) {
      this.logger.error('Natural Language API接続エラー:', error);
      return false;
    }
  }
}
```

## 感情分析の実装

**基本の感情分析**

```typescript:src/services/sentiment-analyzer.ts
import { LanguageServiceClient } from '@google-cloud/language';
import { GCP_CONFIG } from '../config/gcp-config';

interface SentimentResult {
  score: number;        // -1.0（ネガティブ） ～ 1.0（ポジティブ）
  magnitude: number;    // 0.0（中性） ～ +∞（感情の強度）
  text: string;
  classification: 'positive' | 'negative' | 'neutral';
  confidence: 'high' | 'medium' | 'low';
}

export class SentimentAnalyzer {
  private client: LanguageServiceClient;

  constructor() {
    this.client = new LanguageServiceClient({
      projectId: GCP_CONFIG.projectId,
      keyFilename: GCP_CONFIG.keyFilename,
    });
  }

  /**
   * テキストの感情分析を実行
   */
  async analyzeSentiment(text: string): Promise<SentimentResult> {
    const request = {
      document: {
        content: text,
        type: 'PLAIN_TEXT' as const
      },
      encodingType: 'UTF8' as const
    };

    const [result] = await this.client.analyzeSentiment(request);
    
    if (!result.documentSentiment) {
      throw new Error('感情分析結果が取得できませんでした');
    }

    const { score, magnitude } = result.documentSentiment;
    
    return {
      score: score || 0,
      magnitude: magnitude || 0,
      text,
      classification: this.classifySentiment(score || 0),
      confidence: this.calculateConfidence(magnitude || 0)
    };
  }

  /**
   * スコアに基づく感情分類
   */
  private classifySentiment(score: number): 'positive' | 'negative' | 'neutral' {
    if (score > 0.1) return 'positive';
    if (score < -0.1) return 'negative';
    return 'neutral';
  }

  /**
   * 信頼度の計算
   */
  private calculateConfidence(magnitude: number): 'high' | 'medium' | 'low' {
    if (magnitude > 0.8) return 'high';
    if (magnitude > 0.4) return 'medium';
    return 'low';
  }

  /**
   * バッチ処理による複数テキストの感情分析
   */
  async analyzeBatch(texts: string[]): Promise<SentimentResult[]> {
    const results: SentimentResult[] = [];
    
    // 並列処理でスループット向上
    const promises = texts.map(text => this.analyzeSentiment(text));
    const sentimentResults = await Promise.all(promises);
    
    return sentimentResults;
  }
}
```

**カスタマーレビュー分析の実践例**

```typescript:src/services/review-analyzer.ts
import { SentimentAnalyzer } from './sentiment-analyzer';

interface Review {
  id: string;
  content: string;
  rating: number;
  date: Date;
}

interface ReviewAnalysis extends Review {
  sentiment: {
    score: number;
    classification: string;
    confidence: string;
    isConsistent: boolean;  // 評価点と感情分析の一致度
  };
}

export class ReviewAnalyzer {
  private sentimentAnalyzer: SentimentAnalyzer;

  constructor() {
    this.sentimentAnalyzer = new SentimentAnalyzer();
  }

  /**
   * レビューの感情分析と評価の一致性チェック
   */
  async analyzeReview(review: Review): Promise<ReviewAnalysis> {
    const sentimentResult = await this.sentimentAnalyzer.analyzeSentiment(review.content);
    
    return {
      ...review,
      sentiment: {
        score: sentimentResult.score,
        classification: sentimentResult.classification,
        confidence: sentimentResult.confidence,
        isConsistent: this.checkConsistency(review.rating, sentimentResult.score)
      }
    };
  }

  /**
   * 評価点と感情分析結果の一致性確認
   */
  private checkConsistency(rating: number, sentimentScore: number): boolean {
    const normalizedRating = (rating - 3) / 2; // 1-5を-1〜1に正規化
    const scoreDifference = Math.abs(normalizedRating - sentimentScore);
    
    return scoreDifference < 0.5; // 0.5以内なら一致と判定
  }

  /**
   * レビュー群の統計分析
   */
  async generateReviewStats(reviews: Review[]): Promise<{
    totalReviews: number;
    averageSentiment: number;
    sentimentDistribution: Record<string, number>;
    inconsistentReviews: ReviewAnalysis[];
  }> {
    const analyses = await Promise.all(
      reviews.map(review => this.analyzeReview(review))
    );

    const totalSentiment = analyses.reduce((sum, analysis) => sum + analysis.sentiment.score, 0);
    const averageSentiment = totalSentiment / analyses.length;

    const distribution = analyses.reduce((acc, analysis) => {
      acc[analysis.sentiment.classification] = (acc[analysis.sentiment.classification] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);

    const inconsistentReviews = analyses.filter(analysis => !analysis.sentiment.isConsistent);

    return {
      totalReviews: analyses.length,
      averageSentiment,
      sentimentDistribution: distribution,
      inconsistentReviews
    };
  }
}
```

## エンティティ分析の実装

### 1\. 基本のエンティティ抽出

```typescript:src/services/entity-extractor.ts
import { LanguageServiceClient } from '@google-cloud/language';
import { GCP_CONFIG } from '../config/gcp-config';

interface Entity {
  name: string;
  type: string;
  salience: number;    // 重要度（0.0 ～ 1.0）
  mentions: Array<{
    text: string;
    type: string;
    beginOffset: number;
  }>;
  metadata: Record<string, string>;
  sentiment?: {
    score: number;
    magnitude: number;
  };
}

export class EntityExtractor {
  private client: LanguageServiceClient;

  constructor() {
    this.client = new LanguageServiceClient({
      projectId: GCP_CONFIG.projectId,
      keyFilename: GCP_CONFIG.keyFilename,
    });
  }

  /**
   * エンティティ抽出と感情分析を同時実行
   */
  async extractEntities(text: string, includeSentiment = true): Promise<Entity[]> {
    const request = {
      document: {
        content: text,
        type: 'PLAIN_TEXT' as const
      },
      encodingType: 'UTF8' as const
    };

    const [result] = includeSentiment 
      ? await this.client.analyzeEntitySentiment(request)
      : await this.client.analyzeEntities(request);

    if (!result.entities) {
      return [];
    }

    return result.entities.map(entity => ({
      name: entity.name || '',
      type: entity.type || 'UNKNOWN',
      salience: entity.salience || 0,
      mentions: (entity.mentions || []).map(mention => ({
        text: mention.text?.content || '',
        type: mention.type || '',
        beginOffset: mention.text?.beginOffset || 0
      })),
      metadata: entity.metadata || {},
      sentiment: includeSentiment && entity.sentiment ? {
        score: entity.sentiment.score || 0,
        magnitude: entity.sentiment.magnitude || 0
      } : undefined
    }));
  }

  /**
   * 重要度でフィルタリングしたエンティティ取得
   */
  async extractImportantEntities(text: string, threshold = 0.1): Promise<Entity[]> {
    const entities = await this.extractEntities(text);
    return entities.filter(entity => entity.salience >= threshold);
  }

  /**
   * タイプ別エンティティ分類
   */
  async categorizeEntities(text: string): Promise<Record<string, Entity[]>> {
    const entities = await this.extractEntities(text);
    
    return entities.reduce((categories, entity) => {
      const type = entity.type.toLowerCase();
      if (!categories[type]) {
        categories[type] = [];
      }
      categories[type].push(entity);
      return categories;
    }, {} as Record<string, Entity[]>);
  }
}
```

### 2\. 文書解析

```typescript:src/services/document-analyzer.ts
import { EntityExtractor } from './entity-extractor';
import { SentimentAnalyzer } from './sentiment-analyzer';

interface DocumentAnalysis {
  document: string;
  summary: {
    wordCount: number;
    sentenceCount: number;
    sentiment: {
      overall: string;
      score: number;
      confidence: string;
    };
  };
  entities: {
    people: string[];
    organizations: string[];
    locations: string[];
    other: Array<{name: string; type: string}>;
  };
  keyTopics: string[];
}

export class DocumentAnalyzer {
  private entityExtractor: EntityExtractor;
  private sentimentAnalyzer: SentimentAnalyzer;

  constructor() {
    this.entityExtractor = new EntityExtractor();
    this.sentimentAnalyzer = new SentimentAnalyzer();
  }

  /**
   * 包括的な文書分析
   */
  async analyzeDocument(text: string): Promise<DocumentAnalysis> {
    // 並列で各種分析を実行
    const [sentimentResult, categorizedEntities] = await Promise.all([
      this.sentimentAnalyzer.analyzeSentiment(text),
      this.entityExtractor.categorizeEntities(text)
    ]);

    const basicStats = this.calculateBasicStats(text);

    return {
      document: text,
      summary: {
        wordCount: basicStats.wordCount,
        sentenceCount: basicStats.sentenceCount,
        sentiment: {
          overall: sentimentResult.classification,
          score: sentimentResult.score,
          confidence: sentimentResult.confidence
        }
      },
      entities: {
        people: (categorizedEntities.person || []).map(e => e.name),
        organizations: (categorizedEntities.organization || []).map(e => e.name),
        locations: (categorizedEntities.location || []).map(e => e.name),
        other: Object.entries(categorizedEntities)
          .filter(([type]) => !['person', 'organization', 'location'].includes(type))
          .flatMap(([type, entities]) => entities.map(e => ({name: e.name, type})))
      },
      keyTopics: this.extractKeyTopics(categorizedEntities)
    };
  }

  /**
   * 基本統計の計算
   */
  private calculateBasicStats(text: string): {wordCount: number; sentenceCount: number} {
    const wordCount = text.trim().split(/\s+/).length;
    const sentenceCount = text.split(/[.!?]+/).filter(s => s.trim().length > 0).length;
    
    return { wordCount, sentenceCount };
  }

  /**
   * 重要度に基づくキートピック抽出
   */
  private extractKeyTopics(categorizedEntities: Record<string, any[]>): string[] {
    const allEntities = Object.values(categorizedEntities).flat();
    
    return allEntities
      .filter(entity => entity.salience > 0.1)
      .sort((a, b) => b.salience - a.salience)
      .slice(0, 5)
      .map(entity => entity.name);
  }
}
```

### 3\. 構文解析の実装

```typescript:src/services/syntax-analyzer.ts
import { LanguageServiceClient } from '@google-cloud/language';
import { GCP_CONFIG } from '../config/gcp-config';

interface Token {
  text: string;
  partOfSpeech: {
    tag: string;
    aspect: string;
    case: string;
    form: string;
    gender: string;
    mood: string;
    number: string;
    person: string;
    proper: string;
    reciprocity: string;
    tense: string;
    voice: string;
  };
  dependencyEdge: {
    headTokenIndex: number;
    label: string;
  };
  lemma: string;
}

export class SyntaxAnalyzer {
  private client: LanguageServiceClient;

  constructor() {
    this.client = new LanguageServiceClient({
      projectId: GCP_CONFIG.projectId,
      keyFilename: GCP_CONFIG.keyFilename,
    });
  }

  /**
   * 構文解析の実行
   */
  async analyzeSyntax(text: string): Promise<Token[]> {
    const request = {
      document: {
        content: text,
        type: 'PLAIN_TEXT' as const
      },
      encodingType: 'UTF8' as const
    };

    const [result] = await this.client.analyzeSyntax(request);

    if (!result.tokens) {
      return [];
    }

    return result.tokens.map(token => ({
      text: token.text?.content || '',
      partOfSpeech: {
        tag: token.partOfSpeech?.tag || '',
        aspect: token.partOfSpeech?.aspect || '',
        case: token.partOfSpeech?.case || '',
        form: token.partOfSpeech?.form || '',
        gender: token.partOfSpeech?.gender || '',
        mood: token.partOfSpeech?.mood || '',
        number: token.partOfSpeech?.number || '',
        person: token.partOfSpeech?.person || '',
        proper: token.partOfSpeech?.proper || '',
        reciprocity: token.partOfSpeech?.reciprocity || '',
        tense: token.partOfSpeech?.tense || '',
        voice: token.partOfSpeech?.voice || ''
      },
      dependencyEdge: {
        headTokenIndex: token.dependencyEdge?.headTokenIndex || 0,
        label: token.dependencyEdge?.label || ''
      },
      lemma: token.lemma || ''
    }));
  }

  /**
   * 品詞別の統計分析
   */
  async getPOSStatistics(text: string): Promise<Record<string, number>> {
    const tokens = await this.analyzeSyntax(text);
    
    return tokens.reduce((stats, token) => {
      const tag = token.partOfSpeech.tag;
      stats[tag] = (stats[tag] || 0) + 1;
      return stats;
    }, {} as Record<string, number>);
  }
}
```

### 4\. REST API の実装

```typescript:src/api/natural-language-api.ts
import express from 'express';
import { SentimentAnalyzer } from '../services/sentiment-analyzer';
import { EntityExtractor } from '../services/entity-extractor';
import { DocumentAnalyzer } from '../services/document-analyzer';
import { SyntaxAnalyzer } from '../services/syntax-analyzer';

const router = express.Router();

// サービスインスタンス
const sentimentAnalyzer = new SentimentAnalyzer();
const entityExtractor = new EntityExtractor();
const documentAnalyzer = new DocumentAnalyzer();
const syntaxAnalyzer = new SyntaxAnalyzer();

/**
 * 感情分析エンドポイント
 */
router.post('/sentiment', async (req, res) => {
  try {
    const { text } = req.body;
    
    if (!text) {
      return res.status(400).json({ error: 'テキストが必要です' });
    }

    const result = await sentimentAnalyzer.analyzeSentiment(text);
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: '感情分析に失敗しました' });
  }
});

/**
 * エンティティ分析エンドポイント
 */
router.post('/entities', async (req, res) => {
  try {
    const { text, threshold = 0.1 } = req.body;
    
    if (!text) {
      return res.status(400).json({ error: 'テキストが必要です' });
    }

    const entities = await entityExtractor.extractImportantEntities(text, threshold);
    res.json({ entities });
  } catch (error) {
    res.status(500).json({ error: 'エンティティ分析に失敗しました' });
  }
});

/**
 * 包括的な文書分析エンドポイント
 */
router.post('/analyze', async (req, res) => {
  try {
    const { text } = req.body;
    
    if (!text) {
      return res.status(400).json({ error: 'テキストが必要です' });
    }

    const analysis = await documentAnalyzer.analyzeDocument(text);
    res.json(analysis);
  } catch (error) {
    res.status(500).json({ error: '文書分析に失敗しました' });
  }
});

/**
 * 構文解析エンドポイント
 */
router.post('/syntax', async (req, res) => {
  try {
    const { text } = req.body;
    
    if (!text) {
      return res.status(400).json({ error: 'テキストが必要です' });
    }

    const tokens = await syntaxAnalyzer.analyzeSyntax(text);
    const stats = await syntaxAnalyzer.getPOSStatistics(text);
    
    res.json({ tokens, statistics: stats });
  } catch (error) {
    res.status(500).json({ error: '構文解析に失敗しました' });
  }
});

/**
 * バッチ処理エンドポイント
 */
router.post('/batch/sentiment', async (req, res) => {
  try {
    const { texts } = req.body;
    
    if (!Array.isArray(texts)) {
      return res.status(400).json({ error: 'テキスト配列が必要です' });
    }

    const results = await sentimentAnalyzer.analyzeBatch(texts);
    res.json({ results });
  } catch (error) {
    res.status(500).json({ error: 'バッチ処理に失敗しました' });
  }
});

export default router;
```

## 実用的な使用例

### 1\. カスタマーサポート分析

```typescript:src/examples/customer-support-analyzer.ts
import { DocumentAnalyzer } from '../services/document-analyzer';
import { SentimentAnalyzer } from '../services/sentiment-analyzer';

interface SupportTicket {
  id: string;
  subject: string;
  content: string;
  priority: 'low' | 'medium' | 'high';
  created: Date;
}

export class CustomerSupportAnalyzer {
  private documentAnalyzer: DocumentAnalyzer;
  private sentimentAnalyzer: SentimentAnalyzer;

  constructor() {
    this.documentAnalyzer = new DocumentAnalyzer();
    this.sentimentAnalyzer = new SentimentAnalyzer();
  }

  /**
   * サポートチケットの自動分析と優先度調整
   */
  async analyzeTicket(ticket: SupportTicket): Promise<{
    originalPriority: string;
    suggestedPriority: string;
    reasonForChange?: string;
    sentiment: any;
    keyTopics: string[];
  }> {
    const analysis = await this.documentAnalyzer.analyzeDocument(ticket.content);
    const sentimentResult = await this.sentimentAnalyzer.analyzeSentiment(ticket.content);
    
    // ネガティブで強い感情の場合は優先度を上げる
    let suggestedPriority = ticket.priority;
    let reasonForChange;
    
    if (sentimentResult.score < -0.5 && sentimentResult.confidence === 'high') {
      if (ticket.priority === 'low') {
        suggestedPriority = 'medium';
        reasonForChange = '強いネガティブ感情が検出されました';
      } else if (ticket.priority === 'medium') {
        suggestedPriority = 'high';
        reasonForChange = '強いネガティブ感情が検出されました';
      }
    }

    return {
      originalPriority: ticket.priority,
      suggestedPriority,
      reasonForChange,
      sentiment: sentimentResult,
      keyTopics: analysis.keyTopics
    };
  }
}
```

### 2\. ソーシャルメディア監視

```typescript:src/examples/social-media-monitor.ts
import { SentimentAnalyzer } from '../services/sentiment-analyzer';
import { EntityExtractor } from '../services/entity-extractor';

interface SocialPost {
  id: string;
  content: string;
  platform: 'twitter' | 'facebook' | 'instagram';
  author: string;
  timestamp: Date;
}

export class SocialMediaMonitor {
  private sentimentAnalyzer: SentimentAnalyzer;
  private entityExtractor: EntityExtractor;

  constructor() {
    this.sentimentAnalyzer = new SentimentAnalyzer();
    this.entityExtractor = new EntityExtractor();
  }

  /**
   * ブランド言及の分析
   */
  async analyzeBrandMentions(posts: SocialPost[], brandKeywords: string[]): Promise<{
    totalMentions: number;
    sentimentBreakdown: Record<string, number>;
    influentialPosts: Array<SocialPost & {sentiment: any; entities: any[]}>;
    alerts: Array<{post: SocialPost; reason: string}>;
  }> {
    const relevantPosts = posts.filter(post => 
      brandKeywords.some(keyword => 
        post.content.toLowerCase().includes(keyword.toLowerCase())
      )
    );

    const analyses = await Promise.all(
      relevantPosts.map(async post => {
        const [sentiment, entities] = await Promise.all([
          this.sentimentAnalyzer.analyzeSentiment(post.content),
          this.entityExtractor.extractImportantEntities(post.content)
        ]);

        return { ...post, sentiment, entities };
      })
    );

    // 感情分析の集計
    const sentimentBreakdown = analyses.reduce((acc, analysis) => {
      acc[analysis.sentiment.classification] = (acc[analysis.sentiment.classification] || 0) + 1;
      return acc;
    }, {} as Record<string, number>);

    // 影響力のある投稿（強いネガティブまたはポジティブ）
    const influentialPosts = analyses.filter(analysis => 
      Math.abs(analysis.sentiment.score) > 0.5 && analysis.sentiment.confidence === 'high'
    );

    // アラート対象（強いネガティブ感情）
    const alerts = analyses
      .filter(analysis => analysis.sentiment.score < -0.7)
      .map(analysis => ({
        post: analysis,
        reason: `強いネガティブ感情（スコア: ${analysis.sentiment.score.toFixed(2)}）`
      }));

    return {
      totalMentions: relevantPosts.length,
      sentimentBreakdown,
      influentialPosts,
      alerts
    };
  }
}
```

## パフォーマンス最適化とベストプラクティス

### 1\. キャッシング戦略

```typescript:src/utils/cache-manager.ts
import { createHash } from 'crypto';

interface CacheItem<T> {
  data: T;
  timestamp: number;
  ttl: number;
}

export class CacheManager<T> {
  private cache = new Map<string, CacheItem<T>>();
  private defaultTTL: number;

  constructor(defaultTTL = 3600000) { // 1時間
    this.defaultTTL = defaultTTL;
  }

  /**
   * テキストベースのキャッシュキー生成
   */
  private generateKey(text: string, options?: any): string {
    const content = JSON.stringify({ text, options });
    return createHash('md5').update(content).digest('hex');
  }

  /**
   * キャッシュから取得
   */
  get(text: string, options?: any): T | null {
    const key = this.generateKey(text, options);
    const item = this.cache.get(key);
    
    if (!item) return null;
    
    if (Date.now() - item.timestamp > item.ttl) {
      this.cache.delete(key);
      return null;
    }
    
    return item.data;
  }

  /**
   * キャッシュに保存
   */
  set(text: string, data: T, options?: any, ttl?: number): void {
    const key = this.generateKey(text, options);
    
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl: ttl || this.defaultTTL
    });
  }

  /**
   * キャッシュクリア
   */
  clear(): void {
    this.cache.clear();
  }
}
```

### 2\. レート制限対応

```typescript:src/utils/rate-limiter.ts
export class RateLimiter {
  private requests: number[] = [];
  private maxRequests: number;
  private windowMs: number;

  constructor(maxRequests = 100, windowMs = 60000) { // 1分間に100リクエスト
    this.maxRequests = maxRequests;
    this.windowMs = windowMs;
  }

  /**
   * リクエスト可能かチェック
   */
  canMakeRequest(): boolean {
    const now = Date.now();
    
    // ウィンドウ外のリクエストを除去
    this.requests = this.requests.filter(time => now - time < this.windowMs);
    
    return this.requests.length < this.maxRequests;
  }

  /**
   * リクエスト記録
   */
  recordRequest(): void {
    this.requests.push(Date.now());
  }

  /**
   * 次のリクエスト可能時刻
   */
  getNextAvailableTime(): number {
    if (this.canMakeRequest()) return 0;
    
    const oldestRequest = Math.min(...this.requests);
    return oldestRequest + this.windowMs - Date.now();
  }
}
```

### 3\. エラーハンドリング強化

```typescript:src/services/enhanced-natural-language-client.ts
import { NaturalLanguageClient } from './natural-language-client';
import { CacheManager } from '../utils/cache-manager';
import { RateLimiter } from '../utils/rate-limiter';
import { createLogger } from '../utils/logger';

export class EnhancedNaturalLanguageClient extends NaturalLanguageClient {
  private cache: CacheManager<any>;
  private rateLimiter: RateLimiter;
  private logger = createLogger('EnhancedNaturalLanguageClient');

  constructor() {
    super();
    this.cache = new CacheManager();
    this.rateLimiter = new RateLimiter();
  }

  /**
   * キャッシュとレート制限を考慮した感情分析
   */
  async analyzeSentimentWithCache(text: string): Promise<any> {
    // キャッシュチェック
    const cachedResult = this.cache.get(text, 'sentiment');
    if (cachedResult) {
      this.logger.info('キャッシュからの結果を返します');
      return cachedResult;
    }

    // レート制限チェック
    if (!this.rateLimiter.canMakeRequest()) {
      const waitTime = this.rateLimiter.getNextAvailableTime();
      await new Promise(resolve => setTimeout(resolve, waitTime));
    }

    try {
      this.rateLimiter.recordRequest();
      const result = await this.client.analyzeSentiment({
        document: { content: text, type: 'PLAIN_TEXT' },
        encodingType: 'UTF8'
      });

      // キャッシュに保存
      this.cache.set(text, result, 'sentiment');
      
      return result;
    } catch (error: any) {
      this.logger.error('感情分析エラー:', error);
      
      if (error.code === 429) {
        // レート制限エラーの場合は再試行
        await new Promise(resolve => setTimeout(resolve, 5000));
        return this.analyzeSentimentWithCache(text);
      }
      
      throw error;
    }
  }
}
```

## まとめ

本記事では、**GCP Natural Language AIをTypeScriptで活用したテキスト解析システム**の構築方法を解説しました。

**実装のポイント**：

> ✅ **感情分析**: 顧客レビューやソーシャルメディア監視に活用
> ✅ **エンティティ抽出**: 重要な人物・組織・場所の自動識別  
> ✅ **構文解析**: 品詞分析による高度なテキスト理解
> ✅ **REST API化**: マイクロサービスとして他システムと連携
> ✅ **パフォーマンス最適化**: キャッシング・レート制限・エラーハンドリング

**運用での注意点**：

> 💡 **コスト管理**: APIコール数の監視とキャッシュ活用
> 💡 **精度向上**: ドメイン特化データでの検証・調整
> 💡 **プライバシー**: 機密データの適切な処理

これで、本格的なテキスト解析システムの基盤が完成です。ビジネスニーズに応じてカスタマイズし、自然言語処理の力を活用してくださいね！