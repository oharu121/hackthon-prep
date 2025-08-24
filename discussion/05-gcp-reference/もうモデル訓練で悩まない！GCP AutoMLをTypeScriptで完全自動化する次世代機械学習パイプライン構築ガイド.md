# もうモデル訓練で悩まない！GCP AutoMLをTypeScriptで完全自動化する次世代機械学習パイプライン構築ガイド

## はじめに

「機械学習モデルを作りたいけど、専門知識がない」「データサイエンティストがいないチームでもAIを活用したい」そんな悩みを抱えていませんか？

Google Cloud AutoMLなら、コードを書かずに高品質な機械学習モデルを構築できます。さらにTypeScriptと組み合わせることで、モデルの訓練から予測まで完全に自動化されたMLパイプラインが実現できます。本記事では、AutoMLの基本概念から実践的な実装まで、すぐに現場で使える内容をお届けします！

## Google Cloud AutoMLとは

Google Cloud AutoMLは、機械学習の専門知識がなくても高品質なカスタムモデルを構築できるサービス群です。Googleが長年培ってきたニューラルアーキテクチャ検索技術により、データに最適なモデルを自動で設計・訓練します。

**🎯 AutoMLの主要サービス**

>* **AutoML Vision**: 画像分類・オブジェクト検出
>* **AutoML Natural Language**: テキスト分類・感情分析・エンティティ抽出
>* **AutoML Translation**: カスタム翻訳モデル
>* **AutoML Video Intelligence**: 動画分類・アクション認識
>* **AutoML Tables**: 構造化データの予測（現在はVertex AI Tablesに統合）

**従来のML開発 vs AutoML**

| 従来のアプローチ | Google Cloud AutoML |
|---|---|
| 専門知識が必須 | GUIベースの簡単操作 |
| モデル設計に時間がかかる | 自動でアーキテクチャ最適化 |
| ハイパーパラメータ調整が複雑 | 自動チューニング |
| インフラ構築・管理が必要 | フルマネージドサービス |
| 大量の実験とトライアル | 効率的な学習プロセス |

## 環境セットアップ

### 1\. GCPプロジェクトとAPIの有効化

```bash
# 必要なAPIを有効化
gcloud services enable automl.googleapis.com
gcloud services enable storage-api.googleapis.com
gcloud services enable aiplatform.googleapis.com
```

### 2\. TypeScriptプロジェクトのセットアップ

```bash
npm init -y
npm install @google-cloud/automl @google-cloud/storage
npm install --save-dev typescript @types/node ts-node
```

### 3\. 認証設定

```bash
# サービスアカウント作成（AutoMLとStorageの権限が必要）
gcloud iam service-accounts create automl-service-account
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:automl-service-account@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/automl.admin"
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:automl-service-account@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/storage.admin"
```

## AutoML基盤クライアントの実装

```typescript:src/automl-base-client.ts
import { AutoMlClient } from '@google-cloud/automl';
import { Storage } from '@google-cloud/storage';

export interface AutoMLConfig {
  projectId: string;
  location: string;
  bucketName: string;
  keyFilename?: string;
}

export abstract class AutoMLBaseClient {
  protected client: AutoMlClient;
  protected storage: Storage;
  protected projectId: string;
  protected location: string;
  protected bucketName: string;

  constructor(config: AutoMLConfig) {
    this.projectId = config.projectId;
    this.location = config.location;
    this.bucketName = config.bucketName;

    this.client = new AutoMlClient({
      keyFilename: config.keyFilename
    });

    this.storage = new Storage({
      projectId: config.projectId,
      keyFilename: config.keyFilename
    });
  }

  // Cloud Storageへのデータアップロード
  protected async uploadToGCS(
    localFilePath: string,
    gcsFilePath: string
  ): Promise<string> {
    try {
      const bucket = this.storage.bucket(this.bucketName);
      await bucket.upload(localFilePath, {
        destination: gcsFilePath
      });

      return `gs://${this.bucketName}/${gcsFilePath}`;
    } catch (error) {
      throw new Error(`GCSアップロードエラー: ${error}`);
    }
  }

  // データセット一覧の取得
  async listDatasets(): Promise<any[]> {
    const projectLocation = this.client.locationPath(
      this.projectId,
      this.location
    );

    const [response] = await this.client.listDatasets({
      parent: projectLocation
    });

    return response;
  }

  // モデル一覧の取得
  async listModels(): Promise<any[]> {
    const projectLocation = this.client.locationPath(
      this.projectId,
      this.location
    );

    const [response] = await this.client.listModels({
      parent: projectLocation
    });

    return response;
  }
}
```

## AutoML Vision実装

```typescript:src/automl-vision.ts
import { AutoMLBaseClient, AutoMLConfig } from './automl-base-client';
import * as fs from 'fs';
import * as path from 'path';

interface ImageData {
  imagePath: string;
  label: string;
}

interface TrainingProgress {
  modelId: string;
  status: 'PENDING' | 'RUNNING' | 'SUCCEEDED' | 'FAILED' | 'CANCELLED';
  progress: number;
  estimatedTime?: string;
}

export class AutoMLVision extends AutoMLBaseClient {
  constructor(config: AutoMLConfig) {
    super(config);
  }

  // 画像分類データセットの作成
  async createImageDataset(
    displayName: string,
    classification: 'MULTICLASS' | 'MULTILABEL' = 'MULTICLASS'
  ): Promise<string> {
    try {
      const projectLocation = this.client.locationPath(
        this.projectId,
        this.location
      );

      const dataset = {
        displayName: displayName,
        imageClassificationDatasetMetadata: {
          classificationType: classification
        }
      };

      const [operation] = await this.client.createDataset({
        parent: projectLocation,
        dataset: dataset
      });

      console.log('データセット作成中...');
      const [response] = await operation.promise();
      
      const datasetId = response.name?.split('/').pop();
      console.log(`データセット作成完了: ${datasetId}`);
      
      return datasetId!;
    } catch (error) {
      throw new Error(`データセット作成エラー: ${error}`);
    }
  }

  // 画像データのアップロードとCSVファイル生成
  async uploadImageData(
    imageDataList: ImageData[],
    datasetName: string
  ): Promise<string> {
    const csvContent: string[] = [];
    
    for (const imageData of imageDataList) {
      try {
        // 画像をCloud Storageにアップロード
        const fileName = path.basename(imageData.imagePath);
        const gcsPath = `datasets/${datasetName}/images/${fileName}`;
        const gcsUri = await this.uploadToGCS(imageData.imagePath, gcsPath);
        
        // CSV行を作成（GCS URI, ラベル）
        csvContent.push(`${gcsUri},${imageData.label}`);
        
        console.log(`アップロード完了: ${fileName} -> ${imageData.label}`);
      } catch (error) {
        console.error(`画像アップロードエラー: ${imageData.imagePath}`, error);
      }
    }

    // CSVファイルを作成・アップロード
    const csvFileName = `datasets/${datasetName}/data.csv`;
    const localCsvPath = `/tmp/${datasetName}_data.csv`;
    
    fs.writeFileSync(localCsvPath, csvContent.join('\n'));
    const csvGcsUri = await this.uploadToGCS(localCsvPath, csvFileName);
    
    // ローカルCSVファイルを削除
    fs.unlinkSync(localCsvPath);
    
    console.log(`CSVファイル作成完了: ${csvGcsUri}`);
    return csvGcsUri;
  }

  // データセットへのデータインポート
  async importData(datasetId: string, csvGcsUri: string): Promise<void> {
    try {
      const datasetPath = this.client.datasetPath(
        this.projectId,
        this.location,
        datasetId
      );

      const inputConfig = {
        gcsSource: {
          inputUris: [csvGcsUri]
        }
      };

      console.log('データインポート開始...');
      const [operation] = await this.client.importData({
        name: datasetPath,
        inputConfig: inputConfig
      });

      await operation.promise();
      console.log('データインポート完了');
    } catch (error) {
      throw new Error(`データインポートエラー: ${error}`);
    }
  }

  // モデルの訓練
  async trainModel(
    datasetId: string,
    modelDisplayName: string,
    trainBudgetMilliNodeHours: number = 1000
  ): Promise<string> {
    try {
      const datasetPath = this.client.datasetPath(
        this.projectId,
        this.location,
        datasetId
      );

      const projectLocation = this.client.locationPath(
        this.projectId,
        this.location
      );

      const model = {
        displayName: modelDisplayName,
        datasetId: datasetId,
        imageClassificationModelMetadata: {
          trainBudgetMilliNodeHours: trainBudgetMilliNodeHours
        }
      };

      console.log('モデル訓練開始...');
      const [operation] = await this.client.createModel({
        parent: projectLocation,
        model: model
      });

      // 訓練は長時間かかるため、オペレーションIDを返す
      const operationId = operation.name;
      console.log(`訓練開始（オペレーションID: ${operationId}）`);
      
      return operationId!;
    } catch (error) {
      throw new Error(`モデル訓練エラー: ${error}`);
    }
  }

  // 訓練進捗の確認
  async getTrainingProgress(operationId: string): Promise<TrainingProgress> {
    try {
      const [operation] = await this.client.operationsClient.getOperation({
        name: operationId
      });

      let status: TrainingProgress['status'] = 'PENDING';
      if (operation.done) {
        status = operation.error ? 'FAILED' : 'SUCCEEDED';
      } else {
        status = 'RUNNING';
      }

      return {
        modelId: operationId,
        status: status,
        progress: operation.done ? 100 : 50, // 簡易進捗表示
        estimatedTime: operation.done ? undefined : '推定1-2時間'
      };
    } catch (error) {
      throw new Error(`訓練進捗確認エラー: ${error}`);
    }
  }

  // 画像予測の実行
  async predictImage(
    modelId: string,
    imagePath: string
  ): Promise<Array<{ label: string; confidence: number }>> {
    try {
      const modelPath = this.client.modelPath(
        this.projectId,
        this.location,
        modelId
      );

      // 画像をBase64エンコード
      const imageContent = fs.readFileSync(imagePath);
      const base64Image = imageContent.toString('base64');

      const payload = {
        image: {
          imageBytes: base64Image
        }
      };

      const [response] = await this.client.predict({
        name: modelPath,
        payload: payload
      });

      // 予測結果の解析
      const predictions: Array<{ label: string; confidence: number }> = [];
      
      if (response.payload) {
        response.payload.forEach((result: any) => {
          const classification = result.classification;
          if (classification) {
            predictions.push({
              label: classification.displayName,
              confidence: classification.score
            });
          }
        });
      }

      return predictions.sort((a, b) => b.confidence - a.confidence);
    } catch (error) {
      throw new Error(`画像予測エラー: ${error}`);
    }
  }
}
```

## AutoML Natural Language実装

```typescript:src/automl-natural-language.ts
import { AutoMLBaseClient, AutoMLConfig } from './automl-base-client';
import * as fs from 'fs';

interface TextData {
  text: string;
  label: string;
}

interface SentimentResult {
  sentiment: 'positive' | 'negative' | 'neutral';
  confidence: number;
  magnitude: number;
}

export class AutoMLNaturalLanguage extends AutoMLBaseClient {
  constructor(config: AutoMLConfig) {
    super(config);
  }

  // テキスト分類データセットの作成
  async createTextDataset(
    displayName: string,
    classificationType: 'SINGLE_LABEL' | 'MULTI_LABEL' = 'SINGLE_LABEL'
  ): Promise<string> {
    try {
      const projectLocation = this.client.locationPath(
        this.projectId,
        this.location
      );

      const dataset = {
        displayName: displayName,
        textClassificationDatasetMetadata: {
          classificationType: classificationType
        }
      };

      const [operation] = await this.client.createDataset({
        parent: projectLocation,
        dataset: dataset
      });

      const [response] = await operation.promise();
      const datasetId = response.name?.split('/').pop();
      
      console.log(`テキスト分類データセット作成完了: ${datasetId}`);
      return datasetId!;
    } catch (error) {
      throw new Error(`テキストデータセット作成エラー: ${error}`);
    }
  }

  // テキストデータのアップロード
  async uploadTextData(
    textDataList: TextData[],
    datasetName: string
  ): Promise<string> {
    // JSONL形式でデータを準備
    const jsonlContent = textDataList.map(data => 
      JSON.stringify({
        textSnippet: { content: data.text },
        displayName: data.label
      })
    ).join('\n');

    // JSONL ファイルをCloud Storageにアップロード
    const jsonlFileName = `datasets/${datasetName}/data.jsonl`;
    const localJsonlPath = `/tmp/${datasetName}_data.jsonl`;
    
    fs.writeFileSync(localJsonlPath, jsonlContent);
    const jsonlGcsUri = await this.uploadToGCS(localJsonlPath, jsonlFileName);
    
    fs.unlinkSync(localJsonlPath);
    
    console.log(`JSONLファイル作成完了: ${jsonlGcsUri}`);
    return jsonlGcsUri;
  }

  // テキスト分類モデルの訓練
  async trainTextModel(
    datasetId: string,
    modelDisplayName: string
  ): Promise<string> {
    try {
      const projectLocation = this.client.locationPath(
        this.projectId,
        this.location
      );

      const model = {
        displayName: modelDisplayName,
        datasetId: datasetId,
        textClassificationModelMetadata: {}
      };

      console.log('テキスト分類モデル訓練開始...');
      const [operation] = await this.client.createModel({
        parent: projectLocation,
        model: model
      });

      const operationId = operation.name;
      console.log(`テキストモデル訓練開始（オペレーションID: ${operationId}）`);
      
      return operationId!;
    } catch (error) {
      throw new Error(`テキストモデル訓練エラー: ${error}`);
    }
  }

  // テキスト分類予測
  async predictText(
    modelId: string,
    text: string
  ): Promise<Array<{ label: string; confidence: number }>> {
    try {
      const modelPath = this.client.modelPath(
        this.projectId,
        this.location,
        modelId
      );

      const payload = {
        textSnippet: {
          content: text
        }
      };

      const [response] = await this.client.predict({
        name: modelPath,
        payload: payload
      });

      const predictions: Array<{ label: string; confidence: number }> = [];
      
      if (response.payload) {
        response.payload.forEach((result: any) => {
          const classification = result.classification;
          if (classification) {
            predictions.push({
              label: classification.displayName,
              confidence: classification.score
            });
          }
        });
      }

      return predictions.sort((a, b) => b.confidence - a.confidence);
    } catch (error) {
      throw new Error(`テキスト予測エラー: ${error}`);
    }
  }

  // 感情分析（事前訓練済みモデル使用）
  async analyzeSentiment(text: string): Promise<SentimentResult> {
    try {
      // 感情分析は事前訓練済みのNatural Language APIを使用
      const { LanguageServiceClient } = require('@google-cloud/language');
      const languageClient = new LanguageServiceClient();

      const document = {
        content: text,
        type: 'PLAIN_TEXT'
      };

      const [result] = await languageClient.analyzeSentiment({
        document: document
      });

      const sentiment = result.documentSentiment;
      let sentimentLabel: 'positive' | 'negative' | 'neutral' = 'neutral';
      
      if (sentiment.score > 0.1) {
        sentimentLabel = 'positive';
      } else if (sentiment.score < -0.1) {
        sentimentLabel = 'negative';
      }

      return {
        sentiment: sentimentLabel,
        confidence: Math.abs(sentiment.score),
        magnitude: sentiment.magnitude
      };
    } catch (error) {
      throw new Error(`感情分析エラー: ${error}`);
    }
  }
}
```

## AutoML Tables実装（Vertex AI Tables）

**構造化データの予測モデル**

```typescript:src/automl-tables.ts
import { AutoMLBaseClient, AutoMLConfig } from './automl-base-client';
import * as fs from 'fs';

interface ColumnSpec {
  columnName: string;
  dataType: 'STRING' | 'NUMBER' | 'FLOAT' | 'TIMESTAMP' | 'CATEGORY';
  nullable: boolean;
}

interface TabularDataConfig {
  targetColumn: string;
  columns: ColumnSpec[];
  predictionType: 'CLASSIFICATION' | 'REGRESSION';
}

export class AutoMLTables extends AutoMLBaseClient {
  constructor(config: AutoMLConfig) {
    super(config);
  }

  // テーブル形式データセットの作成
  async createTabularDataset(
    displayName: string,
    config: TabularDataConfig
  ): Promise<string> {
    try {
      const projectLocation = this.client.locationPath(
        this.projectId,
        this.location
      );

      const dataset = {
        displayName: displayName,
        tablesDatasetMetadata: {
          targetColumnSpec: {
            name: config.targetColumn
          },
          primaryTableSpec: {
            columnSpecs: config.columns.map(col => ({
              name: col.columnName,
              dataType: col.dataType,
              nullable: col.nullable
            }))
          }
        }
      };

      const [operation] = await this.client.createDataset({
        parent: projectLocation,
        dataset: dataset
      });

      const [response] = await operation.promise();
      const datasetId = response.name?.split('/').pop();
      
      console.log(`テーブル形式データセット作成完了: ${datasetId}`);
      return datasetId!;
    } catch (error) {
      throw new Error(`テーブル形式データセット作成エラー: ${error}`);
    }
  }

  // CSVデータの準備とアップロード
  async uploadCSVData(
    csvFilePath: string,
    datasetName: string
  ): Promise<string> {
    const fileName = `datasets/${datasetName}/training_data.csv`;
    const gcsUri = await this.uploadToGCS(csvFilePath, fileName);
    
    console.log(`CSVデータアップロード完了: ${gcsUri}`);
    return gcsUri;
  }

  // テーブル形式データのモデル訓練
  async trainTabularModel(
    datasetId: string,
    modelDisplayName: string,
    targetColumn: string,
    predictionType: 'CLASSIFICATION' | 'REGRESSION',
    trainBudgetMilliNodeHours: number = 1000
  ): Promise<string> {
    try {
      const projectLocation = this.client.locationPath(
        this.projectId,
        this.location
      );

      const model = {
        displayName: modelDisplayName,
        datasetId: datasetId,
        tablesModelMetadata: {
          trainBudgetMilliNodeHours: trainBudgetMilliNodeHours,
          targetColumnSpec: {
            name: targetColumn
          }
        }
      };

      console.log('テーブル形式モデル訓練開始...');
      const [operation] = await this.client.createModel({
        parent: projectLocation,
        model: model
      });

      const operationId = operation.name;
      console.log(`テーブルモデル訓練開始（オペレーションID: ${operationId}）`);
      
      return operationId!;
    } catch (error) {
      throw new Error(`テーブルモデル訓練エラー: ${error}`);
    }
  }

  // テーブル形式データの予測
  async predictTabular(
    modelId: string,
    inputData: { [key: string]: any }
  ): Promise<{ prediction: any; confidence?: number }> {
    try {
      const modelPath = this.client.modelPath(
        this.projectId,
        this.location,
        modelId
      );

      const payload = {
        row: {
          values: Object.values(inputData)
        }
      };

      const [response] = await this.client.predict({
        name: modelPath,
        payload: payload
      });

      if (response.payload && response.payload[0]) {
        const result = response.payload[0];
        return {
          prediction: result.tables?.value,
          confidence: result.tables?.score
        };
      }

      throw new Error('予期しない応答形式です');
    } catch (error) {
      throw new Error(`テーブル予測エラー: ${error}`);
    }
  }
}
```

## 統合MLパイプライン

**オーケストレーション層の実装**

```typescript:src/ml-pipeline-orchestrator.ts
import { AutoMLVision } from './automl-vision';
import { AutoMLNaturalLanguage } from './automl-natural-language';
import { AutoMLTables } from './automl-tables';
import { AutoMLConfig } from './automl-base-client';

interface PipelineConfig extends AutoMLConfig {
  notifications?: {
    email?: string;
    webhook?: string;
  };
}

interface PipelineStep {
  id: string;
  name: string;
  status: 'pending' | 'running' | 'completed' | 'failed';
  startTime?: Date;
  endTime?: Date;
  error?: string;
  result?: any;
}

export class MLPipelineOrchestrator {
  private vision: AutoMLVision;
  private nlp: AutoMLNaturalLanguage;
  private tables: AutoMLTables;
  private config: PipelineConfig;
  private steps: PipelineStep[] = [];

  constructor(config: PipelineConfig) {
    this.config = config;
    this.vision = new AutoMLVision(config);
    this.nlp = new AutoMLNaturalLanguage(config);
    this.tables = new AutoMLTables(config);
  }

  // パイプラインステップの追加
  private addStep(id: string, name: string): void {
    this.steps.push({
      id,
      name,
      status: 'pending'
    });
  }

  // ステップの更新
  private updateStep(
    id: string, 
    updates: Partial<PipelineStep>
  ): void {
    const step = this.steps.find(s => s.id === id);
    if (step) {
      Object.assign(step, updates);
    }
  }

  // 画像分類パイプラインの実行
  async runImageClassificationPipeline(
    datasetName: string,
    imageDataList: Array<{ imagePath: string; label: string }>,
    modelName: string
  ): Promise<string> {
    console.log('🚀 画像分類パイプライン開始');

    // Step 1: データセット作成
    this.addStep('dataset', 'データセット作成');
    this.updateStep('dataset', { status: 'running', startTime: new Date() });
    
    try {
      const datasetId = await this.vision.createImageDataset(datasetName);
      this.updateStep('dataset', { 
        status: 'completed', 
        endTime: new Date(),
        result: { datasetId }
      });

      // Step 2: データアップロード
      this.addStep('upload', 'データアップロード');
      this.updateStep('upload', { status: 'running', startTime: new Date() });
      
      const csvUri = await this.vision.uploadImageData(imageDataList, datasetName);
      
      // Step 3: データインポート
      await this.vision.importData(datasetId, csvUri);
      this.updateStep('upload', { 
        status: 'completed', 
        endTime: new Date(),
        result: { csvUri }
      });

      // Step 4: モデル訓練
      this.addStep('training', 'モデル訓練');
      this.updateStep('training', { status: 'running', startTime: new Date() });
      
      const operationId = await this.vision.trainModel(datasetId, modelName);
      this.updateStep('training', { 
        status: 'completed', 
        endTime: new Date(),
        result: { operationId }
      });

      console.log('✅ 画像分類パイプライン完了');
      return operationId;
    } catch (error) {
      const currentStep = this.steps.find(s => s.status === 'running');
      if (currentStep) {
        this.updateStep(currentStep.id, { 
          status: 'failed', 
          endTime: new Date(),
          error: error.message 
        });
      }
      throw error;
    }
  }

  // テキスト分類パイプラインの実行
  async runTextClassificationPipeline(
    datasetName: string,
    textDataList: Array<{ text: string; label: string }>,
    modelName: string
  ): Promise<string> {
    console.log('🚀 テキスト分類パイプライン開始');

    try {
      // データセット作成
      this.addStep('nlp-dataset', 'テキストデータセット作成');
      this.updateStep('nlp-dataset', { status: 'running', startTime: new Date() });
      
      const datasetId = await this.nlp.createTextDataset(datasetName);
      this.updateStep('nlp-dataset', { 
        status: 'completed', 
        endTime: new Date(),
        result: { datasetId }
      });

      // データアップロード
      this.addStep('nlp-upload', 'テキストデータアップロード');
      this.updateStep('nlp-upload', { status: 'running', startTime: new Date() });
      
      const jsonlUri = await this.nlp.uploadTextData(textDataList, datasetName);
      await this.nlp.importData(datasetId, jsonlUri);
      this.updateStep('nlp-upload', { 
        status: 'completed', 
        endTime: new Date(),
        result: { jsonlUri }
      });

      // モデル訓練
      this.addStep('nlp-training', 'テキストモデル訓練');
      this.updateStep('nlp-training', { status: 'running', startTime: new Date() });
      
      const operationId = await this.nlp.trainTextModel(datasetId, modelName);
      this.updateStep('nlp-training', { 
        status: 'completed', 
        endTime: new Date(),
        result: { operationId }
      });

      console.log('✅ テキスト分類パイプライン完了');
      return operationId;
    } catch (error) {
      console.error('❌ テキスト分類パイプライン失敗:', error);
      throw error;
    }
  }

  // パイプライン状況の取得
  getPipelineStatus(): {
    overallStatus: string;
    steps: PipelineStep[];
    progress: number;
  } {
    const completedSteps = this.steps.filter(s => s.status === 'completed').length;
    const failedSteps = this.steps.filter(s => s.status === 'failed').length;
    const totalSteps = this.steps.length;

    let overallStatus = 'running';
    if (failedSteps > 0) {
      overallStatus = 'failed';
    } else if (completedSteps === totalSteps && totalSteps > 0) {
      overallStatus = 'completed';
    } else if (totalSteps === 0) {
      overallStatus = 'not_started';
    }

    return {
      overallStatus,
      steps: this.steps,
      progress: totalSteps > 0 ? (completedSteps / totalSteps) * 100 : 0
    };
  }
}
```

## 実用的な使用例

### 1\. Eコマース商品画像分類システム

```typescript:src/ecommerce-image-classifier.ts
import { MLPipelineOrchestrator } from './ml-pipeline-orchestrator';
import * as path from 'path';
import * as fs from 'fs';

export class EcommerceImageClassifier {
  private orchestrator: MLPipelineOrchestrator;
  private modelId?: string;

  constructor(projectId: string, bucketName: string) {
    this.orchestrator = new MLPipelineOrchestrator({
      projectId,
      location: 'us-central1',
      bucketName
    });
  }

  // 商品カテゴリ分類モデルの訓練
  async trainProductCategoryModel(
    productImagesDir: string
  ): Promise<void> {
    // 画像ディレクトリから訓練データを準備
    const imageDataList = this.prepareTrainingData(productImagesDir);
    
    console.log(`📸 ${imageDataList.length}件の商品画像を準備`);

    // パイプライン実行
    const operationId = await this.orchestrator.runImageClassificationPipeline(
      'ecommerce-products',
      imageDataList,
      'product-category-classifier'
    );

    console.log('モデル訓練を開始しました。完了まで1-2時間程度かかります。');
    console.log(`操作ID: ${operationId}`);
  }

  // 商品画像の自動カテゴリ分類
  async classifyProductImage(
    imagePath: string
  ): Promise<{ category: string; confidence: number }> {
    if (!this.modelId) {
      throw new Error('モデルが訓練されていません');
    }

    const predictions = await this.orchestrator.vision.predictImage(
      this.modelId,
      imagePath
    );

    if (predictions.length === 0) {
      throw new Error('分類結果が取得できませんでした');
    }

    return {
      category: predictions[0].label,
      confidence: predictions[0].confidence
    };
  }

  // 訓練データの準備（ディレクトリ構造から自動生成）
  private prepareTrainingData(baseDir: string): Array<{imagePath: string; label: string}> {
    const imageData: Array<{imagePath: string; label: string}> = [];
    
    // カテゴリごとのディレクトリを走査
    const categoryDirs = fs.readdirSync(baseDir, { withFileTypes: true })
      .filter(dirent => dirent.isDirectory())
      .map(dirent => dirent.name);

    for (const category of categoryDirs) {
      const categoryPath = path.join(baseDir, category);
      const imageFiles = fs.readdirSync(categoryPath)
        .filter(file => /\.(jpg|jpeg|png|gif)$/i.test(file));

      for (const imageFile of imageFiles) {
        imageData.push({
          imagePath: path.join(categoryPath, imageFile),
          label: category
        });
      }
    }

    return imageData;
  }
}
```

### 2\. カスタマーレビュー感情分析システム

```typescript:src/review-sentiment-analyzer.ts
import { MLPipelineOrchestrator } from './ml-pipeline-orchestrator';

interface ReviewData {
  reviewText: string;
  rating: number;
  productId: string;
}

interface SentimentAnalysisResult {
  sentiment: 'positive' | 'negative' | 'neutral';
  confidence: number;
  suggestions?: string[];
}

export class ReviewSentimentAnalyzer {
  private orchestrator: MLPipelineOrchestrator;
  private modelId?: string;

  constructor(projectId: string, bucketName: string) {
    this.orchestrator = new MLPipelineOrchestrator({
      projectId,
      location: 'us-central1',
      bucketName
    });
  }

  // レビューデータからモデル訓練
  async trainSentimentModel(reviews: ReviewData[]): Promise<void> {
    // レビューデータを感情ラベル付きテキストに変換
    const textDataList = reviews.map(review => ({
      text: review.reviewText,
      label: this.ratingToSentiment(review.rating)
    }));

    console.log(`📝 ${textDataList.length}件のレビューデータで訓練開始`);

    const operationId = await this.orchestrator.runTextClassificationPipeline(
      'customer-reviews',
      textDataList,
      'review-sentiment-classifier'
    );

    console.log(`感情分析モデル訓練開始: ${operationId}`);
  }

  // レビューテキストの感情分析
  async analyzeReviewSentiment(
    reviewText: string
  ): Promise<SentimentAnalysisResult> {
    // カスタムモデルによる予測
    const predictions = await this.orchestrator.nlp.predictText(
      this.modelId!,
      reviewText
    );

    // 事前訓練済みモデルによる詳細分析
    const detailedSentiment = await this.orchestrator.nlp.analyzeSentiment(
      reviewText
    );

    const result: SentimentAnalysisResult = {
      sentiment: predictions[0]?.label as any || detailedSentiment.sentiment,
      confidence: predictions[0]?.confidence || detailedSentiment.confidence
    };

    // 改善提案の生成
    if (result.sentiment === 'negative') {
      result.suggestions = [
        '顧客サポートへの迅速な対応を検討',
        '商品品質の改善点を調査',
        'フォローアップメールの送信を推奨'
      ];
    }

    return result;
  }

  // レビューの一括分析
  async analyzeBatchReviews(
    reviews: string[]
  ): Promise<SentimentAnalysisResult[]> {
    const results: SentimentAnalysisResult[] = [];
    
    for (const review of reviews) {
      try {
        const result = await this.analyzeReviewSentiment(review);
        results.push(result);
      } catch (error) {
        console.error(`レビュー分析エラー: ${review.substring(0, 50)}...`, error);
        results.push({
          sentiment: 'neutral',
          confidence: 0
        });
      }
    }

    return results;
  }

  // 評価スコアを感情ラベルに変換
  private ratingToSentiment(rating: number): string {
    if (rating >= 4) return 'positive';
    if (rating <= 2) return 'negative';
    return 'neutral';
  }
}
```

## モニタリング・運用

```typescript:src/automl-monitor.ts
export class AutoMLMonitor {
  private metrics: Map<string, any> = new Map();

  // モデルパフォーマンスの記録
  recordPrediction(
    modelId: string,
    predictionType: string,
    responseTime: number,
    confidence: number,
    success: boolean
  ): void {
    const key = `${modelId}_${predictionType}`;
    
    if (!this.metrics.has(key)) {
      this.metrics.set(key, {
        totalPredictions: 0,
        successRate: 0,
        averageResponseTime: 0,
        averageConfidence: 0,
        errors: 0
      });
    }

    const stats = this.metrics.get(key);
    stats.totalPredictions++;
    
    if (success) {
      stats.averageResponseTime = 
        (stats.averageResponseTime + responseTime) / 2;
      stats.averageConfidence = 
        (stats.averageConfidence + confidence) / 2;
    } else {
      stats.errors++;
    }
    
    stats.successRate = 
      ((stats.totalPredictions - stats.errors) / stats.totalPredictions) * 100;
  }

  // パフォーマンスレポートの生成
  generateReport(): { [modelId: string]: any } {
    const report: { [key: string]: any } = {};
    
    for (const [key, stats] of this.metrics.entries()) {
      report[key] = {
        ...stats,
        healthStatus: this.getHealthStatus(stats)
      };
    }
    
    return report;
  }

  private getHealthStatus(stats: any): 'healthy' | 'warning' | 'critical' {
    if (stats.successRate < 80) return 'critical';
    if (stats.averageResponseTime > 5000) return 'warning';
    if (stats.averageConfidence < 0.7) return 'warning';
    return 'healthy';
  }
}
```

## まとめ

Google Cloud AutoMLとTypeScriptを組み合わせることで、以下のメリットが実現できます：

✅ **専門知識不要**: データサイエンスの知識がなくても高品質なモデルを構築
✅ **自動最適化**: Googleの技術によるアーキテクチャとハイパーパラメータの自動調整
✅ **統合環境**: 複数のAutoMLサービスを統一されたインターフェースで管理
✅ **本番対応**: スケーラブルで監視可能なMLパイプラインを構築
✅ **型安全性**: TypeScriptによる堅牢なコード品質と開発効率

**🔍 次のステップ**

>* Vertex AI Pipelinesとの連携でMLOps完全自動化
>* BigQueryとの連携で大規模データセット処理
>* Cloud Functionsでサーバーレス推論API構築
>* Monitoring・Alertingシステムの本格導入

AutoMLの力で、あなたのプロダクトに簡単にAI機能を追加できます。まずは小さなデータセットから始めて、AIの可能性を体感してみてください！