# もう画像解析で迷わない！GCP Vision AIをTypeScriptで完全制御する次世代画像認識システム構築ガイド

## はじめに

「アプリに画像認識機能を追加したいけど、どのサービスを選べばいいか分からない...」「OpenCVは難しそうだし、もっと簡単にテキスト抽出や物体検出ができるサービスはないの？」

そんな悩みを抱えている開発者の皆さんに朗報です！Google CloudのVision AIなら、画像からのテキスト抽出、顔検出、ラベル検出など、複雑な画像解析がTypeScriptでサクッと実装できるんです。

本記事では、GCP Vision AIをTypeScriptで実装する方法から、実際のWebアプリケーションへの組み込み方まで、現場ですぐ使える実装パターンを徹底解説します。

## GCP Vision AIとは？

**Vision AIの特徴**

GCP Vision AIは、Google Cloudが提供する機械学習ベースの画像解析サービスです。事前に訓練されたモデルを使用して、画像から様々な情報を抽出できます。

| 機能 | 説明 | 料金（1000リクエスト） |
|------|------|----------------------|
| **テキスト検出** | 画像内のテキストを抽出 | $1.50 |
| **ラベル検出** | 物体・概念を識別 | $1.50 |
| **顔検出** | 人物の顔と感情を検出 | $1.50 |
| **セーフサーチ** | 不適切コンテンツ判定 | $1.50 |
| **ロゴ検出** | ブランドロゴを認識 | $1.50 |
| **ランドマーク検出** | 有名な建造物を識別 | $1.50 |

**💡 Vision AIの強み**

>* 99%以上の高精度な画像認識
>* 60以上の言語でのテキスト抽出（OCR）対応
>* バッチ処理で大量画像の一括解析も可能
>* AutoMLでカスタムモデルも構築可能
>* 企業レベルのセキュリティとスケーラビリティ

## 環境構築

### 1\. GCPプロジェクトとVision APIの有効化

まず、GCP プロジェクトでVision APIを有効化しましょう。

```bash
# GCP CLIでVision API APIを有効化
gcloud services enable vision.googleapis.com

# 認証情報を設定
gcloud auth application-default login
```

### 2\. TypeScriptプロジェクトの初期化

```bash
mkdir vision-ai-app && cd vision-ai-app
npm init -y
npm install @google-cloud/vision dotenv
npm install -D @types/node typescript ts-node
```

### 3\. TypeScript設定ファイルの作成

```typescript:tsconfig.json
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
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

### 4\. 環境変数の設定

```.env
GOOGLE_CLOUD_PROJECT_ID=your-project-id
GOOGLE_APPLICATION_CREDENTIALS=./path/to/service-account-key.json
```

## Vision AI基本実装：画像解析クラスの作成

### 1\. Vision AIクライアントクラスの実装

```typescript:src/vision-ai-client.ts
import { ImageAnnotatorClient } from '@google-cloud/vision';
import * as fs from 'fs';
import * as path from 'path';

export interface VisionAnalysisResult {
  textAnnotations?: string[];
  labelAnnotations?: Array<{
    description: string;
    score: number;
  }>;
  faceAnnotations?: Array<{
    boundingPoly: any;
    confidence: number;
    joyLikelihood: string;
    sorrowLikelihood: string;
    angerLikelihood: string;
    surpriseLikelihood: string;
  }>;
  safeSearchAnnotation?: {
    adult: string;
    spoof: string;
    medical: string;
    violence: string;
    racy: string;
  };
  logoAnnotations?: Array<{
    description: string;
    score: number;
  }>;
}

export class VisionAIClient {
  private client: ImageAnnotatorClient;

  constructor() {
    this.client = new ImageAnnotatorClient({
      projectId: process.env.GOOGLE_CLOUD_PROJECT_ID,
      keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
    });
  }

  /**
   * 画像からテキストを抽出する（OCR）
   */
  async detectText(imagePath: string): Promise<string[]> {
    try {
      const [result] = await this.client.textDetection(imagePath);
      const detections = result.textAnnotations || [];
      
      return detections.map(text => text.description || '').filter(Boolean);
    } catch (error) {
      console.error('テキスト検出エラー:', error);
      throw error;
    }
  }

  /**
   * 画像内の物体・概念を検出する
   */
  async detectLabels(imagePath: string, maxResults: number = 10) {
    try {
      const [result] = await this.client.labelDetection({
        image: { source: { filename: imagePath } },
        maxResults,
      });

      const labels = result.labelAnnotations || [];
      return labels.map(label => ({
        description: label.description || '',
        score: label.score || 0,
      }));
    } catch (error) {
      console.error('ラベル検出エラー:', error);
      throw error;
    }
  }

  /**
   * 顔検出と感情分析
   */
  async detectFaces(imagePath: string) {
    try {
      const [result] = await this.client.faceDetection(imagePath);
      const faces = result.faceAnnotations || [];

      return faces.map(face => ({
        boundingPoly: face.boundingPoly,
        confidence: face.detectionConfidence || 0,
        joyLikelihood: face.joyLikelihood || 'UNKNOWN',
        sorrowLikelihood: face.sorrowLikelihood || 'UNKNOWN',
        angerLikelihood: face.angerLikelihood || 'UNKNOWN',
        surpriseLikelihood: face.surpriseLikelihood || 'UNKNOWN',
      }));
    } catch (error) {
      console.error('顔検出エラー:', error);
      throw error;
    }
  }

  /**
   * 包括的な画像解析（複数機能を一度に実行）
   */
  async analyzeImage(imagePath: string): Promise<VisionAnalysisResult> {
    try {
      const [result] = await this.client.annotateImage({
        image: { source: { filename: imagePath } },
        features: [
          { type: 'TEXT_DETECTION', maxResults: 50 },
          { type: 'LABEL_DETECTION', maxResults: 10 },
          { type: 'FACE_DETECTION', maxResults: 10 },
          { type: 'SAFE_SEARCH_DETECTION' },
          { type: 'LOGO_DETECTION', maxResults: 10 },
        ],
      });

      return {
        textAnnotations: result.textAnnotations?.map(text => text.description || '').filter(Boolean),
        labelAnnotations: result.labelAnnotations?.map(label => ({
          description: label.description || '',
          score: label.score || 0,
        })),
        faceAnnotations: result.faceAnnotations?.map(face => ({
          boundingPoly: face.boundingPoly,
          confidence: face.detectionConfidence || 0,
          joyLikelihood: face.joyLikelihood || 'UNKNOWN',
          sorrowLikelihood: face.sorrowLikelihood || 'UNKNOWN',
          angerLikelihood: face.angerLikelihood || 'UNKNOWN',
          surpriseLikelihood: face.surpriseLikelihood || 'UNKNOWN',
        })),
        safeSearchAnnotation: result.safeSearchAnnotation ? {
          adult: result.safeSearchAnnotation.adult || 'UNKNOWN',
          spoof: result.safeSearchAnnotation.spoof || 'UNKNOWN',
          medical: result.safeSearchAnnotation.medical || 'UNKNOWN',
          violence: result.safeSearchAnnotation.violence || 'UNKNOWN',
          racy: result.safeSearchAnnotation.racy || 'UNKNOWN',
        } : undefined,
        logoAnnotations: result.logoAnnotations?.map(logo => ({
          description: logo.description || '',
          score: logo.score || 0,
        })),
      };
    } catch (error) {
      console.error('画像解析エラー:', error);
      throw error;
    }
  }

  /**
   * Base64エンコードされた画像を解析する
   */
  async analyzeImageFromBase64(base64Image: string): Promise<VisionAnalysisResult> {
    try {
      const [result] = await this.client.annotateImage({
        image: { content: base64Image },
        features: [
          { type: 'TEXT_DETECTION', maxResults: 50 },
          { type: 'LABEL_DETECTION', maxResults: 10 },
          { type: 'FACE_DETECTION', maxResults: 10 },
          { type: 'SAFE_SEARCH_DETECTION' },
          { type: 'LOGO_DETECTION', maxResults: 10 },
        ],
      });

      return {
        textAnnotations: result.textAnnotations?.map(text => text.description || '').filter(Boolean),
        labelAnnotations: result.labelAnnotations?.map(label => ({
          description: label.description || '',
          score: label.score || 0,
        })),
        faceAnnotations: result.faceAnnotations?.map(face => ({
          boundingPoly: face.boundingPoly,
          confidence: face.detectionConfidence || 0,
          joyLikelihood: face.joyLikelihood || 'UNKNOWN',
          sorrowLikelihood: face.sorrowLikelihood || 'UNKNOWN',
          angerLikelihood: face.angerLikelihood || 'UNKNOWN',
          surpriseLikelihood: face.surpriseLikelihood || 'UNKNOWN',
        })),
        safeSearchAnnotation: result.safeSearchAnnotation ? {
          adult: result.safeSearchAnnotation.adult || 'UNKNOWN',
          spoof: result.safeSearchAnnotation.spoof || 'UNKNOWN',
          medical: result.safeSearchAnnotation.medical || 'UNKNOWN',
          violence: result.safeSearchAnnotation.violence || 'UNKNOWN',
          racy: result.safeSearchAnnotation.racy || 'UNKNOWN',
        } : undefined,
        logoAnnotations: result.logoAnnotations?.map(logo => ({
          description: logo.description || '',
          score: logo.score || 0,
        })),
      };
    } catch (error) {
      console.error('Base64画像解析エラー:', error);
      throw error;
    }
  }
}
```

### 2\. 基本的な使用例の実装

```typescript:src/example-basic.ts
import * as dotenv from 'dotenv';
import { VisionAIClient } from './vision-ai-client';

dotenv.config();

async function basicExample() {
  const visionClient = new VisionAIClient();
  
  try {
    // テキスト検出の例
    console.log('=== テキスト検出 ===');
    const textResults = await visionClient.detectText('./images/document.jpg');
    console.log('検出されたテキスト:');
    textResults.forEach((text, index) => {
      console.log(`${index + 1}: ${text}`);
    });

    // ラベル検出の例
    console.log('\n=== ラベル検出 ===');
    const labelResults = await visionClient.detectLabels('./images/scene.jpg');
    console.log('検出されたラベル:');
    labelResults.forEach(label => {
      console.log(`${label.description}: ${(label.score * 100).toFixed(1)}%`);
    });

    // 顔検出の例
    console.log('\n=== 顔検出 ===');
    const faceResults = await visionClient.detectFaces('./images/people.jpg');
    console.log(`検出された顔の数: ${faceResults.length}`);
    faceResults.forEach((face, index) => {
      console.log(`顔 ${index + 1}:`);
      console.log(`  信頼度: ${(face.confidence * 100).toFixed(1)}%`);
      console.log(`  喜び: ${face.joyLikelihood}`);
      console.log(`  悲しみ: ${face.sorrowLikelihood}`);
      console.log(`  怒り: ${face.angerLikelihood}`);
      console.log(`  驚き: ${face.surpriseLikelihood}`);
    });

    // 包括的解析の例
    console.log('\n=== 包括的画像解析 ===');
    const analysisResult = await visionClient.analyzeImage('./images/complex.jpg');
    
    if (analysisResult.textAnnotations?.length) {
      console.log(`テキスト: ${analysisResult.textAnnotations.length}個検出`);
    }
    
    if (analysisResult.labelAnnotations?.length) {
      console.log(`ラベル: ${analysisResult.labelAnnotations.length}個検出`);
      analysisResult.labelAnnotations.slice(0, 3).forEach(label => {
        console.log(`  - ${label.description} (${(label.score * 100).toFixed(1)}%)`);
      });
    }
    
    if (analysisResult.faceAnnotations?.length) {
      console.log(`顔: ${analysisResult.faceAnnotations.length}個検出`);
    }
    
    if (analysisResult.safeSearchAnnotation) {
      console.log('セーフサーチ:');
      console.log(`  成人向けコンテンツ: ${analysisResult.safeSearchAnnotation.adult}`);
      console.log(`  暴力的コンテンツ: ${analysisResult.safeSearchAnnotation.violence}`);
    }

  } catch (error) {
    console.error('エラーが発生しました:', error);
  }
}

// 実行
if (require.main === module) {
  basicExample();
}
```

## 実践編：Document AI風のOCRシステム構築

### 1\. 高度なOCR処理クラス

```typescript:src/advanced-ocr.ts
import { VisionAIClient } from './vision-ai-client';
import * as fs from 'fs/promises';

export interface OCRResult {
  fullText: string;
  paragraphs: string[];
  confidence: number;
  language: string;
  wordCount: number;
}

export interface DocumentStructure {
  title?: string;
  headers: string[];
  paragraphs: string[];
  tables?: string[][];
  metadata: {
    pageCount: number;
    confidence: number;
    detectedLanguage: string;
  };
}

export class AdvancedOCR extends VisionAIClient {
  
  /**
   * 構造化されたドキュメント解析
   */
  async analyzeDocument(imagePath: string): Promise<DocumentStructure> {
    try {
      const [result] = await this.client.documentTextDetection(imagePath);
      const fullTextAnnotation = result.fullTextAnnotation;
      
      if (!fullTextAnnotation) {
        throw new Error('テキストが検出されませんでした');
      }

      const text = fullTextAnnotation.text || '';
      const pages = fullTextAnnotation.pages || [];
      
      // テキストを段落に分割
      const paragraphs = this.extractParagraphs(text);
      
      // ヘッダーを抽出（フォントサイズや位置から推測）
      const headers = this.extractHeaders(pages);
      
      // タイトルを推測（最初の大きなテキストまたは最初の行）
      const title = this.extractTitle(paragraphs, headers);
      
      // 信頼度を計算
      const confidence = this.calculateConfidence(pages);
      
      return {
        title,
        headers,
        paragraphs: paragraphs.filter(p => !headers.includes(p) && p !== title),
        metadata: {
          pageCount: pages.length,
          confidence,
          detectedLanguage: this.detectLanguage(text),
        },
      };
    } catch (error) {
      console.error('ドキュメント解析エラー:', error);
      throw error;
    }
  }

  /**
   * バッチ処理：複数画像の一括OCR
   */
  async batchOCR(imagePaths: string[]): Promise<Array<{ path: string; result: OCRResult }>> {
    const results = [];
    
    for (const imagePath of imagePaths) {
      try {
        console.log(`処理中: ${imagePath}`);
        const textResult = await this.detectText(imagePath);
        
        const fullText = textResult.join('\n');
        const paragraphs = this.extractParagraphs(fullText);
        
        results.push({
          path: imagePath,
          result: {
            fullText,
            paragraphs,
            confidence: 0.95, // 実際の実装では詳細な信頼度計算を行う
            language: this.detectLanguage(fullText),
            wordCount: fullText.split(/\s+/).length,
          },
        });
      } catch (error) {
        console.error(`${imagePath}の処理に失敗:`, error);
        results.push({
          path: imagePath,
          result: {
            fullText: '',
            paragraphs: [],
            confidence: 0,
            language: 'unknown',
            wordCount: 0,
          },
        });
      }
    }
    
    return results;
  }

  /**
   * PDF風のドキュメント出力
   */
  async extractToMarkdown(imagePath: string): Promise<string> {
    const document = await this.analyzeDocument(imagePath);
    
    let markdown = '';
    
    if (document.title) {
      markdown += `# ${document.title}\n\n`;
    }
    
    document.headers.forEach(header => {
      markdown += `## ${header}\n\n`;
    });
    
    document.paragraphs.forEach(paragraph => {
      markdown += `${paragraph}\n\n`;
    });
    
    if (document.tables) {
      document.tables.forEach(table => {
        markdown += '| ' + table.join(' | ') + ' |\n';
        markdown += '| ' + table.map(() => '---').join(' | ') + ' |\n\n';
      });
    }
    
    markdown += `---\n`;
    markdown += `**メタデータ**\n`;
    markdown += `- ページ数: ${document.metadata.pageCount}\n`;
    markdown += `- 信頼度: ${(document.metadata.confidence * 100).toFixed(1)}%\n`;
    markdown += `- 検出言語: ${document.metadata.detectedLanguage}\n`;
    
    return markdown;
  }

  private extractParagraphs(text: string): string[] {
    return text
      .split(/\n\s*\n/)
      .map(p => p.trim())
      .filter(p => p.length > 0);
  }

  private extractHeaders(pages: any[]): string[] {
    // 簡単な実装：各段落の最初の短い行をヘッダーとして扱う
    const headers: string[] = [];
    
    pages.forEach(page => {
      page.blocks?.forEach((block: any) => {
        block.paragraphs?.forEach((paragraph: any) => {
          const text = paragraph.words
            ?.map((word: any) => word.symbols?.map((s: any) => s.text).join(''))
            .join(' ');
          
          if (text && text.length < 50 && text.length > 5) {
            headers.push(text);
          }
        });
      });
    });
    
    return [...new Set(headers)]; // 重複除去
  }

  private extractTitle(paragraphs: string[], headers: string[]): string | undefined {
    if (headers.length > 0) {
      return headers[0];
    }
    if (paragraphs.length > 0) {
      return paragraphs[0].length < 100 ? paragraphs[0] : undefined;
    }
    return undefined;
  }

  private calculateConfidence(pages: any[]): number {
    let totalConfidence = 0;
    let wordCount = 0;
    
    pages.forEach(page => {
      page.blocks?.forEach((block: any) => {
        block.paragraphs?.forEach((paragraph: any) => {
          paragraph.words?.forEach((word: any) => {
            if (word.confidence !== undefined) {
              totalConfidence += word.confidence;
              wordCount++;
            }
          });
        });
      });
    });
    
    return wordCount > 0 ? totalConfidence / wordCount : 0;
  }

  private detectLanguage(text: string): string {
    // 簡単な言語検出（ひらがな・カタカナがあれば日本語）
    if (/[\u3040-\u309F\u30A0-\u30FF]/.test(text)) {
      return 'ja';
    }
    if (/[a-zA-Z]/.test(text)) {
      return 'en';
    }
    return 'unknown';
  }
}
```

### 2\. リアルタイム画像解析API

```typescript:src/vision-api-server.ts
import express from 'express';
import multer from 'multer';
import { VisionAIClient } from './vision-ai-client';
import { AdvancedOCR } from './advanced-ocr';
import * as fs from 'fs';
import * as path from 'path';

const app = express();
const port = process.env.PORT || 3000;

// ファイルアップロード設定
const upload = multer({
  dest: 'uploads/',
  limits: {
    fileSize: 10 * 1024 * 1024, // 10MB制限
  },
  fileFilter: (req, file, cb) => {
    const allowedMimes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
    if (allowedMimes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('対応していないファイル形式です'));
    }
  },
});

const visionClient = new VisionAIClient();
const ocrClient = new AdvancedOCR();

app.use(express.json({ limit: '10mb' }));
app.use(express.static('public'));

/**
 * 基本的な画像解析エンドポイント
 */
app.post('/api/analyze', upload.single('image'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: '画像ファイルが必要です' });
    }

    const result = await visionClient.analyzeImage(req.file.path);
    
    // アップロードファイルを削除
    fs.unlinkSync(req.file.path);
    
    res.json({
      success: true,
      data: result,
    });
  } catch (error) {
    console.error('画像解析エラー:', error);
    res.status(500).json({ error: '画像解析に失敗しました' });
  }
});

/**
 * OCR専用エンドポイント
 */
app.post('/api/ocr', upload.single('image'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: '画像ファイルが必要です' });
    }

    const result = await ocrClient.analyzeDocument(req.file.path);
    
    // アップロードファイルを削除
    fs.unlinkSync(req.file.path);
    
    res.json({
      success: true,
      data: result,
    });
  } catch (error) {
    console.error('OCRエラー:', error);
    res.status(500).json({ error: 'OCR処理に失敗しました' });
  }
});

/**
 * Base64画像解析エンドポイント
 */
app.post('/api/analyze-base64', async (req, res) => {
  try {
    const { image, type = 'basic' } = req.body;
    
    if (!image) {
      return res.status(400).json({ error: 'Base64画像データが必要です' });
    }

    const result = await visionClient.analyzeImageFromBase64(image);
    
    res.json({
      success: true,
      data: result,
    });
  } catch (error) {
    console.error('Base64画像解析エラー:', error);
    res.status(500).json({ error: '画像解析に失敗しました' });
  }
});

/**
 * バッチOCR処理エンドポイント
 */
app.post('/api/batch-ocr', upload.array('images', 10), async (req, res) => {
  try {
    if (!req.files || req.files.length === 0) {
      return res.status(400).json({ error: '画像ファイルが必要です' });
    }

    const imagePaths = (req.files as Express.Multer.File[]).map(file => file.path);
    const results = await ocrClient.batchOCR(imagePaths);
    
    // アップロードファイルを削除
    imagePaths.forEach(path => {
      fs.unlinkSync(path);
    });
    
    res.json({
      success: true,
      data: results,
    });
  } catch (error) {
    console.error('バッチOCRエラー:', error);
    res.status(500).json({ error: 'バッチOCR処理に失敗しました' });
  }
});

/**
 * ヘルスチェックエンドポイント
 */
app.get('/api/health', (req, res) => {
  res.json({
    status: 'OK',
    timestamp: new Date().toISOString(),
    service: 'GCP Vision AI Server',
  });
});

app.listen(port, () => {
  console.log(`🚀 Vision AI Server running on port ${port}`);
  console.log(`📱 API Endpoints:`);
  console.log(`   POST /api/analyze - 基本的な画像解析`);
  console.log(`   POST /api/ocr - OCR処理`);
  console.log(`   POST /api/analyze-base64 - Base64画像解析`);
  console.log(`   POST /api/batch-ocr - バッチOCR処理`);
  console.log(`   GET /api/health - ヘルスチェック`);
});

export default app;
```

## 実用的な応用例：名刺読み取りシステム

### 1\. 名刺データ抽出クラス

```typescript:src/business-card-reader.ts
import { AdvancedOCR } from './advanced-ocr';

export interface BusinessCardData {
  name?: string;
  company?: string;
  title?: string;
  email?: string;
  phone?: string;
  address?: string;
  website?: string;
  rawText: string;
  confidence: number;
}

export class BusinessCardReader extends AdvancedOCR {
  
  /**
   * 名刺から構造化データを抽出
   */
  async extractBusinessCardData(imagePath: string): Promise<BusinessCardData> {
    try {
      const textResult = await this.detectText(imagePath);
      const rawText = textResult.join('\n');
      
      const businessCardData: BusinessCardData = {
        rawText,
        confidence: 0.8, // 実際の実装では詳細な信頼度計算
      };
      
      // 各フィールドを抽出
      businessCardData.name = this.extractName(textResult);
      businessCardData.company = this.extractCompany(textResult);
      businessCardData.title = this.extractTitle(textResult);
      businessCardData.email = this.extractEmail(rawText);
      businessCardData.phone = this.extractPhone(rawText);
      businessCardData.address = this.extractAddress(textResult);
      businessCardData.website = this.extractWebsite(rawText);
      
      return businessCardData;
    } catch (error) {
      console.error('名刺読み取りエラー:', error);
      throw error;
    }
  }

  private extractName(textLines: string[]): string | undefined {
    // 日本の名刺では通常名前が大きく表示される
    // ここでは簡単な実装として、短い行で漢字とひらがなを含む行を探す
    for (const line of textLines.slice(0, 5)) { // 上位5行をチェック
      if (line.length >= 2 && line.length <= 20 && /[\u3040-\u309F\u4E00-\u9FAF]/.test(line)) {
        // 会社名や肩書きのキーワードが含まれていない場合
        if (!this.isCompanyOrTitle(line)) {
          return line.trim();
        }
      }
    }
    return undefined;
  }

  private extractCompany(textLines: string[]): string | undefined {
    const companyKeywords = ['株式会社', '有限会社', '合同会社', '一般社団法人', '公益財団法人', 'Corp', 'Inc', 'LLC', 'Co.', 'Ltd'];
    
    for (const line of textLines) {
      if (companyKeywords.some(keyword => line.includes(keyword))) {
        return line.trim();
      }
    }
    
    // 会社っぽい単語を含む行を探す
    for (const line of textLines) {
      if (line.length > 3 && /[会社組合団法人]/.test(line)) {
        return line.trim();
      }
    }
    
    return undefined;
  }

  private extractTitle(textLines: string[]): string | undefined {
    const titleKeywords = ['代表取締役', '取締役', '部長', '課長', '主任', '係長', 'CEO', 'CTO', 'Manager', 'Director'];
    
    for (const line of textLines) {
      if (titleKeywords.some(keyword => line.includes(keyword))) {
        return line.trim();
      }
    }
    
    return undefined;
  }

  private extractEmail(text: string): string | undefined {
    const emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g;
    const match = text.match(emailRegex);
    return match ? match[0] : undefined;
  }

  private extractPhone(text: string): string | undefined {
    // 日本の電話番号パターン
    const phonePatterns = [
      /0\d{1,4}-\d{1,4}-\d{4}/g, // 03-1234-5678
      /0\d{9,10}/g, // 09012345678
      /\+81-\d{1,4}-\d{1,4}-\d{4}/g, // +81-3-1234-5678
    ];
    
    for (const pattern of phonePatterns) {
      const match = text.match(pattern);
      if (match) {
        return match[0];
      }
    }
    
    return undefined;
  }

  private extractAddress(textLines: string[]): string | undefined {
    const addressKeywords = ['東京都', '大阪府', '愛知県', '神奈川県', '区', '市', '町', '村'];
    
    for (const line of textLines) {
      if (addressKeywords.some(keyword => line.includes(keyword)) && line.length > 10) {
        return line.trim();
      }
    }
    
    return undefined;
  }

  private extractWebsite(text: string): string | undefined {
    const urlRegex = /https?:\/\/[^\s]+|www\.[^\s]+\.[a-zA-Z]{2,}/g;
    const match = text.match(urlRegex);
    return match ? match[0] : undefined;
  }

  private isCompanyOrTitle(text: string): boolean {
    const companyTitleKeywords = ['株式会社', '有限会社', '代表取締役', '部長', '課長', '主任'];
    return companyTitleKeywords.some(keyword => text.includes(keyword));
  }

  /**
   * 複数の名刺を一括処理
   */
  async processBatchBusinessCards(imagePaths: string[]): Promise<BusinessCardData[]> {
    const results = [];
    
    for (const imagePath of imagePaths) {
      try {
        const data = await this.extractBusinessCardData(imagePath);
        results.push(data);
      } catch (error) {
        console.error(`名刺処理エラー (${imagePath}):`, error);
        results.push({
          rawText: '',
          confidence: 0,
        });
      }
    }
    
    return results;
  }

  /**
   * 名刺データをvCardフォーマットで出力
   */
  generateVCard(data: BusinessCardData): string {
    let vcard = 'BEGIN:VCARD\n';
    vcard += 'VERSION:3.0\n';
    
    if (data.name) {
      vcard += `FN:${data.name}\n`;
    }
    
    if (data.company) {
      vcard += `ORG:${data.company}\n`;
    }
    
    if (data.title) {
      vcard += `TITLE:${data.title}\n`;
    }
    
    if (data.email) {
      vcard += `EMAIL:${data.email}\n`;
    }
    
    if (data.phone) {
      vcard += `TEL:${data.phone}\n`;
    }
    
    if (data.address) {
      vcard += `ADR:${data.address}\n`;
    }
    
    if (data.website) {
      vcard += `URL:${data.website}\n`;
    }
    
    vcard += 'END:VCARD';
    
    return vcard;
  }
}
```

### 2\. 名刺読み取りAPIの実装

```typescript:src/business-card-api.ts
import express from 'express';
import multer from 'multer';
import { BusinessCardReader } from './business-card-reader';
import * as fs from 'fs';

const router = express.Router();
const upload = multer({ dest: 'uploads/' });
const businessCardReader = new BusinessCardReader();

/**
 * 名刺読み取りエンドポイント
 */
router.post('/extract', upload.single('businessCard'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: '名刺画像が必要です' });
    }

    const result = await businessCardReader.extractBusinessCardData(req.file.path);
    
    // アップロードファイルを削除
    fs.unlinkSync(req.file.path);
    
    res.json({
      success: true,
      data: result,
    });
  } catch (error) {
    console.error('名刺読み取りエラー:', error);
    res.status(500).json({ error: '名刺読み取りに失敗しました' });
  }
});

/**
 * vCard生成エンドポイント
 */
router.post('/generate-vcard', upload.single('businessCard'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: '名刺画像が必要です' });
    }

    const businessCardData = await businessCardReader.extractBusinessCardData(req.file.path);
    const vcard = businessCardReader.generateVCard(businessCardData);
    
    // アップロードファイルを削除
    fs.unlinkSync(req.file.path);
    
    res.setHeader('Content-Type', 'text/vcard');
    res.setHeader('Content-Disposition', 'attachment; filename="contact.vcf"');
    res.send(vcard);
  } catch (error) {
    console.error('vCard生成エラー:', error);
    res.status(500).json({ error: 'vCard生成に失敗しました' });
  }
});

/**
 * 一括名刺処理エンドポイント
 */
router.post('/batch-extract', upload.array('businessCards', 20), async (req, res) => {
  try {
    if (!req.files || req.files.length === 0) {
      return res.status(400).json({ error: '名刺画像が必要です' });
    }

    const imagePaths = (req.files as Express.Multer.File[]).map(file => file.path);
    const results = await businessCardReader.processBatchBusinessCards(imagePaths);
    
    // アップロードファイルを削除
    imagePaths.forEach(path => {
      fs.unlinkSync(path);
    });
    
    res.json({
      success: true,
      data: results,
      count: results.length,
    });
  } catch (error) {
    console.error('一括名刺処理エラー:', error);
    res.status(500).json({ error: '一括名刺処理に失敗しました' });
  }
});

export default router;
```

## まとめ

本記事では、GCP Vision AIをTypeScriptで活用する実践的な方法をご紹介しました。

**📌 実装したもの**

>* 基本的なVision AIクライアントクラス
>* 高度なOCR処理システム
>* リアルタイム画像解析API
>* 名刺読み取りシステム

**✅ 次のステップ**

>* AutoMLで独自のカスタムモデル作成
>* Cloud Storageと連携したバッチ処理システム
>* フロントエンドとの統合（React/Vue.js）
>* パフォーマンス最適化とキャッシュ戦略

Vision AIを使えば、複雑な画像認識処理がたった数行のTypeScriptコードで実現できます。ぜひ実際のプロジェクトで試してみてください！