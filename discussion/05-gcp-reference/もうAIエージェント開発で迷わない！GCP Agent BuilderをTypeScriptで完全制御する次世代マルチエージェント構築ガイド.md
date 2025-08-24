# もうAIエージェント開発で迷わない！GCP Agent BuilderをTypeScriptで完全制御する次世代マルチエージェント構築ガイド

## はじめに

「AIエージェントを作りたいけど、複雑すぎて何から始めればいいか分からない...」「LangChainやCrewAIは試したけど、Google製品と連携しやすいフレームワークはないの？」

そんな悩みを持つ開発者の皆さんに朗報です！GoogleのAgent Development Kit (ADK) + Agent Builderなら、TypeScriptでマルチエージェントシステムを簡単に構築できるんです。

本記事では、GCP Agent BuilderとADKをTypeScriptで活用する方法から、実際のWebアプリケーションへの組み込みまで、現場ですぐ使える実装パターンを徹底解説します。

## GCP Agent Builderとは？Googleが提供する次世代AIエージェント開発プラットフォーム

**Agent Builderの全体構成**

GCP Agent Builderは、AIエージェント開発のライフサイクル全体をカバーする統合プラットフォームです。以下の3つの主要コンポーネントで構成されています。

| コンポーネント | 役割 | 現在の状況 |
|---------------|------|----------|
| **Agent Garden** | サンプルエージェント・ツールライブラリ | Preview |
| **Agent Development Kit (ADK)** | オープンソース開発フレームワーク | Preview |
| **Vertex AI Agent Engine** | マネージドサービス・本番運用基盤 | Preview |

**💡 Agent Builderの強み**

>* Googleエコシステム（Search、Vertex AI、Cloud Tools）との深い統合
>* マルチエージェント協調システムの簡単な構築
>* 双方向音声・動画ストリーミング対応
>* エンタープライズグレードのスケーラビリティとセキュリティ
>* オープンソース（ADK）+ マネージドサービスのハイブリッド構成

## TypeScriptでAIエージェントを構築

### 1\. 環境構築とセットアップ

まず、TypeScriptプロジェクトでGoogle Gen AI SDKを使用する準備をしましょう。

```bash
# プロジェクトの初期化
mkdir agent-builder-app && cd agent-builder-app
npm init -y

# 必要なパッケージのインストール
npm install @google/genai dotenv express
npm install -D typescript @types/node @types/express ts-node nodemon
```

```typescript:src/config/index.ts
import * as dotenv from 'dotenv'

dotenv.config()

export const config = {
  // Gemini Developer API用
  geminiApiKey: process.env.GEMINI_API_KEY!,
  
  // Vertex AI用
  gcpProjectId: process.env.GCP_PROJECT_ID,
  gcpLocation: process.env.GCP_LOCATION || 'us-central1',
  
  // アプリケーション設定
  port: process.env.PORT || 3000,
  nodeEnv: process.env.NODE_ENV || 'development'
} as const

// 環境変数の検証
if (!config.geminiApiKey && !config.gcpProjectId) {
  throw new Error('Either GEMINI_API_KEY or GCP_PROJECT_ID must be provided')
}
```

### 2\. 基本的なAIエージェントクラスの実装

```typescript:src/agents/BaseAgent.ts
import { GoogleGenAI, GenerateContentResponse } from '@google/genai'
import { config } from '../config'

export interface AgentConfig {
  name: string
  description: string
  instructions: string
  model?: string
  temperature?: number
  maxTokens?: number
}

export interface AgentMessage {
  role: 'user' | 'assistant' | 'system'
  content: string
  timestamp: Date
}

export interface AgentContext {
  conversationHistory: AgentMessage[]
  metadata?: Record<string, any>
}

export class BaseAgent {
  private genAI: GoogleGenAI
  private agentConfig: AgentConfig
  private context: AgentContext

  constructor(agentConfig: AgentConfig) {
    this.genAI = new GoogleGenAI({
      apiKey: config.geminiApiKey,
    })
    
    this.agentConfig = {
      model: 'gemini-2.0-flash-exp',
      temperature: 0.7,
      maxTokens: 1024,
      ...agentConfig
    }
    
    this.context = {
      conversationHistory: []
    }
  }

  async generateResponse(userMessage: string): Promise<string> {
    try {
      // メッセージを履歴に追加
      this.addMessageToHistory('user', userMessage)
      
      // プロンプトの構築
      const prompt = this.buildPrompt(userMessage)
      
      // Gemini APIでレスポンス生成
      const response = await this.genAI.models.generateContent({
        model: this.agentConfig.model!,
        contents: [{
          role: 'user',
          parts: [{ text: prompt }]
        }],
        generationConfig: {
          temperature: this.agentConfig.temperature,
          maxOutputTokens: this.agentConfig.maxTokens
        }
      })

      const responseText = this.extractResponseText(response)
      
      // レスポンスを履歴に追加
      this.addMessageToHistory('assistant', responseText)
      
      return responseText
    } catch (error) {
      console.error(`Agent ${this.agentConfig.name} error:`, error)
      throw new Error(`Response generation failed: ${error.message}`)
    }
  }

  private buildPrompt(userMessage: string): string {
    const systemPrompt = `
あなたは${this.agentConfig.name}です。
役割: ${this.agentConfig.description}

指示事項:
${this.agentConfig.instructions}

過去の会話履歴:
${this.getFormattedHistory()}

現在のユーザーメッセージ: ${userMessage}

上記の役割と指示事項に従って、適切に応答してください。
    `.trim()
    
    return systemPrompt
  }

  private extractResponseText(response: GenerateContentResponse): string {
    return response.candidates?.[0]?.content?.parts?.[0]?.text || 
           'Sorry, I could not generate a response.'
  }

  private addMessageToHistory(role: 'user' | 'assistant' | 'system', content: string): void {
    this.context.conversationHistory.push({
      role,
      content,
      timestamp: new Date()
    })
    
    // 履歴が長くなりすぎた場合の制限
    if (this.context.conversationHistory.length > 20) {
      this.context.conversationHistory = this.context.conversationHistory.slice(-20)
    }
  }

  private getFormattedHistory(): string {
    return this.context.conversationHistory
      .slice(-5) // 最近の5件のみ表示
      .map(msg => `${msg.role}: ${msg.content}`)
      .join('\n')
  }

  // 外部ツールとの連携用メソッド
  async callTool(toolName: string, parameters: Record<string, any>): Promise<any> {
    // サブクラスで実装される抽象メソッド
    throw new Error('Tool calling not implemented in base agent')
  }

  // 会話履歴のクリア
  clearHistory(): void {
    this.context.conversationHistory = []
  }

  // エージェント情報の取得
  getInfo(): AgentConfig {
    return { ...this.agentConfig }
  }
}
```

## 専門エージェントの実装：実用的なユースケース

### 1\. 検索エージェント：Google Search連携

```typescript:src/agents/SearchAgent.ts
import { BaseAgent, AgentConfig } from './BaseAgent'
import axios from 'axios'

export class SearchAgent extends BaseAgent {
  private searchApiKey: string
  private searchEngineId: string

  constructor() {
    const agentConfig: AgentConfig = {
      name: 'SearchAgent',
      description: 'インターネット検索を実行して最新情報を提供するエージェント',
      instructions: `
あなたは情報検索の専門家です。
- ユーザーの質問に対して適切な検索クエリを生成
- 検索結果から関連性の高い情報を抽出
- 複数の情報源を比較して信頼性の高い回答を提供
- 検索できない場合は代替案を提示
      `
    }
    
    super(agentConfig)
    
    this.searchApiKey = process.env.GOOGLE_SEARCH_API_KEY!
    this.searchEngineId = process.env.GOOGLE_SEARCH_ENGINE_ID!
  }

  async callTool(toolName: string, parameters: Record<string, any>): Promise<any> {
    switch (toolName) {
      case 'googleSearch':
        return await this.performGoogleSearch(parameters.query)
      default:
        throw new Error(`Unknown tool: ${toolName}`)
    }
  }

  private async performGoogleSearch(query: string): Promise<any> {
    try {
      const response = await axios.get('https://www.googleapis.com/customsearch/v1', {
        params: {
          key: this.searchApiKey,
          cx: this.searchEngineId,
          q: query,
          num: 5 // 上位5件を取得
        }
      })

      return {
        items: response.data.items?.map(item => ({
          title: item.title,
          snippet: item.snippet,
          link: item.link
        })) || []
      }
    } catch (error) {
      throw new Error(`Search failed: ${error.message}`)
    }
  }

  async searchAndAnswer(userQuestion: string): Promise<string> {
    try {
      // Step 1: 検索クエリを生成
      const searchQuery = await this.generateSearchQuery(userQuestion)
      
      // Step 2: 検索実行
      const searchResults = await this.callTool('googleSearch', { query: searchQuery })
      
      // Step 3: 検索結果を含めて最終回答生成
      const contextualPrompt = `
ユーザーの質問: ${userQuestion}

検索結果:
${searchResults.items.map((item: any, index: number) => `
${index + 1}. ${item.title}
   ${item.snippet}
   URL: ${item.link}
`).join('\n')}

上記の検索結果を参考に、ユーザーの質問に対して正確で有用な回答を提供してください。
情報の出典も明記してください。
      `
      
      return await this.generateResponse(contextualPrompt)
    } catch (error) {
      return `検索中にエラーが発生しました: ${error.message}`
    }
  }

  private async generateSearchQuery(userQuestion: string): Promise<string> {
    const prompt = `
以下のユーザーの質問に対して、最も効果的なGoogle検索クエリを生成してください。
質問: ${userQuestion}

検索クエリのみを返してください（説明不要）:
    `
    
    return await this.generateResponse(prompt)
  }
}
```

### 2\. コードレビューエージェント：技術的な分析

```typescript:src/agents/CodeReviewAgent.ts
import { BaseAgent, AgentConfig } from './BaseAgent'

export interface CodeReviewRequest {
  code: string
  language: string
  context?: string
  focusAreas?: string[]
}

export interface CodeReviewResult {
  overallScore: number
  issues: Array<{
    type: 'error' | 'warning' | 'suggestion'
    message: string
    line?: number
    severity: 'high' | 'medium' | 'low'
  }>
  improvements: string[]
  summary: string
}

export class CodeReviewAgent extends BaseAgent {
  constructor() {
    const agentConfig: AgentConfig = {
      name: 'CodeReviewAgent',
      description: 'ソースコードの品質分析とレビューを行う専門エージェント',
      instructions: `
あなたは経験豊富なソフトウェアエンジニアです。
コードレビューでは以下の観点から分析してください:

1. **コードの正確性**: バグや論理エラーの検出
2. **可読性**: 変数名、関数名、コメントの適切性
3. **保守性**: 修正・拡張のしやすさ
4. **パフォーマンス**: 実行効率の最適化ポイント
5. **セキュリティ**: 脆弱性や潜在的リスク
6. **ベストプラクティス**: 言語固有の慣例遵守

建設的で具体的なフィードバックを提供してください。
      `
    }
    
    super(agentConfig)
  }

  async reviewCode(request: CodeReviewRequest): Promise<CodeReviewResult> {
    try {
      const reviewPrompt = `
コードレビュー依頼:

言語: ${request.language}
${request.context ? `コンテキスト: ${request.context}` : ''}
${request.focusAreas ? `重点的にチェックする項目: ${request.focusAreas.join(', ')}` : ''}

コード:
\`\`\`${request.language}
${request.code}
\`\`\`

以下のJSON形式で回答してください:
{
  "overallScore": 1-10の数値,
  "issues": [
    {
      "type": "error|warning|suggestion",
      "message": "具体的な問題点",
      "line": 行番号(該当する場合),
      "severity": "high|medium|low"
    }
  ],
  "improvements": ["改善提案1", "改善提案2"],
  "summary": "総合的な評価コメント"
}
      `
      
      const response = await this.generateResponse(reviewPrompt)
      return this.parseReviewResponse(response)
    } catch (error) {
      throw new Error(`Code review failed: ${error.message}`)
    }
  }

  private parseReviewResponse(response: string): CodeReviewResult {
    try {
      // JSON形式でないレスポンスの場合のフォールバック処理
      const jsonMatch = response.match(/\{[\s\S]*\}/)?.[0]
      if (jsonMatch) {
        return JSON.parse(jsonMatch)
      }
      
      // フォールバック：構造化されたレスポンス生成
      return {
        overallScore: 7,
        issues: [{
          type: 'suggestion',
          message: 'Detailed analysis could not be parsed, but general feedback is available',
          severity: 'medium'
        }],
        improvements: ['Consider following standard coding conventions'],
        summary: response
      }
    } catch (error) {
      throw new Error(`Failed to parse review response: ${error.message}`)
    }
  }

  async getBestPractices(language: string): Promise<string[]> {
    const prompt = `
${language}プログラミング言語における主要なベストプラクティスを
実務で特に重要なものを中心に5〜8個リストアップしてください。

形式: 
- 実践項目1
- 実践項目2
...
    `
    
    const response = await this.generateResponse(prompt)
    return response.split('\n')
      .filter(line => line.trim().startsWith('-'))
      .map(line => line.replace(/^-\s*/, '').trim())
  }
}
```

## マルチエージェントシステム：協調動作の実装

**エージェントオーケストレーター**

```typescript:src/orchestration/AgentOrchestrator.ts
import { BaseAgent } from '../agents/BaseAgent'

export interface TaskRequest {
  description: string
  priority: 'low' | 'medium' | 'high'
  requiredCapabilities: string[]
  context?: Record<string, any>
}

export interface TaskResult {
  success: boolean
  result?: any
  error?: string
  executedBy: string
  executionTime: number
}

export class AgentOrchestrator {
  private agents: Map<string, BaseAgent> = new Map()
  private capabilities: Map<string, string[]> = new Map()

  registerAgent(name: string, agent: BaseAgent, capabilities: string[]): void {
    this.agents.set(name, agent)
    this.capabilities.set(name, capabilities)
  }

  async executeTask(task: TaskRequest): Promise<TaskResult> {
    const startTime = Date.now()
    
    try {
      // Step 1: 適切なエージェントを選択
      const selectedAgent = this.selectBestAgent(task)
      
      if (!selectedAgent) {
        return {
          success: false,
          error: 'No suitable agent found for this task',
          executedBy: 'orchestrator',
          executionTime: Date.now() - startTime
        }
      }

      // Step 2: タスクを実行
      const result = await selectedAgent.agent.generateResponse(
        this.buildTaskPrompt(task, selectedAgent.capabilities)
      )

      return {
        success: true,
        result,
        executedBy: selectedAgent.name,
        executionTime: Date.now() - startTime
      }
    } catch (error) {
      return {
        success: false,
        error: error.message,
        executedBy: 'unknown',
        executionTime: Date.now() - startTime
      }
    }
  }

  private selectBestAgent(task: TaskRequest): { name: string; agent: BaseAgent; capabilities: string[] } | null {
    let bestMatch: { name: string; score: number; agent: BaseAgent; capabilities: string[] } | null = null

    for (const [agentName, agent] of this.agents) {
      const agentCapabilities = this.capabilities.get(agentName) || []
      const matchScore = this.calculateMatchScore(task.requiredCapabilities, agentCapabilities)
      
      if (matchScore > 0 && (!bestMatch || matchScore > bestMatch.score)) {
        bestMatch = {
          name: agentName,
          score: matchScore,
          agent,
          capabilities: agentCapabilities
        }
      }
    }

    return bestMatch ? {
      name: bestMatch.name,
      agent: bestMatch.agent,
      capabilities: bestMatch.capabilities
    } : null
  }

  private calculateMatchScore(required: string[], available: string[]): number {
    const matches = required.filter(req => available.includes(req))
    return matches.length / required.length
  }

  private buildTaskPrompt(task: TaskRequest, agentCapabilities: string[]): string {
    return `
タスクの詳細:
${task.description}

優先度: ${task.priority}
あなたの能力: ${agentCapabilities.join(', ')}
${task.context ? `追加コンテキスト: ${JSON.stringify(task.context)}` : ''}

このタスクを実行してください。
    `
  }

  // 複雑なタスクの分解と並列実行
  async executeComplexTask(
    mainTask: string, 
    subtasks: string[]
  ): Promise<{ mainResult: TaskResult; subtaskResults: TaskResult[] }> {
    
    // サブタスクを並列実行
    const subtaskPromises = subtasks.map(subtask => 
      this.executeTask({
        description: subtask,
        priority: 'medium',
        requiredCapabilities: ['general']
      })
    )

    const subtaskResults = await Promise.all(subtaskPromises)
    
    // サブタスクの結果をまとめてメインタスクを実行
    const combinedContext = subtaskResults
      .filter(result => result.success)
      .map((result, index) => `サブタスク${index + 1}の結果: ${result.result}`)
      .join('\n')

    const mainResult = await this.executeTask({
      description: `${mainTask}\n\nサブタスクの実行結果:\n${combinedContext}`,
      priority: 'high',
      requiredCapabilities: ['synthesis', 'analysis']
    })

    return {
      mainResult,
      subtaskResults
    }
  }

  listAvailableAgents(): Array<{ name: string; capabilities: string[] }> {
    return Array.from(this.capabilities.entries()).map(([name, capabilities]) => ({
      name,
      capabilities
    }))
  }
}
```

## Web API実装：Express.jsでの統合

```typescript:src/server/app.ts
import express, { Request, Response } from 'express'
import cors from 'cors'
import { config } from '../config'
import { AgentOrchestrator, TaskRequest } from '../orchestration/AgentOrchestrator'
import { SearchAgent } from '../agents/SearchAgent'
import { CodeReviewAgent } from '../agents/CodeReviewAgent'
import { BaseAgent } from '../agents/BaseAgent'

const app = express()
const orchestrator = new AgentOrchestrator()

// ミドルウェア設定
app.use(cors())
app.use(express.json({ limit: '10mb' }))
app.use(express.urlencoded({ extended: true }))

// エージェントの初期化と登録
function initializeAgents() {
  // 検索エージェント
  const searchAgent = new SearchAgent()
  orchestrator.registerAgent('search', searchAgent, ['web-search', 'information-retrieval'])

  // コードレビューエージェント
  const codeReviewAgent = new CodeReviewAgent()
  orchestrator.registerAgent('code-review', codeReviewAgent, ['code-analysis', 'quality-assurance'])

  // 汎用アシスタントエージェント
  const generalAgent = new BaseAgent({
    name: 'GeneralAssistant',
    description: '汎用的な質問応答を行うアシスタント',
    instructions: 'ユーザーの質問に対して親切で正確な回答を提供してください。'
  })
  orchestrator.registerAgent('general', generalAgent, ['general', 'conversation'])
}

// エージェント一覧取得
app.get('/agents', (req: Request, res: Response) => {
  const agents = orchestrator.listAvailableAgents()
  res.json({
    success: true,
    data: { agents }
  })
})

// タスク実行エンドポイント
app.post('/execute-task', async (req: Request, res: Response) => {
  try {
    const task: TaskRequest = req.body

    if (!task.description || !task.requiredCapabilities) {
      return res.status(400).json({
        success: false,
        error: 'Description and requiredCapabilities are required'
      })
    }

    const result = await orchestrator.executeTask(task)
    
    res.json({
      success: true,
      data: result
    })
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

// 検索専用エンドポイント
app.post('/search', async (req: Request, res: Response) => {
  try {
    const { query } = req.body

    if (!query) {
      return res.status(400).json({
        success: false,
        error: 'Search query is required'
      })
    }

    const result = await orchestrator.executeTask({
      description: `インターネット検索を実行: ${query}`,
      priority: 'medium',
      requiredCapabilities: ['web-search']
    })

    res.json({
      success: true,
      data: result
    })
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

// コードレビューエンドポイント
app.post('/code-review', async (req: Request, res: Response) => {
  try {
    const { code, language, context } = req.body

    if (!code || !language) {
      return res.status(400).json({
        success: false,
        error: 'Code and language are required'
      })
    }

    const result = await orchestrator.executeTask({
      description: `コードレビューを実行: ${language}コードの品質分析`,
      priority: 'medium',
      requiredCapabilities: ['code-analysis'],
      context: { code, language, context }
    })

    res.json({
      success: true,
      data: result
    })
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

// 複雑タスク処理エンドポイント
app.post('/complex-task', async (req: Request, res: Response) => {
  try {
    const { mainTask, subtasks } = req.body

    if (!mainTask || !Array.isArray(subtasks)) {
      return res.status(400).json({
        success: false,
        error: 'mainTask and subtasks array are required'
      })
    }

    const result = await orchestrator.executeComplexTask(mainTask, subtasks)
    
    res.json({
      success: true,
      data: result
    })
  } catch (error) {
    res.status(500).json({
      success: false,
      error: error.message
    })
  }
})

// ヘルスチェック
app.get('/health', (req: Request, res: Response) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    version: '1.0.0'
  })
})

// サーバー起動
const startServer = () => {
  initializeAgents()
  
  app.listen(config.port, () => {
    console.log(`🚀 Agent Builder Server running on port ${config.port}`)
    console.log(`Environment: ${config.nodeEnv}`)
  })
}

export { app, startServer }
```

## 実践的な活用例とベストプラクティス

### 1\. CLIツールでのマルチエージェント活用

```typescript:src/cli/agent-cli.ts
import { Command } from 'commander'
import { AgentOrchestrator } from '../orchestration/AgentOrchestrator'
import { SearchAgent } from '../agents/SearchAgent'
import { CodeReviewAgent } from '../agents/CodeReviewAgent'
import * as fs from 'fs/promises'

const program = new Command()
const orchestrator = new AgentOrchestrator()

// エージェントの初期化
function initializeCLIAgents() {
  orchestrator.registerAgent('search', new SearchAgent(), ['web-search', 'research'])
  orchestrator.registerAgent('code-review', new CodeReviewAgent(), ['code-analysis', 'quality'])
}

program
  .name('agent-cli')
  .description('Multi-Agent System CLI')
  .version('1.0.0')

program
  .command('ask')
  .description('Ask a question to the most suitable agent')
  .requiredOption('-q, --question <question>', 'Question to ask')
  .option('-c, --capabilities <capabilities>', 'Required capabilities (comma-separated)', 'general')
  .action(async (options) => {
    try {
      const capabilities = options.capabilities.split(',').map(c => c.trim())
      
      const result = await orchestrator.executeTask({
        description: options.question,
        priority: 'medium',
        requiredCapabilities: capabilities
      })

      if (result.success) {
        console.log(`✅ Response from ${result.executedBy}:`)
        console.log(result.result)
        console.log(`⏱️ Executed in ${result.executionTime}ms`)
      } else {
        console.error('❌ Task failed:', result.error)
      }
    } catch (error) {
      console.error('❌ CLI Error:', error.message)
    }
  })

program
  .command('review-file')
  .description('Review code from a file')
  .requiredOption('-f, --file <file>', 'Path to code file')
  .option('-l, --language <language>', 'Programming language', 'javascript')
  .action(async (options) => {
    try {
      const code = await fs.readFile(options.file, 'utf-8')
      
      const result = await orchestrator.executeTask({
        description: `コードレビュー: ${options.file}の品質分析`,
        priority: 'high',
        requiredCapabilities: ['code-analysis'],
        context: { code, language: options.language, filePath: options.file }
      })

      if (result.success) {
        console.log(`📋 Code Review Results for ${options.file}:`)
        console.log(result.result)
      } else {
        console.error('❌ Review failed:', result.error)
      }
    } catch (error) {
      console.error('❌ File reading error:', error.message)
    }
  })

program
  .command('research')
  .description('Research a topic using web search')
  .requiredOption('-t, --topic <topic>', 'Research topic')
  .option('-d, --depth <depth>', 'Research depth (1-3)', '2')
  .action(async (options) => {
    try {
      const depth = parseInt(options.depth)
      const subtasks = []
      
      // 研究の深度に応じてサブタスクを生成
      for (let i = 1; i <= depth; i++) {
        subtasks.push(`${options.topic}について段階${i}の詳細調査を実行`)
      }

      const result = await orchestrator.executeComplexTask(
        `${options.topic}に関する包括的な研究レポートを作成`,
        subtasks
      )

      console.log('🔍 Research Results:')
      console.log('\n📊 Main Report:')
      console.log(result.mainResult.result)
      
      console.log('\n📝 Sub-research Results:')
      result.subtaskResults.forEach((subResult, index) => {
        console.log(`\n${index + 1}. ${subResult.success ? '✅' : '❌'} ${subResult.result || subResult.error}`)
      })
    } catch (error) {
      console.error('❌ Research failed:', error.message)
    }
  })

program
  .command('agents')
  .description('List all available agents')
  .action(() => {
    const agents = orchestrator.listAvailableAgents()
    console.log('🤖 Available Agents:')
    agents.forEach(agent => {
      console.log(`\n• ${agent.name}`)
      console.log(`  Capabilities: ${agent.capabilities.join(', ')}`)
    })
  })

// CLI初期化と実行
initializeCLIAgents()
program.parse()
```

### 2\. キャッシュとレート制限の実装

```typescript:src/utils/CacheManager.ts
export interface CacheEntry<T> {
  data: T
  timestamp: number
  ttl: number
}

export class CacheManager<T> {
  private cache: Map<string, CacheEntry<T>> = new Map()
  private defaultTTL: number

  constructor(defaultTTL: number = 300000) { // 5分
    this.defaultTTL = defaultTTL
  }

  set(key: string, data: T, ttl?: number): void {
    this.cache.set(key, {
      data,
      timestamp: Date.now(),
      ttl: ttl || this.defaultTTL
    })
  }

  get(key: string): T | null {
    const entry = this.cache.get(key)
    
    if (!entry) return null
    
    if (Date.now() - entry.timestamp > entry.ttl) {
      this.cache.delete(key)
      return null
    }
    
    return entry.data
  }

  clear(): void {
    this.cache.clear()
  }

  size(): number {
    return this.cache.size
  }
}
```

### 3\. メトリクス収集とモニタリング

```typescript:src/monitoring/MetricsCollector.ts
export interface AgentMetrics {
  agentName: string
  requestCount: number
  successCount: number
  failureCount: number
  averageResponseTime: number
  lastUsed: Date
}

export class MetricsCollector {
  private metrics: Map<string, AgentMetrics> = new Map()

  recordRequest(agentName: string, success: boolean, responseTime: number): void {
    const current = this.metrics.get(agentName) || {
      agentName,
      requestCount: 0,
      successCount: 0,
      failureCount: 0,
      averageResponseTime: 0,
      lastUsed: new Date()
    }

    current.requestCount++
    current.lastUsed = new Date()
    
    if (success) {
      current.successCount++
    } else {
      current.failureCount++
    }

    // 移動平均の計算
    current.averageResponseTime = 
      (current.averageResponseTime * (current.requestCount - 1) + responseTime) / 
      current.requestCount

    this.metrics.set(agentName, current)
  }

  getMetrics(): AgentMetrics[] {
    return Array.from(this.metrics.values())
  }

  getHealthStatus(): { status: 'healthy' | 'degraded' | 'unhealthy', details: any } {
    const metrics = this.getMetrics()
    const totalRequests = metrics.reduce((sum, m) => sum + m.requestCount, 0)
    const totalFailures = metrics.reduce((sum, m) => sum + m.failureCount, 0)
    
    const successRate = totalRequests > 0 ? (totalRequests - totalFailures) / totalRequests : 1
    
    if (successRate > 0.95) return { status: 'healthy', details: { successRate } }
    if (successRate > 0.8) return { status: 'degraded', details: { successRate } }
    return { status: 'unhealthy', details: { successRate } }
  }
}
```

## まとめ

GCP Agent BuilderとADKをTypeScriptで活用することで、スケーラブルで実用的なマルチエージェントシステムを構築できます。

**✅ 今日から実践できること**

>* Google Gen AI SDKで基本的なエージェントを実装してAPIの感覚を掴む
>* 専門エージェント（検索、コードレビュー等）から始めて段階的に拡張
>* オーケストレーターで複数エージェントの協調動作を実現
>* メトリクス収集でシステムの健全性を継続的に監視

**📌 プロダクション運用のポイント**

>* **エラーハンドリング**: 各エージェントの障害を想定した堅牢な設計
>* **キャッシュ戦略**: 重複するクエリの最適化でコストとレスポンス時間を改善
>* **レート制限**: APIクォータ管理で予期しない課金を防止
>* **ログとモニタリング**: エージェント間の相互作用を可視化

現在ADKはプレビュー段階ですが、TypeScriptでの基盤実装を今から準備しておけば、正式リリース時にスムーズに移行できますね！まずは簡単なエージェントから作って、あなたのプロジェクトに最適なマルチエージェント構成を見つけてみましょう！

**次のステップ**: 実際にコードを動かして、特定のドメイン（カスタマーサポート、データ分析、コンテンツ生成など）向けの専門エージェントを開発してみてください！