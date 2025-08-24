## もうAIの判断で迷わない！GCP Explainable AIをTypeScriptで完全制御する機械学習解釈ガイド

## はじめに

機械学習モデルを本格的に運用し始めると、必ずこんな疑問が浮かびませんか？

> 「なぜこの予測結果になったの？」
> 「モデルの判断根拠を説明する必要があるけど...」
> 「TypeScriptで機械学習の解釈性を実装できる？」

本記事では、**GCP Explainable AIを使ってTypeScriptでAIの判断根拠を可視化・説明する方法**を、実際のコード例とともに解説します。

## GCP Explainable AIとは

GCP Explainable AIは、**機械学習モデルの予測結果の根拠を理解するためのサービス**です。

| 機能 | 説明 | 適用場面 |
|------|------|----------|
| **Feature-based説明** | 各特徴量の寄与度を分析 | 表形式データ、画像分類 |
| **Example-based説明** | 類似した訓練データを特定 | 異常検知、データ品質改善 |
| **複数の解釈手法** | Shapley値、勾配統合、XRAI | モデルタイプに応じた最適化 |

## TypeScript環境のセットアップ

### 1\. 必要なパッケージのインストール

```bash
npm install @google-cloud/aiplatform @google-cloud/storage
npm install @types/node dotenv
```

### 2\. 環境設定

```typescript:src/config/gcp-config.ts
import * as dotenv from 'dotenv';

dotenv.config();

export const GCP_CONFIG = {
  projectId: process.env.GOOGLE_CLOUD_PROJECT_ID!,
  location: process.env.GOOGLE_CLOUD_LOCATION || 'us-central1',
  keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
  
  // Vertex AI設定
  vertexAI: {
    apiEndpoint: `${process.env.GOOGLE_CLOUD_LOCATION || 'us-central1'}-aiplatform.googleapis.com`
  }
} as const;

// 環境変数の検証
export function validateConfig(): void {
  const required = ['GOOGLE_CLOUD_PROJECT_ID'];
  
  for (const key of required) {
    if (!process.env[key]) {
      throw new Error(`必須の環境変数が設定されていません: ${key}`);
    }
  }
}
```

### 3\. Vertex AI Clientの初期化

```typescript:src/services/vertex-ai-client.ts
import { PredictionServiceClient } from '@google-cloud/aiplatform';
import { GCP_CONFIG, validateConfig } from '../config/gcp-config';

export class VertexAIClient {
  private predictionClient: PredictionServiceClient;
  
  constructor() {
    validateConfig();
    
    this.predictionClient = new PredictionServiceClient({
      apiEndpoint: GCP_CONFIG.vertexAI.apiEndpoint,
      keyFilename: GCP_CONFIG.keyFilename
    });
  }
  
  /**
   * エンドポイントのパスを生成
   */
  getEndpointPath(modelId: string): string {
    return this.predictionClient.endpointPath(
      GCP_CONFIG.projectId,
      GCP_CONFIG.location,
      modelId
    );
  }
  
  /**
   * 予測クライアントを取得
   */
  getPredictionClient(): PredictionServiceClient {
    return this.predictionClient;
  }
}
```

## Explainable AI実装の基本

### 1\. Feature Attribution（特徴量寄与度分析）

```typescript:src/explainable/feature-attribution.ts
import { VertexAIClient } from '../services/vertex-ai-client';
import { GCP_CONFIG } from '../config/gcp-config';

export interface FeatureAttributionConfig {
  method: 'sampled-shapley' | 'integrated-gradients' | 'xrai';
  pathCount?: number;
  stepCount?: number;
}

export interface AttributionResult {
  featureAttributions: {
    featureName: string;
    attribution: number;
    baselineValue?: number;
  }[];
  outputName: string;
  approxError?: number;
}

export class FeatureAttributionAnalyzer {
  private vertexClient: VertexAIClient;
  
  constructor() {
    this.vertexClient = new VertexAIClient();
  }
  
  /**
   * 表形式データの特徴量寄与度を分析（Sampled Shapley）
   */
  async analyzeTabularFeatures(
    modelEndpoint: string,
    instances: Record<string, any>[],
    config: FeatureAttributionConfig = { method: 'sampled-shapley' }
  ): Promise<AttributionResult[]> {
    const client = this.vertexClient.getPredictionClient();
    
    try {
      const explanationSpec = {
        parameters: this.buildExplanationParameters(config),
        metadata: {
          inputs: this.buildInputMetadata(instances[0])
        }
      };
      
      const request = {
        endpoint: this.vertexClient.getEndpointPath(modelEndpoint),
        instances: instances.map(instance => ({ value: instance })),
        explanationSpec
      };
      
      const [response] = await client.explain(request);
      
      return this.parseAttributionResponse(response);
    } catch (error) {
      console.error('特徴量寄与度分析でエラーが発生:', error);
      throw new Error(`特徴量分析に失敗しました: ${error}`);
    }
  }
  
  /**
   * 画像データの特徴量寄与度を分析（Integrated Gradients / XRAI）
   */
  async analyzeImageFeatures(
    modelEndpoint: string,
    imageBase64: string,
    config: FeatureAttributionConfig = { method: 'integrated-gradients', stepCount: 50 }
  ): Promise<AttributionResult[]> {
    const client = this.vertexClient.getPredictionClient();
    
    try {
      const explanationSpec = {
        parameters: this.buildExplanationParameters(config),
        metadata: {
          inputs: {
            'image': {
              inputTensorName: 'image',
              encoding: 'base64',
              modality: 'image'
            }
          }
        }
      };
      
      const request = {
        endpoint: this.vertexClient.getEndpointPath(modelEndpoint),
        instances: [{ 
          image: { b64: imageBase64 }
        }],
        explanationSpec
      };
      
      const [response] = await client.explain(request);
      
      return this.parseAttributionResponse(response);
    } catch (error) {
      console.error('画像特徴量分析でエラーが発生:', error);
      throw new Error(`画像分析に失敗しました: ${error}`);
    }
  }
  
  /**
   * 解釈パラメータを構築
   */
  private buildExplanationParameters(config: FeatureAttributionConfig) {
    switch (config.method) {
      case 'sampled-shapley':
        return {
          sampledShapleyAttribution: {
            pathCount: config.pathCount || 10
          }
        };
        
      case 'integrated-gradients':
        return {
          integratedGradientsAttribution: {
            stepCount: config.stepCount || 50,
            smoothGradConfig: {
              noiseSigma: 0.15,
              noisySampleCount: 25
            }
          }
        };
        
      case 'xrai':
        return {
          xraiAttribution: {
            stepCount: config.stepCount || 50,
            smoothGradConfig: {
              noiseSigma: 0.15,
              noisySampleCount: 25
            }
          }
        };
        
      default:
        throw new Error(`未対応の解釈手法: ${config.method}`);
    }
  }
  
  /**
   * 入力メタデータを構築
   */
  private buildInputMetadata(sampleInstance: Record<string, any>) {
    const metadata: Record<string, any> = {};
    
    for (const [key, value] of Object.entries(sampleInstance)) {
      metadata[key] = {
        inputTensorName: key,
        encoding: typeof value === 'string' ? 'text' : 'numeric',
        modality: typeof value === 'string' ? 'text' : 'numeric'
      };
    }
    
    return metadata;
  }
  
  /**
   * レスポンスを解析
   */
  private parseAttributionResponse(response: any): AttributionResult[] {
    const results: AttributionResult[] = [];
    
    if (response.explanations) {
      for (const explanation of response.explanations) {
        if (explanation.attributions) {
          for (const attribution of explanation.attributions) {
            results.push({
              featureAttributions: this.extractFeatureAttributions(attribution),
              outputName: attribution.outputName || 'default',
              approxError: attribution.approximationError
            });
          }
        }
      }
    }
    
    return results;
  }
  
  /**
   * 特徴量寄与度を抽出
   */
  private extractFeatureAttributions(attribution: any) {
    const features = [];
    
    if (attribution.featureAttributions) {
      for (const [featureName, value] of Object.entries(attribution.featureAttributions)) {
        features.push({
          featureName,
          attribution: typeof value === 'number' ? value : (value as any).value || 0,
          baselineValue: (value as any).baselineValue
        });
      }
    }
    
    return features.sort((a, b) => Math.abs(b.attribution) - Math.abs(a.attribution));
  }
}
```

### 2\. 事例ベース説明

```typescript:src/explainable/example-based.ts
import { VertexAIClient } from '../services/vertex-ai-client';

export interface ExampleBasedConfig {
  neighborCount?: number;
  returnEmbeddings?: boolean;
}

export interface SimilarExample {
  exampleId: string;
  similarity: number;
  embedding?: number[];
  metadata?: Record<string, any>;
}

export interface ExampleBasedResult {
  query: any;
  neighbors: SimilarExample[];
  explanationType: 'example-based';
}

export class ExampleBasedAnalyzer {
  private vertexClient: VertexAIClient;
  
  constructor() {
    this.vertexClient = new VertexAIClient();
  }
  
  /**
   * 類似事例を検索して説明を生成
   */
  async findSimilarExamples(
    modelEndpoint: string,
    queryInstance: Record<string, any>,
    config: ExampleBasedConfig = {}
  ): Promise<ExampleBasedResult> {
    const client = this.vertexClient.getPredictionClient();
    
    try {
      const explanationSpec = {
        parameters: {
          examples: {
            neighborCount: config.neighborCount || 5,
            returnEmbeddings: config.returnEmbeddings || false
          }
        }
      };
      
      const request = {
        endpoint: this.vertexClient.getEndpointPath(modelEndpoint),
        instances: [{ value: queryInstance }],
        explanationSpec
      };
      
      const [response] = await client.explain(request);
      
      return this.parseExampleBasedResponse(response, queryInstance);
    } catch (error) {
      console.error('類似事例検索でエラーが発生:', error);
      throw new Error(`事例ベース分析に失敗しました: ${error}`);
    }
  }
  
  /**
   * 異常検知のための事例分析
   */
  async detectAnomalies(
    modelEndpoint: string,
    instances: Record<string, any>[],
    threshold: number = 0.7
  ): Promise<{
    anomalies: Array<{
      instance: Record<string, any>;
      anomalyScore: number;
      explanation: ExampleBasedResult;
    }>;
    normal: Array<{
      instance: Record<string, any>;
      similarityScore: number;
    }>;
  }> {
    const results = {
      anomalies: [] as any[],
      normal: [] as any[]
    };
    
    for (const instance of instances) {
      const explanation = await this.findSimilarExamples(modelEndpoint, instance);
      
      // 最も類似度の高い事例の類似度を確認
      const maxSimilarity = Math.max(...explanation.neighbors.map(n => n.similarity));
      
      if (maxSimilarity < threshold) {
        results.anomalies.push({
          instance,
          anomalyScore: 1 - maxSimilarity,
          explanation
        });
      } else {
        results.normal.push({
          instance,
          similarityScore: maxSimilarity
        });
      }
    }
    
    return results;
  }
  
  /**
   * レスポンスを解析
   */
  private parseExampleBasedResponse(response: any, query: Record<string, any>): ExampleBasedResult {
    const neighbors: SimilarExample[] = [];
    
    if (response.explanations && response.explanations[0]?.neighbors) {
      for (const neighbor of response.explanations[0].neighbors) {
        neighbors.push({
          exampleId: neighbor.neighborId || `example_${neighbors.length}`,
          similarity: neighbor.neighborDistance || 0,
          embedding: neighbor.embedding,
          metadata: neighbor.metadata
        });
      }
    }
    
    return {
      query,
      neighbors: neighbors.sort((a, b) => b.similarity - a.similarity),
      explanationType: 'example-based'
    };
  }
}
```

## 実用的な解釈分析システム

### 1\. 統合解釈分析クラス

```typescript:src/explainable/explanation-system.ts
import { FeatureAttributionAnalyzer, AttributionResult } from './feature-attribution';
import { ExampleBasedAnalyzer, ExampleBasedResult } from './example-based';

export interface ExplanationRequest {
  modelEndpoint: string;
  instance: Record<string, any>;
  explanationTypes: Array<'feature-attribution' | 'example-based'>;
  config?: {
    attribution?: any;
    examples?: any;
  };
}

export interface ComprehensiveExplanation {
  instance: Record<string, any>;
  predictions: any[];
  featureAttribution?: AttributionResult[];
  exampleBased?: ExampleBasedResult;
  summary: {
    topFeatures: string[];
    confidence: number;
    explanation: string;
  };
}

export class ExplanationSystem {
  private featureAnalyzer: FeatureAttributionAnalyzer;
  private exampleAnalyzer: ExampleBasedAnalyzer;
  
  constructor() {
    this.featureAnalyzer = new FeatureAttributionAnalyzer();
    this.exampleAnalyzer = new ExampleBasedAnalyzer();
  }
  
  /**
   * 包括的な解釈分析を実行
   */
  async explainPrediction(request: ExplanationRequest): Promise<ComprehensiveExplanation> {
    const explanation: ComprehensiveExplanation = {
      instance: request.instance,
      predictions: [],
      summary: {
        topFeatures: [],
        confidence: 0,
        explanation: ''
      }
    };
    
    try {
      // Feature Attribution分析
      if (request.explanationTypes.includes('feature-attribution')) {
        explanation.featureAttribution = await this.featureAnalyzer.analyzeTabularFeatures(
          request.modelEndpoint,
          [request.instance],
          request.config?.attribution
        );
      }
      
      // Example-based分析
      if (request.explanationTypes.includes('example-based')) {
        explanation.exampleBased = await this.exampleAnalyzer.findSimilarExamples(
          request.modelEndpoint,
          request.instance,
          request.config?.examples
        );
      }
      
      // サマリー生成
      explanation.summary = this.generateSummary(explanation);
      
      return explanation;
    } catch (error) {
      console.error('解釈分析でエラーが発生:', error);
      throw new Error(`解釈分析に失敗しました: ${error}`);
    }
  }
  
  /**
   * バッチ解釈分析
   */
  async explainBatch(
    modelEndpoint: string,
    instances: Record<string, any>[],
    explanationTypes: Array<'feature-attribution' | 'example-based'> = ['feature-attribution']
  ): Promise<ComprehensiveExplanation[]> {
    const results: ComprehensiveExplanation[] = [];
    
    for (const instance of instances) {
      try {
        const explanation = await this.explainPrediction({
          modelEndpoint,
          instance,
          explanationTypes
        });
        results.push(explanation);
      } catch (error) {
        console.error(`インスタンス ${JSON.stringify(instance)} の解釈に失敗:`, error);
        // エラーが発生した場合も基本情報は残す
        results.push({
          instance,
          predictions: [],
          summary: {
            topFeatures: [],
            confidence: 0,
            explanation: 'エラーが発生しました'
          }
        });
      }
    }
    
    return results;
  }
  
  /**
   * 解釈結果のサマリーを生成
   */
  private generateSummary(explanation: ComprehensiveExplanation) {
    const summary = {
      topFeatures: [] as string[],
      confidence: 0,
      explanation: ''
    };
    
    if (explanation.featureAttribution && explanation.featureAttribution.length > 0) {
      const attributions = explanation.featureAttribution[0];
      
      // 上位特徴量を抽出（絶対値で）
      summary.topFeatures = attributions.featureAttributions
        .slice(0, 5)
        .map(f => f.featureName);
      
      // 信頼度を計算（寄与度の合計から）
      const totalAttribution = attributions.featureAttributions
        .reduce((sum, f) => sum + Math.abs(f.attribution), 0);
      summary.confidence = Math.min(totalAttribution / 10, 1.0);
      
      // 説明文を生成
      const topFeature = attributions.featureAttributions[0];
      const impact = topFeature.attribution > 0 ? '正の' : '負の';
      summary.explanation = 
        `最も影響が大きい特徴量は「${topFeature.featureName}」で、${impact}影響（寄与度: ${topFeature.attribution.toFixed(3)}）を与えています。`;
    }
    
    if (explanation.exampleBased) {
      const mostSimilar = explanation.exampleBased.neighbors[0];
      if (mostSimilar) {
        summary.explanation += ` 最も類似した事例との類似度は ${mostSimilar.similarity.toFixed(3)} です。`;
      }
    }
    
    return summary;
  }
}
```

### 2\. 可視化とレポート生成

```typescript:src/visualization/explanation-reporter.ts
import { ComprehensiveExplanation } from '../explainable/explanation-system';
import * as fs from 'fs';
import * as path from 'path';

export interface VisualizationOptions {
  format: 'html' | 'json' | 'csv';
  includeCharts: boolean;
  outputPath: string;
}

export class ExplanationReporter {
  
  /**
   * 解釈結果をHTMLレポートとして出力
   */
  async generateHTMLReport(
    explanations: ComprehensiveExplanation[],
    options: VisualizationOptions
  ): Promise<string> {
    const htmlContent = `
    <!DOCTYPE html>
    <html lang="ja">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>AI解釈レポート</title>
      <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
      <style>
        body { font-family: 'Helvetica Neue', Arial, sans-serif; margin: 40px; background-color: #f8f9fa; }
        .container { max-width: 1200px; margin: 0 auto; background: white; padding: 30px; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        .explanation { margin-bottom: 40px; padding: 20px; border: 1px solid #e0e0e0; border-radius: 8px; background: #fafafa; }
        .feature-list { list-style: none; padding: 0; }
        .feature-item { padding: 8px 0; border-bottom: 1px solid #eee; }
        .positive { color: #28a745; font-weight: bold; }
        .negative { color: #dc3545; font-weight: bold; }
        .chart-container { width: 100%; height: 300px; margin: 20px 0; }
        .summary-box { background: #e7f3ff; padding: 15px; border-left: 4px solid #007bff; margin: 10px 0; }
      </style>
    </head>
    <body>
      <div class="container">
        <h1>🤖 AI解釈レポート</h1>
        <p>生成日時: ${new Date().toLocaleString('ja-JP')}</p>
        ${this.generateHTMLContent(explanations)}
      </div>
    </body>
    </html>`;
    
    const outputFile = path.join(options.outputPath, 'explanation_report.html');
    await fs.promises.writeFile(outputFile, htmlContent, 'utf-8');
    
    return outputFile;
  }
  
  /**
   * CSVレポート生成
   */
  async generateCSVReport(
    explanations: ComprehensiveExplanation[],
    outputPath: string
  ): Promise<string> {
    const csvHeaders = [
      'インスタンス番号',
      '上位特徴量',
      '信頼度',
      '説明',
      '最大寄与度',
      '最小寄与度'
    ];
    
    const csvRows = explanations.map((exp, index) => {
      const topFeatures = exp.summary.topFeatures.join('; ');
      const maxAttribution = exp.featureAttribution?.[0]?.featureAttributions[0]?.attribution || 0;
      const minAttribution = exp.featureAttribution?.[0]?.featureAttributions.slice(-1)[0]?.attribution || 0;
      
      return [
        index + 1,
        `"${topFeatures}"`,
        exp.summary.confidence.toFixed(3),
        `"${exp.summary.explanation}"`,
        maxAttribution.toFixed(3),
        minAttribution.toFixed(3)
      ].join(',');
    });
    
    const csvContent = [csvHeaders.join(','), ...csvRows].join('\n');
    
    const outputFile = path.join(outputPath, 'explanation_data.csv');
    await fs.promises.writeFile(outputFile, csvContent, 'utf-8');
    
    return outputFile;
  }
  
  /**
   * HTMLコンテンツ生成
   */
  private generateHTMLContent(explanations: ComprehensiveExplanation[]): string {
    return explanations.map((exp, index) => `
      <div class="explanation">
        <h2>📊 予測事例 #${index + 1}</h2>
        
        <div class="summary-box">
          <h3>🎯 サマリー</h3>
          <p><strong>信頼度:</strong> ${(exp.summary.confidence * 100).toFixed(1)}%</p>
          <p><strong>説明:</strong> ${exp.summary.explanation}</p>
        </div>
        
        ${exp.featureAttribution ? this.generateFeatureAttributionHTML(exp.featureAttribution[0]) : ''}
        
        ${exp.exampleBased ? this.generateExampleBasedHTML(exp.exampleBased) : ''}
        
        <details>
          <summary>📋 入力データ詳細</summary>
          <pre>${JSON.stringify(exp.instance, null, 2)}</pre>
        </details>
      </div>
    `).join('');
  }
  
  /**
   * 特徴量寄与度のHTML生成
   */
  private generateFeatureAttributionHTML(attribution: any): string {
    const features = attribution.featureAttributions.slice(0, 10);
    
    return `
      <div>
        <h3>🔍 特徴量寄与度分析</h3>
        <ul class="feature-list">
          ${features.map(f => `
            <li class="feature-item">
              <strong>${f.featureName}:</strong> 
              <span class="${f.attribution > 0 ? 'positive' : 'negative'}">
                ${f.attribution > 0 ? '+' : ''}${f.attribution.toFixed(4)}
              </span>
            </li>
          `).join('')}
        </ul>
      </div>
    `;
  }
  
  /**
   * 事例ベース説明のHTML生成
   */
  private generateExampleBasedHTML(exampleBased: any): string {
    const neighbors = exampleBased.neighbors.slice(0, 5);
    
    return `
      <div>
        <h3>👥 類似事例分析</h3>
        <ul class="feature-list">
          ${neighbors.map((neighbor, idx) => `
            <li class="feature-item">
              <strong>類似事例 #${idx + 1}:</strong> 
              類似度 ${neighbor.similarity.toFixed(3)}
              ${neighbor.metadata ? ` (${JSON.stringify(neighbor.metadata)})` : ''}
            </li>
          `).join('')}
        </ul>
      </div>
    `;
  }
}
```

## 実践的な使用例

### 1\. 信用スコアリングモデルの解釈

```typescript:src/examples/credit-scoring.ts
import { ExplanationSystem } from '../explainable/explanation-system';
import { ExplanationReporter } from '../visualization/explanation-reporter';

export class CreditScoringExplainer {
  private explanationSystem: ExplanationSystem;
  private reporter: ExplanationReporter;
  
  constructor() {
    this.explanationSystem = new ExplanationSystem();
    this.reporter = new ExplanationReporter();
  }
  
  /**
   * 信用スコアの判定根拠を解釈
   */
  async explainCreditDecision(
    modelEndpoint: string,
    applicantData: {
      age: number;
      income: number;
      debtToIncomeRatio: number;
      creditHistory: number;
      employmentYears: number;
      loanAmount: number;
    }
  ) {
    console.log('💳 信用スコアリングの解釈分析を実行中...');
    
    const explanation = await this.explanationSystem.explainPrediction({
      modelEndpoint,
      instance: applicantData,
      explanationTypes: ['feature-attribution', 'example-based'],
      config: {
        attribution: { method: 'sampled-shapley', pathCount: 20 },
        examples: { neighborCount: 5 }
      }
    });
    
    // 結果の解釈
    console.log('\n📊 信用判定の解釈結果:');
    console.log(`信頼度: ${(explanation.summary.confidence * 100).toFixed(1)}%`);
    console.log(`主要な判定要因: ${explanation.summary.topFeatures.join(', ')}`);
    console.log(`説明: ${explanation.summary.explanation}`);
    
    // 詳細分析
    if (explanation.featureAttribution) {
      console.log('\n🔍 各要因の影響度:');
      explanation.featureAttribution[0].featureAttributions.forEach(f => {
        const impact = f.attribution > 0 ? '承認寄与' : '拒否寄与';
        console.log(`  ${f.featureName}: ${f.attribution.toFixed(4)} (${impact})`);
      });
    }
    
    // 類似事例の確認
    if (explanation.exampleBased) {
      console.log('\n👥 類似する過去事例:');
      explanation.exampleBased.neighbors.forEach((neighbor, idx) => {
        console.log(`  事例${idx + 1}: 類似度 ${neighbor.similarity.toFixed(3)}`);
      });
    }
    
    return explanation;
  }
  
  /**
   * バッチ処理で複数申請者を解釈
   */
  async analyzeCreditBatch(
    modelEndpoint: string,
    applicants: Array<Record<string, any>>
  ) {
    console.log(`📋 ${applicants.length}件の信用申請を一括解釈中...`);
    
    const explanations = await this.explanationSystem.explainBatch(
      modelEndpoint,
      applicants,
      ['feature-attribution']
    );
    
    // レポート生成
    const reportPath = await this.reporter.generateHTMLReport(explanations, {
      format: 'html',
      includeCharts: true,
      outputPath: './reports'
    });
    
    console.log(`📄 レポートが生成されました: ${reportPath}`);
    
    // 統計サマリー
    const approvalFactors = this.analyzeCommonFactors(explanations);
    console.log('\n📈 承認要因分析:');
    console.log(approvalFactors);
    
    return { explanations, reportPath, statistics: approvalFactors };
  }
  
  /**
   * 共通要因を分析
   */
  private analyzeCommonFactors(explanations: any[]) {
    const factorCounts: Record<string, { positive: number; negative: number; total: number }> = {};
    
    explanations.forEach(exp => {
      if (exp.featureAttribution && exp.featureAttribution[0]) {
        exp.featureAttribution[0].featureAttributions.forEach((f: any) => {
          if (!factorCounts[f.featureName]) {
            factorCounts[f.featureName] = { positive: 0, negative: 0, total: 0 };
          }
          
          factorCounts[f.featureName].total++;
          if (f.attribution > 0) {
            factorCounts[f.featureName].positive++;
          } else {
            factorCounts[f.featureName].negative++;
          }
        });
      }
    });
    
    return Object.entries(factorCounts)
      .map(([factor, counts]) => ({
        factor,
        positiveRate: counts.positive / counts.total,
        negativeRate: counts.negative / counts.total,
        frequency: counts.total
      }))
      .sort((a, b) => b.frequency - a.frequency);
  }
}
```

### 2\. 使用方法の実例

```typescript:src/examples/usage-example.ts
import { CreditScoringExplainer } from './credit-scoring';

async function main() {
  const explainer = new CreditScoringExplainer();
  
  // 単一事例の解釈
  const singleCase = await explainer.explainCreditDecision(
    'credit-model-endpoint-123',
    {
      age: 35,
      income: 75000,
      debtToIncomeRatio: 0.3,
      creditHistory: 720,
      employmentYears: 8,
      loanAmount: 250000
    }
  );
  
  // バッチ解釈
  const batchCases = [
    { age: 28, income: 45000, debtToIncomeRatio: 0.4, creditHistory: 650, employmentYears: 3, loanAmount: 180000 },
    { age: 42, income: 95000, debtToIncomeRatio: 0.2, creditHistory: 780, employmentYears: 12, loanAmount: 320000 },
    { age: 25, income: 38000, debtToIncomeRatio: 0.5, creditHistory: 580, employmentYears: 2, loanAmount: 150000 }
  ];
  
  const batchResults = await explainer.analyzeCreditBatch(
    'credit-model-endpoint-123',
    batchCases
  );
  
  console.log('\n✅ 解釈分析が完了しました！');
}

main().catch(console.error);
```

## まとめ

本記事では、**GCP Explainable AIをTypeScriptで活用し、AIモデルの判断根拠を可視化・説明する方法**を詳しく解説しました。

**✅ GCP Explainable AIのメリット**

>* **多様な解釈手法**: Shapley値、勾配統合、XRAIなど
>* **Feature + Example分析**: 特徴量寄与度と類似事例の両面から解釈
>* **スケーラブル**: バッチ処理による大量データ対応
>* **可視化対応**: HTMLレポートとCSV出力

**🎯 実装のポイント**

>* **適切な手法選択**: データタイプに応じた解釈手法の使い分け
>* **包括的分析**: Feature AttributionとExample-basedの組み合わせ
>* **自動レポート**: HTML/CSV形式での結果可視化
>* **エラーハンドリング**: 堅牢な例外処理とフォールバック

**実用的な応用場面**

>* **金融**: 信用スコアリング、リスク評価の根拠説明
>* **医療**: 診断支援AIの判断根拠の可視化
>* **製造**: 品質管理AIの異常検知理由の特定
>* **マーケティング**: 顧客行動予測の要因分析

TypeScriptでExplainable AIを実装することで、AIシステムの**透明性と信頼性を大幅に向上**させることができます。ぜひあなたのプロジェクトでも活用してみてください！