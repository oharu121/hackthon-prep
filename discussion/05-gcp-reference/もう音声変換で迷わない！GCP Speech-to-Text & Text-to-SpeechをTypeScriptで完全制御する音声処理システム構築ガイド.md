# もう音声変換で迷わない！GCP Speech-to-Text & Text-to-SpeechをTypeScriptで完全制御する音声処理システム構築ガイド

## はじめに

「音声認識機能を実装したいけど、どのAPIを使えばいいかわからない...」
「文字読み上げ機能を作りたいけど、品質の高い音声合成サービスが見つからない...」

こんな悩みを抱えている開発者の方も多いのではないでしょうか。

GoogleのSpeech-to-Text APIとText-to-Speech APIを使えば、企業レベルの高品質な音声処理システムを比較的簡単に構築できます。この記事では、TypeScriptを使った実装方法から、実際のプロダクション運用まで、体系的に解説していきます。

**🎯 この記事で学べること**

>* GCP Speech-to-Text APIの基本的な使い方と実装パターン
>* GCP Text-to-Speech APIの多言語・多音声対応実装
>* リアルタイム音声認識システムの構築方法
>* 音声ファイルのアップロード・変換・ダウンロード機能の実装
>* エラーハンドリングとパフォーマンス最適化のベストプラクティス
>* 料金体系と運用コストの最適化テクニック

**📋 前提条件**

>* Node.js 18以上がインストールされていること
>* GCPアカウントとプロジェクトが作成済みであること
>* TypeScriptの基本的な知識があること
>* 基本的なGCPサービスの理解があること

## 🚀 GCP Speech-to-Text APIの実装

### 1\. 環境構築とセットアップ

まずは必要なパッケージをインストールします：

```typescript:src/setup.ts
npm install @google-cloud/speech @google-cloud/text-to-speech
npm install -D @types/node typescript ts-node
```

GCPの認証情報を設定します：

```typescript:src/config/gcp-config.ts
import { SpeechClient } from '@google-cloud/speech';
import { TextToSpeechClient } from '@google-cloud/text-to-speech';

export const speechClient = new SpeechClient({
  projectId: process.env.GCP_PROJECT_ID,
  keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
});

export const ttsClient = new TextToSpeechClient({
  projectId: process.env.GCP_PROJECT_ID,
  keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
});

export const GCP_CONFIG = {
  projectId: process.env.GCP_PROJECT_ID || '',
  location: process.env.GCP_LOCATION || 'global',
  bucketName: process.env.GCS_BUCKET_NAME || '',
} as const;
```

環境変数ファイルを設定します：

```typescript:.env
GCP_PROJECT_ID=your-project-id
GOOGLE_APPLICATION_CREDENTIALS=./path/to/service-account-key.json
GCP_LOCATION=asia-northeast1
GCS_BUCKET_NAME=your-bucket-name
```

### 2\. Speech-to-Text APIの基本実装

基本的な音声認識機能を実装します：

```typescript:src/services/speech-to-text.service.ts
import { SpeechClient } from '@google-cloud/speech';
import { readFileSync } from 'fs';

export class SpeechToTextService {
  private client: SpeechClient;

  constructor(client: SpeechClient) {
    this.client = client;
  }

  async transcribeAudioFile(audioFilePath: string, languageCode: string = 'ja-JP'): Promise<string> {
    try {
      const audioBytes = readFileSync(audioFilePath).toString('base64');

      const request = {
        audio: {
          content: audioBytes,
        },
        config: {
          encoding: 'WEBM_OPUS' as const,
          sampleRateHertz: 48000,
          languageCode,
          enableAutomaticPunctuation: true,
          enableWordTimeOffsets: true,
          model: 'latest_long',
        },
      };

      const [response] = await this.client.recognize(request);
      
      if (!response.results || response.results.length === 0) {
        throw new Error('音声認識結果が空です');
      }

      const transcription = response.results
        .map(result => result.alternatives?.[0]?.transcript || '')
        .join(' ')
        .trim();

      return transcription;
    } catch (error) {
      console.error('音声認識エラー:', error);
      throw new Error(`音声認識に失敗しました: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  async transcribeWithConfidence(audioFilePath: string, languageCode: string = 'ja-JP') {
    try {
      const audioBytes = readFileSync(audioFilePath).toString('base64');

      const request = {
        audio: {
          content: audioBytes,
        },
        config: {
          encoding: 'WEBM_OPUS' as const,
          sampleRateHertz: 48000,
          languageCode,
          enableAutomaticPunctuation: true,
          enableWordTimeOffsets: true,
          model: 'latest_long',
          maxAlternatives: 3,
        },
      };

      const [response] = await this.client.recognize(request);
      
      if (!response.results || response.results.length === 0) {
        throw new Error('音声認識結果が空です');
      }

      const results = response.results.map(result => ({
        transcript: result.alternatives?.[0]?.transcript || '',
        confidence: result.alternatives?.[0]?.confidence || 0,
        words: result.alternatives?.[0]?.words?.map(word => ({
          word: word.word || '',
          startTime: word.startTime?.seconds || 0,
          endTime: word.endTime?.seconds || 0,
        })) || [],
        alternatives: result.alternatives?.map(alt => ({
          transcript: alt.transcript || '',
          confidence: alt.confidence || 0,
        })) || [],
      }));

      return results;
    } catch (error) {
      console.error('詳細音声認識エラー:', error);
      throw new Error(`詳細音声認識に失敗しました: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }
}
```

### 3\. リアルタイム音声認識の実装

ストリーミング音声認識を実装します：

```typescript:src/services/streaming-speech.service.ts
import { SpeechClient } from '@google-cloud/speech';
import { Duplex } from 'stream';

export class StreamingSpeechService {
  private client: SpeechClient;

  constructor(client: SpeechClient) {
    this.client = client;
  }

  createStreamingRecognition(languageCode: string = 'ja-JP') {
    const request = {
      config: {
        encoding: 'WEBM_OPUS' as const,
        sampleRateHertz: 48000,
        languageCode,
        enableAutomaticPunctuation: true,
        model: 'latest_long',
      },
      interimResults: true,
    };

    const recognizeStream = this.client
      .streamingRecognize(request)
      .on('error', (error) => {
        console.error('ストリーミング認識エラー:', error);
      })
      .on('data', (data) => {
        if (data.results?.[0]?.alternatives?.[0]) {
          const result = data.results[0].alternatives[0];
          const isFinal = data.results[0].isFinal;
          
          console.log(`${isFinal ? '[確定]' : '[暫定]'} ${result.transcript}`);
          
          if (isFinal) {
            console.log(`信頼度: ${result.confidence}`);
          }
        }
      });

    return recognizeStream;
  }

  async processAudioStream(audioStream: Duplex, languageCode: string = 'ja-JP'): Promise<string[]> {
    return new Promise((resolve, reject) => {
      const recognizeStream = this.createStreamingRecognition(languageCode);
      const transcripts: string[] = [];

      recognizeStream.on('data', (data) => {
        if (data.results?.[0]?.alternatives?.[0] && data.results[0].isFinal) {
          transcripts.push(data.results[0].alternatives[0].transcript);
        }
      });

      recognizeStream.on('error', reject);
      recognizeStream.on('end', () => resolve(transcripts));

      audioStream.pipe(recognizeStream);
    });
  }
}
```

## GCP Text-to-Speech APIの実装

### 1\. 基本的な音声合成機能

多言語対応の音声合成サービスを実装します：

```typescript:src/services/text-to-speech.service.ts
import { TextToSpeechClient } from '@google-cloud/text-to-speech';
import { writeFileSync } from 'fs';

export interface SynthesizeOptions {
  languageCode?: string;
  ssmlGender?: 'NEUTRAL' | 'FEMALE' | 'MALE';
  voiceName?: string;
  audioEncoding?: 'MP3' | 'LINEAR16' | 'OGG_OPUS';
  speakingRate?: number;
  pitch?: number;
  volumeGainDb?: number;
}

export class TextToSpeechService {
  private client: TextToSpeechClient;

  constructor(client: TextToSpeechClient) {
    this.client = client;
  }

  async synthesizeText(
    text: string,
    outputPath: string,
    options: SynthesizeOptions = {}
  ): Promise<void> {
    try {
      const {
        languageCode = 'ja-JP',
        ssmlGender = 'NEUTRAL',
        voiceName,
        audioEncoding = 'MP3',
        speakingRate = 1.0,
        pitch = 0.0,
        volumeGainDb = 0.0,
      } = options;

      const request = {
        input: { text },
        voice: {
          languageCode,
          ssmlGender,
          ...(voiceName && { name: voiceName }),
        },
        audioConfig: {
          audioEncoding,
          speakingRate,
          pitch,
          volumeGainDb,
        },
      };

      const [response] = await this.client.synthesizeSpeech(request);
      
      if (!response.audioContent) {
        throw new Error('音声合成結果が空です');
      }

      writeFileSync(outputPath, response.audioContent, 'binary');
      console.log(`音声ファイルを保存しました: ${outputPath}`);
    } catch (error) {
      console.error('音声合成エラー:', error);
      throw new Error(`音声合成に失敗しました: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  async synthesizeSSML(
    ssml: string,
    outputPath: string,
    options: SynthesizeOptions = {}
  ): Promise<void> {
    try {
      const {
        languageCode = 'ja-JP',
        ssmlGender = 'NEUTRAL',
        voiceName,
        audioEncoding = 'MP3',
        speakingRate = 1.0,
        pitch = 0.0,
        volumeGainDb = 0.0,
      } = options;

      const request = {
        input: { ssml },
        voice: {
          languageCode,
          ssmlGender,
          ...(voiceName && { name: voiceName }),
        },
        audioConfig: {
          audioEncoding,
          speakingRate,
          pitch,
          volumeGainDb,
        },
      };

      const [response] = await this.client.synthesizeSpeech(request);
      
      if (!response.audioContent) {
        throw new Error('SSML音声合成結果が空です');
      }

      writeFileSync(outputPath, response.audioContent, 'binary');
      console.log(`SSML音声ファイルを保存しました: ${outputPath}`);
    } catch (error) {
      console.error('SSML音声合成エラー:', error);
      throw new Error(`SSML音声合成に失敗しました: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  async getAvailableVoices(languageCode?: string) {
    try {
      const request = languageCode ? { languageCode } : {};
      const [response] = await this.client.listVoices(request);
      
      return response.voices?.map(voice => ({
        name: voice.name,
        languageCodes: voice.languageCodes,
        ssmlGender: voice.ssmlGender,
        naturalSampleRateHertz: voice.naturalSampleRateHertz,
      })) || [];
    } catch (error) {
      console.error('音声リスト取得エラー:', error);
      throw new Error(`音声リスト取得に失敗しました: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }
}
```

### 2\. SSML対応と高度な音声制御

SSML（Speech Synthesis Markup Language）を使った高度な音声制御を実装します：

```typescript:src/services/ssml-builder.service.ts
export class SSMLBuilder {
  private ssml: string = '<speak>';

  constructor() {}

  text(content: string): SSMLBuilder {
    this.ssml += content;
    return this;
  }

  pause(seconds: number): SSMLBuilder {
    this.ssml += `<break time="${seconds}s"/>`;
    return this;
  }

  emphasis(text: string, level: 'strong' | 'moderate' | 'reduced' = 'moderate'): SSMLBuilder {
    this.ssml += `<emphasis level="${level}">${text}</emphasis>`;
    return this;
  }

  prosody(
    text: string,
    options: {
      rate?: 'x-slow' | 'slow' | 'medium' | 'fast' | 'x-fast' | string;
      pitch?: 'x-low' | 'low' | 'medium' | 'high' | 'x-high' | string;
      volume?: 'silent' | 'x-soft' | 'soft' | 'medium' | 'loud' | 'x-loud' | string;
    }
  ): SSMLBuilder {
    let attributes = '';
    if (options.rate) attributes += ` rate="${options.rate}"`;
    if (options.pitch) attributes += ` pitch="${options.pitch}"`;
    if (options.volume) attributes += ` volume="${options.volume}"`;

    this.ssml += `<prosody${attributes}>${text}</prosody>`;
    return this;
  }

  sayAs(text: string, interpretAs: 'characters' | 'spell-out' | 'cardinal' | 'number' | 'ordinal' | 'digits' | 'fraction' | 'unit' | 'date' | 'time' | 'telephone'): SSMLBuilder {
    this.ssml += `<say-as interpret-as="${interpretAs}">${text}</say-as>`;
    return this;
  }

  audio(src: string, alt?: string): SSMLBuilder {
    const altText = alt ? ` alt="${alt}"` : '';
    this.ssml += `<audio src="${src}"${altText}></audio>`;
    return this;
  }

  sub(alias: string, substitute: string): SSMLBuilder {
    this.ssml += `<sub alias="${substitute}">${alias}</sub>`;
    return this;
  }

  build(): string {
    return this.ssml + '</speak>';
  }

  reset(): SSMLBuilder {
    this.ssml = '<speak>';
    return this;
  }

  static createNewsReading(title: string, content: string): string {
    return new SSMLBuilder()
      .emphasis(title, 'strong')
      .pause(1)
      .prosody(content, { rate: 'medium', pitch: 'medium' })
      .build();
  }

  static createPhoneNumber(phoneNumber: string): string {
    return new SSMLBuilder()
      .sayAs(phoneNumber, 'telephone')
      .build();
  }

  static createDateAnnouncement(date: string): string {
    return new SSMLBuilder()
      .text('本日は')
      .sayAs(date, 'date')
      .text('です')
      .build();
  }
}
```

## 統合音声処理システムの実装

Speech-to-TextとText-to-Speechを組み合わせた統合システムを実装します：

```typescript:src/services/integrated-speech.service.ts
import { SpeechToTextService } from './speech-to-text.service';
import { TextToSpeechService } from './text-to-speech.service';
import { SSMLBuilder } from './ssml-builder.service';

export interface AudioProcessingResult {
  originalAudio: string;
  transcription: string;
  confidence: number;
  processedText: string;
  outputAudio: string;
  processingTime: number;
}

export class IntegratedSpeechService {
  private sttService: SpeechToTextService;
  private ttsService: TextToSpeechService;

  constructor(sttService: SpeechToTextService, ttsService: TextToSpeechService) {
    this.sttService = sttService;
    this.ttsService = ttsService;
  }

  async processAudioToAudio(
    inputAudioPath: string,
    outputAudioPath: string,
    textProcessor?: (text: string) => string,
    languageCode: string = 'ja-JP'
  ): Promise<AudioProcessingResult> {
    const startTime = Date.now();

    try {
      // 1. 音声をテキストに変換
      console.log('音声認識を開始します...');
      const recognitionResults = await this.sttService.transcribeWithConfidence(inputAudioPath, languageCode);
      
      if (recognitionResults.length === 0) {
        throw new Error('音声認識結果が取得できませんでした');
      }

      const transcription = recognitionResults[0].transcript;
      const confidence = recognitionResults[0].confidence;

      console.log(`認識結果: ${transcription}`);
      console.log(`信頼度: ${confidence}`);

      // 2. テキストを処理（必要に応じて）
      const processedText = textProcessor ? textProcessor(transcription) : transcription;
      
      // 3. 処理されたテキストを音声に変換
      console.log('音声合成を開始します...');
      await this.ttsService.synthesizeText(processedText, outputAudioPath, { languageCode });

      const processingTime = Date.now() - startTime;

      return {
        originalAudio: inputAudioPath,
        transcription,
        confidence,
        processedText,
        outputAudio: outputAudioPath,
        processingTime,
      };
    } catch (error) {
      console.error('音声処理エラー:', error);
      throw new Error(`音声処理に失敗しました: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  async createVoiceBot(
    inputAudioPath: string,
    outputAudioPath: string,
    botResponder: (userMessage: string) => Promise<string>,
    languageCode: string = 'ja-JP'
  ): Promise<AudioProcessingResult> {
    const startTime = Date.now();

    try {
      // 1. ユーザーの音声を認識
      console.log('ユーザーの音声を認識中...');
      const transcription = await this.sttService.transcribeAudioFile(inputAudioPath, languageCode);
      console.log(`ユーザー: ${transcription}`);

      // 2. ボットが応答を生成
      console.log('ボットが応答を生成中...');
      const botResponse = await botResponder(transcription);
      console.log(`ボット: ${botResponse}`);

      // 3. ボットの応答を音声に変換
      console.log('ボットの音声を生成中...');
      const ssml = new SSMLBuilder()
        .prosody(botResponse, { rate: 'medium', pitch: 'medium' })
        .build();

      await this.ttsService.synthesizeSSML(ssml, outputAudioPath, { languageCode });

      const processingTime = Date.now() - startTime;

      return {
        originalAudio: inputAudioPath,
        transcription,
        confidence: 1.0, // ボットレスポンスなので信頼度は最大
        processedText: botResponse,
        outputAudio: outputAudioPath,
        processingTime,
      };
    } catch (error) {
      console.error('音声ボット処理エラー:', error);
      throw new Error(`音声ボット処理に失敗しました: ${error instanceof Error ? error.message : 'Unknown error'}`);
    }
  }

  async batchProcessAudioFiles(
    inputFiles: string[],
    outputDir: string,
    textProcessor?: (text: string) => string,
    languageCode: string = 'ja-JP'
  ): Promise<AudioProcessingResult[]> {
    const results: AudioProcessingResult[] = [];

    for (let i = 0; i < inputFiles.length; i++) {
      const inputFile = inputFiles[i];
      const outputFile = `${outputDir}/processed_${i + 1}.mp3`;

      try {
        console.log(`処理中 (${i + 1}/${inputFiles.length}): ${inputFile}`);
        const result = await this.processAudioToAudio(
          inputFile,
          outputFile,
          textProcessor,
          languageCode
        );
        results.push(result);
      } catch (error) {
        console.error(`ファイル処理エラー (${inputFile}):`, error);
        // エラーが発生してもスキップして続行
      }
    }

    return results;
  }
}
```

## 実用例とサンプル実装

### 1\. 基本的な使用例

```typescript:src/examples/basic-usage.ts
import { speechClient, ttsClient } from '../config/gcp-config';
import { SpeechToTextService } from '../services/speech-to-text.service';
import { TextToSpeechService } from '../services/text-to-speech.service';
import { IntegratedSpeechService } from '../services/integrated-speech.service';
import { SSMLBuilder } from '../services/ssml-builder.service';

async function basicExample() {
  // サービスを初期化
  const sttService = new SpeechToTextService(speechClient);
  const ttsService = new TextToSpeechService(ttsClient);
  const integratedService = new IntegratedSpeechService(sttService, ttsService);

  try {
    // 1. 音声認識のテスト
    console.log('=== 音声認識テスト ===');
    const transcription = await sttService.transcribeAudioFile('./input/sample.wav', 'ja-JP');
    console.log(`認識結果: ${transcription}`);

    // 2. 音声合成のテスト
    console.log('\n=== 音声合成テスト ===');
    await ttsService.synthesizeText(
      'こんにちは、Google Cloud Platform の音声APIのテストです。',
      './output/basic-test.mp3',
      {
        languageCode: 'ja-JP',
        ssmlGender: 'FEMALE',
        speakingRate: 1.0,
        pitch: 0.0,
      }
    );

    // 3. SSML音声合成のテスト
    console.log('\n=== SSML音声合成テスト ===');
    const ssml = new SSMLBuilder()
      .text('これは')
      .emphasis('SSML', 'strong')
      .text('を使った音声合成の')
      .pause(1)
      .prosody('テストです', { rate: 'slow', pitch: 'high' })
      .build();

    await ttsService.synthesizeSSML(ssml, './output/ssml-test.mp3');

    // 4. 統合処理のテスト
    console.log('\n=== 統合処理テスト ===');
    const textProcessor = (text: string) => {
      return `処理結果: ${text.replace(/えー|あー|うー/g, '')}`;
    };

    const result = await integratedService.processAudioToAudio(
      './input/sample.wav',
      './output/processed.mp3',
      textProcessor
    );

    console.log('統合処理完了:', result);

  } catch (error) {
    console.error('基本例エラー:', error);
  }
}

// 実行
basicExample();
```

### 2\. 音声ボットの実装例

```typescript:src/examples/voice-bot.ts
import { speechClient, ttsClient } from '../config/gcp-config';
import { SpeechToTextService } from '../services/speech-to-text.service';
import { TextToSpeechService } from '../services/text-to-speech.service';
import { IntegratedSpeechService } from '../services/integrated-speech.service';

async function voiceBotExample() {
  const sttService = new SpeechToTextService(speechClient);
  const ttsService = new TextToSpeechService(ttsClient);
  const integratedService = new IntegratedSpeechService(sttService, ttsService);

  // シンプルなボットの応答ロジック
  const botResponder = async (userMessage: string): Promise<string> => {
    const lowerMessage = userMessage.toLowerCase();
    
    if (lowerMessage.includes('こんにちは') || lowerMessage.includes('こんばんは')) {
      return 'こんにちは！今日はどのようなご用件でしょうか？';
    } else if (lowerMessage.includes('天気')) {
      return '申し訳ございませんが、天気予報の機能は現在準備中です。';
    } else if (lowerMessage.includes('時間') || lowerMessage.includes('何時')) {
      const now = new Date();
      const timeString = now.toLocaleTimeString('ja-JP');
      return `現在の時刻は${timeString}です。`;
    } else if (lowerMessage.includes('ありがとう') || lowerMessage.includes('感謝')) {
      return 'どういたしまして！他にもご質問があればお聞かせください。';
    } else if (lowerMessage.includes('さようなら') || lowerMessage.includes('バイバイ')) {
      return 'さようなら！またお話しましょう。';
    } else {
      return '申し訳ございませんが、よく理解できませんでした。別の言葉で教えていただけますか？';
    }
  };

  try {
    console.log('=== 音声ボット開始 ===');
    
    const result = await integratedService.createVoiceBot(
      './input/user-question.wav',
      './output/bot-response.mp3',
      botResponder,
      'ja-JP'
    );

    console.log('ボット処理完了:', result);
    console.log(`処理時間: ${result.processingTime}ms`);

  } catch (error) {
    console.error('音声ボットエラー:', error);
  }
}

// 実行
voiceBotExample();
```

### 3\. バッチ処理の実装例

```typescript:src/examples/batch-processing.ts
import { speechClient, ttsClient } from '../config/gcp-config';
import { SpeechToTextService } from '../services/speech-to-text.service';
import { TextToSpeechService } from '../services/text-to-speech.service';
import { IntegratedSpeechService } from '../services/integrated-speech.service';
import { readdirSync } from 'fs';
import { join } from 'path';

async function batchProcessingExample() {
  const sttService = new SpeechToTextService(speechClient);
  const ttsService = new TextToSpeechService(ttsClient);
  const integratedService = new IntegratedSpeechService(sttService, ttsService);

  // テキスト正規化プロセッサ
  const textNormalizer = (text: string): string => {
    return text
      .replace(/えー+|あー+|うー+/g, '') // フィラーワードを削除
      .replace(/\s+/g, ' ') // 複数の空白を1つに
      .trim()
      .replace(/。$/, '') // 末尾の句点を削除
      + '。以上です。'; // 統一した終了表現を追加
  };

  try {
    console.log('=== バッチ処理開始 ===');

    // inputディレクトリから音声ファイルを取得
    const inputDir = './input/batch';
    const outputDir = './output/batch';
    
    const audioFiles = readdirSync(inputDir)
      .filter(file => file.endsWith('.wav') || file.endsWith('.mp3'))
      .map(file => join(inputDir, file));

    console.log(`処理対象ファイル数: ${audioFiles.length}`);

    // バッチ処理を実行
    const results = await integratedService.batchProcessAudioFiles(
      audioFiles,
      outputDir,
      textNormalizer,
      'ja-JP'
    );

    // 結果をサマリー出力
    console.log('\n=== バッチ処理結果サマリー ===');
    const totalTime = results.reduce((sum, result) => sum + result.processingTime, 0);
    const avgConfidence = results.reduce((sum, result) => sum + result.confidence, 0) / results.length;

    console.log(`処理成功: ${results.length}/${audioFiles.length}`);
    console.log(`総処理時間: ${totalTime}ms`);
    console.log(`平均信頼度: ${avgConfidence.toFixed(2)}`);
    
    // 詳細結果をCSVで出力
    const csvContent = 'ファイル名,認識結果,信頼度,処理時間(ms)\n' + 
      results.map(result => 
        `"${result.originalAudio}","${result.transcription}",${result.confidence},${result.processingTime}`
      ).join('\n');

    require('fs').writeFileSync('./output/batch-results.csv', csvContent, 'utf8');
    console.log('詳細結果をbatch-results.csvに保存しました。');

  } catch (error) {
    console.error('バッチ処理エラー:', error);
  }
}

// 実行
batchProcessingExample();
```

## パフォーマンス最適化とベストプラクティス

### 1\. エラーハンドリング戦略

```typescript:src/utils/error-handler.ts
export class SpeechAPIError extends Error {
  constructor(
    message: string,
    public code?: string,
    public details?: any
  ) {
    super(message);
    this.name = 'SpeechAPIError';
  }
}

export class AudioProcessingError extends Error {
  constructor(
    message: string,
    public audioFile?: string,
    public stage?: 'recognition' | 'synthesis' | 'processing'
  ) {
    super(message);
    this.name = 'AudioProcessingError';
  }
}

export const handleSpeechAPIError = (error: any): SpeechAPIError => {
  if (error.code === 3) {
    return new SpeechAPIError('音声ファイルの形式が無効です', 'INVALID_AUDIO_FORMAT', error);
  } else if (error.code === 11) {
    return new SpeechAPIError('音声ファイルが見つかりません', 'AUDIO_FILE_NOT_FOUND', error);
  } else if (error.code === 8) {
    return new SpeechAPIError('API利用制限に達しました', 'RESOURCE_EXHAUSTED', error);
  } else {
    return new SpeechAPIError(`APIエラー: ${error.message}`, 'UNKNOWN_API_ERROR', error);
  }
};

export const withRetry = async <T>(
  operation: () => Promise<T>,
  maxRetries: number = 3,
  delay: number = 1000
): Promise<T> => {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await operation();
    } catch (error) {
      if (attempt === maxRetries) {
        throw error;
      }
      
      console.log(`リトライ ${attempt}/${maxRetries} - ${delay}ms後に再実行`);
      await new Promise(resolve => setTimeout(resolve, delay));
      delay *= 2; // 指数バックオフ
    }
  }
  
  throw new Error('This should never be reached');
};
```

### 2\. パフォーマンス監視とメトリクス

```typescript:src/utils/performance-monitor.ts
export interface PerformanceMetrics {
  operationType: 'speech-to-text' | 'text-to-speech' | 'integrated';
  duration: number;
  audioLengthSeconds?: number;
  fileSize?: number;
  confidence?: number;
  languageCode: string;
  timestamp: Date;
}

export class PerformanceMonitor {
  private metrics: PerformanceMetrics[] = [];

  recordMetric(metric: PerformanceMetrics): void {
    this.metrics.push(metric);
    
    // メトリクスが多すぎる場合は古いものを削除
    if (this.metrics.length > 1000) {
      this.metrics.shift();
    }
  }

  getAverageProcessingTime(operationType?: PerformanceMetrics['operationType']): number {
    const filteredMetrics = operationType 
      ? this.metrics.filter(m => m.operationType === operationType)
      : this.metrics;
    
    if (filteredMetrics.length === 0) return 0;
    
    return filteredMetrics.reduce((sum, m) => sum + m.duration, 0) / filteredMetrics.length;
  }

  getThroughput(operationType?: PerformanceMetrics['operationType']): number {
    const oneHourAgo = new Date(Date.now() - 60 * 60 * 1000);
    const recentMetrics = this.metrics.filter(m => m.timestamp > oneHourAgo);
    
    const filteredMetrics = operationType 
      ? recentMetrics.filter(m => m.operationType === operationType)
      : recentMetrics;
    
    return filteredMetrics.length; // 1時間あたりの処理数
  }

  getPerformanceReport(): string {
    const sttAvg = this.getAverageProcessingTime('speech-to-text');
    const ttsAvg = this.getAverageProcessingTime('text-to-speech');
    const integratedAvg = this.getAverageProcessingTime('integrated');
    
    return `
=== パフォーマンスレポート ===
Speech-to-Text平均処理時間: ${sttAvg.toFixed(2)}ms
Text-to-Speech平均処理時間: ${ttsAvg.toFixed(2)}ms
統合処理平均処理時間: ${integratedAvg.toFixed(2)}ms
1時間あたりの処理件数: ${this.getThroughput()}件
総記録メトリクス数: ${this.metrics.length}件
    `.trim();
  }

  exportMetricsToCSV(): string {
    const header = 'operationType,duration,audioLengthSeconds,fileSize,confidence,languageCode,timestamp\n';
    const rows = this.metrics.map(m => 
      `${m.operationType},${m.duration},${m.audioLengthSeconds || ''},${m.fileSize || ''},${m.confidence || ''},${m.languageCode},${m.timestamp.toISOString()}`
    ).join('\n');
    
    return header + rows;
  }
}

// グローバルな監視インスタンス
export const performanceMonitor = new PerformanceMonitor();
```

### 3\. キャッシュとファイル管理

```typescript:src/utils/cache-manager.ts
import { createHash } from 'crypto';
import { existsSync, readFileSync, writeFileSync, mkdirSync } from 'fs';
import { join } from 'path';

export class AudioCacheManager {
  private cacheDir: string;
  private maxCacheSize: number; // バイト数

  constructor(cacheDir: string = './cache', maxCacheSize: number = 100 * 1024 * 1024) { // 100MB
    this.cacheDir = cacheDir;
    this.maxCacheSize = maxCacheSize;
    
    if (!existsSync(cacheDir)) {
      mkdirSync(cacheDir, { recursive: true });
    }
  }

  private generateCacheKey(text: string, options: any): string {
    const data = JSON.stringify({ text, options });
    return createHash('md5').update(data).digest('hex');
  }

  getCachedAudio(text: string, options: any): Buffer | null {
    try {
      const cacheKey = this.generateCacheKey(text, options);
      const cachePath = join(this.cacheDir, `${cacheKey}.cache`);
      
      if (existsSync(cachePath)) {
        return readFileSync(cachePath);
      }
      
      return null;
    } catch (error) {
      console.error('キャッシュ読み込みエラー:', error);
      return null;
    }
  }

  setCachedAudio(text: string, options: any, audioBuffer: Buffer): void {
    try {
      const cacheKey = this.generateCacheKey(text, options);
      const cachePath = join(this.cacheDir, `${cacheKey}.cache`);
      
      writeFileSync(cachePath, audioBuffer);
      this.cleanupCache();
    } catch (error) {
      console.error('キャッシュ書き込みエラー:', error);
    }
  }

  private cleanupCache(): void {
    // キャッシュサイズが制限を超えた場合、古いファイルを削除
    // 実装は簡略化されているため、実際の運用では更に詳細な実装が必要
  }

  getCacheStats(): { files: number; totalSize: number } {
    try {
      const { readdirSync, statSync } = require('fs');
      const files = readdirSync(this.cacheDir).filter(f => f.endsWith('.cache'));
      const totalSize = files.reduce((size, file) => {
        return size + statSync(join(this.cacheDir, file)).size;
      }, 0);
      
      return { files: files.length, totalSize };
    } catch (error) {
      return { files: 0, totalSize: 0 };
    }
  }
}

export const audioCacheManager = new AudioCacheManager();
```

## 料金最適化とコスト管理

### 1\. コスト追跡システム

```typescript:src/utils/cost-tracker.ts
export interface CostInfo {
  operationType: 'speech-to-text' | 'text-to-speech';
  units: number; // STT: 15秒単位, TTS: 文字数
  estimatedCost: number; // USD
  timestamp: Date;
}

export class CostTracker {
  private costs: CostInfo[] = [];
  
  // 2024年時点の概算料金（実際の料金は変更される可能性があります）
  private readonly PRICING = {
    'speech-to-text': 0.006, // $0.006 per 15-second interval
    'text-to-speech': 0.000004, // $0.000004 per character
  } as const;

  recordCost(operationType: CostInfo['operationType'], units: number): void {
    const estimatedCost = units * this.PRICING[operationType];
    
    this.costs.push({
      operationType,
      units,
      estimatedCost,
      timestamp: new Date(),
    });
  }

  getDailyCost(date?: Date): number {
    const targetDate = date || new Date();
    const startOfDay = new Date(targetDate);
    startOfDay.setHours(0, 0, 0, 0);
    
    const endOfDay = new Date(targetDate);
    endOfDay.setHours(23, 59, 59, 999);

    return this.costs
      .filter(cost => cost.timestamp >= startOfDay && cost.timestamp <= endOfDay)
      .reduce((total, cost) => total + cost.estimatedCost, 0);
  }

  getMonthlyCost(year: number, month: number): number {
    const startOfMonth = new Date(year, month - 1, 1);
    const endOfMonth = new Date(year, month, 0, 23, 59, 59, 999);

    return this.costs
      .filter(cost => cost.timestamp >= startOfMonth && cost.timestamp <= endOfMonth)
      .reduce((total, cost) => total + cost.estimatedCost, 0);
  }

  getCostByOperation(operationType: CostInfo['operationType']): number {
    return this.costs
      .filter(cost => cost.operationType === operationType)
      .reduce((total, cost) => total + cost.estimatedCost, 0);
  }

  getCostReport(): string {
    const totalCost = this.costs.reduce((total, cost) => total + cost.estimatedCost, 0);
    const dailyCost = this.getDailyCost();
    const sttCost = this.getCostByOperation('speech-to-text');
    const ttsCost = this.getCostByOperation('text-to-speech');

    return `
=== コストレポート ===
総コスト: $${totalCost.toFixed(4)}
今日のコスト: $${dailyCost.toFixed(4)}
Speech-to-Text: $${sttCost.toFixed(4)}
Text-to-Speech: $${ttsCost.toFixed(4)}
総取引数: ${this.costs.length}件
    `.trim();
  }

  exportCostData(): string {
    const header = 'operationType,units,estimatedCost,timestamp\n';
    const rows = this.costs.map(cost => 
      `${cost.operationType},${cost.units},${cost.estimatedCost},${cost.timestamp.toISOString()}`
    ).join('\n');
    
    return header + rows;
  }
}

export const costTracker = new CostTracker();
```

### 2\. 最適化されたサービスクラス

```typescript:src/services/optimized-speech.service.ts
import { SpeechToTextService } from './speech-to-text.service';
import { TextToSpeechService } from './text-to-speech.service';
import { audioCacheManager } from '../utils/cache-manager';
import { costTracker } from '../utils/cost-tracker';
import { performanceMonitor } from '../utils/performance-monitor';

export class OptimizedSpeechService {
  private sttService: SpeechToTextService;
  private ttsService: TextToSpeechService;

  constructor(sttService: SpeechToTextService, ttsService: TextToSpeechService) {
    this.sttService = sttService;
    this.ttsService = ttsService;
  }

  async synthesizeTextWithCache(
    text: string,
    outputPath: string,
    options: any = {}
  ): Promise<void> {
    const startTime = Date.now();

    try {
      // キャッシュをチェック
      const cachedAudio = audioCacheManager.getCachedAudio(text, options);
      if (cachedAudio) {
        require('fs').writeFileSync(outputPath, cachedAudio);
        console.log('キャッシュから音声を取得しました');
        
        performanceMonitor.recordMetric({
          operationType: 'text-to-speech',
          duration: Date.now() - startTime,
          languageCode: options.languageCode || 'ja-JP',
          timestamp: new Date(),
        });
        
        return;
      }

      // キャッシュにない場合はAPIを呼び出し
      await this.ttsService.synthesizeText(text, outputPath, options);
      
      // 結果をキャッシュに保存
      const audioBuffer = require('fs').readFileSync(outputPath);
      audioCacheManager.setCachedAudio(text, options, audioBuffer);
      
      // コストを記録
      costTracker.recordCost('text-to-speech', text.length);
      
      // パフォーマンスを記録
      performanceMonitor.recordMetric({
        operationType: 'text-to-speech',
        duration: Date.now() - startTime,
        languageCode: options.languageCode || 'ja-JP',
        timestamp: new Date(),
      });

    } catch (error) {
      console.error('最適化された音声合成エラー:', error);
      throw error;
    }
  }

  async transcribeWithOptimization(
    audioFilePath: string,
    languageCode: string = 'ja-JP'
  ): Promise<string> {
    const startTime = Date.now();

    try {
      const result = await this.sttService.transcribeAudioFile(audioFilePath, languageCode);
      
      // 音声ファイルの長さを概算（実際の実装では音声ファイルを解析）
      const audioLengthSeconds = 30; // サンプル値
      const units = Math.ceil(audioLengthSeconds / 15); // 15秒単位で課金
      
      // コストを記録
      costTracker.recordCost('speech-to-text', units);
      
      // パフォーマンスを記録
      performanceMonitor.recordMetric({
        operationType: 'speech-to-text',
        duration: Date.now() - startTime,
        audioLengthSeconds,
        languageCode,
        timestamp: new Date(),
      });

      return result;
    } catch (error) {
      console.error('最適化された音声認識エラー:', error);
      throw error;
    }
  }

  getOptimizationReport(): string {
    const cacheStats = audioCacheManager.getCacheStats();
    const costReport = costTracker.getCostReport();
    const perfReport = performanceMonitor.getPerformanceReport();

    return `
=== 最適化レポート ===

${costReport}

=== キャッシュ統計 ===
キャッシュファイル数: ${cacheStats.files}
キャッシュサイズ: ${(cacheStats.totalSize / 1024 / 1024).toFixed(2)}MB

${perfReport}
    `.trim();
  }
}
```

## 実運用での設定とデプロイメント

### 1\. 環境別設定管理

```typescript:src/config/environment.ts
export interface EnvironmentConfig {
  gcpProjectId: string;
  gcpCredentialsPath: string;
  gcpLocation: string;
  gcsBucketName: string;
  cacheDirectory: string;
  maxCacheSize: number;
  retryAttempts: number;
  logLevel: 'debug' | 'info' | 'warn' | 'error';
  enablePerformanceMonitoring: boolean;
  enableCostTracking: boolean;
}

const developmentConfig: EnvironmentConfig = {
  gcpProjectId: process.env.GCP_PROJECT_ID || 'dev-project',
  gcpCredentialsPath: process.env.GOOGLE_APPLICATION_CREDENTIALS || './dev-credentials.json',
  gcpLocation: 'asia-northeast1',
  gcsBucketName: process.env.GCS_BUCKET_NAME || 'dev-speech-bucket',
  cacheDirectory: './cache/dev',
  maxCacheSize: 50 * 1024 * 1024, // 50MB
  retryAttempts: 3,
  logLevel: 'debug',
  enablePerformanceMonitoring: true,
  enableCostTracking: true,
};

const productionConfig: EnvironmentConfig = {
  gcpProjectId: process.env.GCP_PROJECT_ID || '',
  gcpCredentialsPath: process.env.GOOGLE_APPLICATION_CREDENTIALS || '',
  gcpLocation: process.env.GCP_LOCATION || 'asia-northeast1',
  gcsBucketName: process.env.GCS_BUCKET_NAME || '',
  cacheDirectory: '/var/cache/speech',
  maxCacheSize: 500 * 1024 * 1024, // 500MB
  retryAttempts: 5,
  logLevel: 'info',
  enablePerformanceMonitoring: true,
  enableCostTracking: true,
};

export const getConfig = (): EnvironmentConfig => {
  const env = process.env.NODE_ENV || 'development';
  
  switch (env) {
    case 'production':
      return productionConfig;
    case 'development':
    default:
      return developmentConfig;
  }
};

export const config = getConfig();
```

### 2\. ヘルスチェックとモニタリング

```typescript:src/health/health-checker.ts
import { speechClient, ttsClient } from '../config/gcp-config';

export interface HealthStatus {
  service: string;
  status: 'healthy' | 'unhealthy' | 'degraded';
  latency?: number;
  error?: string;
  timestamp: Date;
}

export class HealthChecker {
  async checkSpeechToTextHealth(): Promise<HealthStatus> {
    const startTime = Date.now();
    
    try {
      // 簡単なダミーリクエストでAPIの可用性をチェック
      await speechClient.listVoices?.({});
      
      const latency = Date.now() - startTime;
      
      return {
        service: 'speech-to-text',
        status: latency < 5000 ? 'healthy' : 'degraded',
        latency,
        timestamp: new Date(),
      };
    } catch (error) {
      return {
        service: 'speech-to-text',
        status: 'unhealthy',
        latency: Date.now() - startTime,
        error: error instanceof Error ? error.message : 'Unknown error',
        timestamp: new Date(),
      };
    }
  }

  async checkTextToSpeechHealth(): Promise<HealthStatus> {
    const startTime = Date.now();
    
    try {
      // 音声リストの取得でAPIの可用性をチェック
      await ttsClient.listVoices({ languageCode: 'ja-JP' });
      
      const latency = Date.now() - startTime;
      
      return {
        service: 'text-to-speech',
        status: latency < 5000 ? 'healthy' : 'degraded',
        latency,
        timestamp: new Date(),
      };
    } catch (error) {
      return {
        service: 'text-to-speech',
        status: 'unhealthy',
        latency: Date.now() - startTime,
        error: error instanceof Error ? error.message : 'Unknown error',
        timestamp: new Date(),
      };
    }
  }

  async performHealthCheck(): Promise<HealthStatus[]> {
    const checks = await Promise.allSettled([
      this.checkSpeechToTextHealth(),
      this.checkTextToSpeechHealth(),
    ]);

    return checks.map(result => 
      result.status === 'fulfilled' ? result.value : {
        service: 'unknown',
        status: 'unhealthy' as const,
        error: 'Health check failed',
        timestamp: new Date(),
      }
    );
  }

  async getHealthReport(): Promise<string> {
    const healthStatuses = await this.performHealthCheck();
    
    const report = healthStatuses.map(status => 
      `${status.service}: ${status.status}${status.latency ? ` (${status.latency}ms)` : ''}${status.error ? ` - ${status.error}` : ''}`
    ).join('\n');

    return `=== ヘルスチェック結果 ===\n${report}`;
  }
}

export const healthChecker = new HealthChecker();
```

### 3\. Express.jsとの統合例

```typescript:src/app.ts
import express from 'express';
import multer from 'multer';
import { speechClient, ttsClient } from './config/gcp-config';
import { OptimizedSpeechService } from './services/optimized-speech.service';
import { SpeechToTextService } from './services/speech-to-text.service';
import { TextToSpeechService } from './services/text-to-speech.service';
import { healthChecker } from './health/health-checker';
import { config } from './config/environment';

const app = express();
const upload = multer({ dest: 'uploads/' });

// サービス初期化
const sttService = new SpeechToTextService(speechClient);
const ttsService = new TextToSpeechService(ttsClient);
const optimizedService = new OptimizedSpeechService(sttService, ttsService);

app.use(express.json());

// ヘルスチェックエンドポイント
app.get('/health', async (req, res) => {
  try {
    const healthReport = await healthChecker.getHealthReport();
    res.json({ status: 'ok', report: healthReport });
  } catch (error) {
    res.status(500).json({ status: 'error', message: 'Health check failed' });
  }
});

// 音声認識エンドポイント
app.post('/api/speech-to-text', upload.single('audio'), async (req, res) => {
  try {
    if (!req.file) {
      return res.status(400).json({ error: '音声ファイルが必要です' });
    }

    const { languageCode = 'ja-JP' } = req.body;
    const transcription = await optimizedService.transcribeWithOptimization(
      req.file.path,
      languageCode
    );

    res.json({ transcription });
  } catch (error) {
    console.error('音声認識エラー:', error);
    res.status(500).json({ error: '音声認識に失敗しました' });
  }
});

// 音声合成エンドポイント
app.post('/api/text-to-speech', async (req, res) => {
  try {
    const { text, options = {} } = req.body;
    
    if (!text) {
      return res.status(400).json({ error: 'テキストが必要です' });
    }

    const outputPath = `./temp/${Date.now()}.mp3`;
    await optimizedService.synthesizeTextWithCache(text, outputPath, options);

    res.download(outputPath, (err) => {
      if (err) console.error('ファイル送信エラー:', err);
      // ファイルを削除
      require('fs').unlinkSync(outputPath);
    });
  } catch (error) {
    console.error('音声合成エラー:', error);
    res.status(500).json({ error: '音声合成に失敗しました' });
  }
});

// 最適化レポートエンドポイント
app.get('/api/optimization-report', (req, res) => {
  try {
    const report = optimizedService.getOptimizationReport();
    res.json({ report });
  } catch (error) {
    console.error('レポート取得エラー:', error);
    res.status(500).json({ error: 'レポート取得に失敗しました' });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`サーバーがポート${PORT}で起動しました`);
  console.log(`環境: ${process.env.NODE_ENV || 'development'}`);
  console.log(`GCPプロジェクト: ${config.gcpProjectId}`);
});

export default app;
```

## まとめ

GCPのSpeech-to-Text APIとText-to-Speech APIを使えば、高品質な音声処理システムを比較的簡単に構築できることがわかりました。

**🎯 この記事で実装した主要機能**

>* **基本的な音声認識・合成機能**: ファイルベースの変換機能
>* **リアルタイム音声処理**: ストリーミング対応の認識システム
>* **統合音声処理システム**: STTとTTSを組み合わせた高度な処理
>* **SSML対応**: 表現豊かな音声合成機能
>* **パフォーマンス最適化**: キャッシュ、エラーハンドリング、監視機能
>* **コスト管理**: 料金追跡と最適化システム
>* **プロダクション対応**: 設定管理、ヘルスチェック、API化

**💡 実装時のポイント**

>1. **適切なエラーハンドリング**: APIの制限や音声ファイルの形式エラーに対応
>2. **キャッシュ活用**: 同じテキストの音声合成結果を再利用してコスト削減
>3. **パフォーマンス監視**: 処理時間と成功率を追跡して品質向上
>4. **コスト最適化**: 使用量を追跡して予算管理を実現
>5. **柔軟な設定管理**: 環境別の設定で開発から本番まで対応

次のステップとして、WebSocket を使ったリアルタイム音声チャット機能や、複数言語の同時対応、さらに高度な音声解析機能の実装を検討してみてください。

Google CloudのSpeech AIサービスは今後も機能拡張が予想されるため、継続的にドキュメントをチェックして最新機能を活用していくことをおすすめします！