## もうVM管理で迷わない！Google Compute EngineをTypeScriptで完全制御する実践ガイド

## はじめに

「クラウドのVM管理は複雑で、手作業でのサーバー設定は時間がかかりすぎる」

そんな悩みを抱えるインフラエンジニアや開発者の方に朗報です。Google Compute Engine（GCE）なら、**TypeScript**を使ってVM の作成から設定、監視まで全てをプログラマブルに管理できます。

この記事では、**Google Cloud SDK** と **TypeScript** を組み合わせてCompute Engineインスタンスを効率的に管理し、実際の運用で使える実践的なコードパターンを解説します。記事を読み終える頃には、あなたも Infrastructure as Code の真髄を体感できているはずです 🚀

>* Google Compute Engine の基本概念とメリット・デメリットの理解
>* TypeScript + Google Cloud Client Library でのVM管理自動化
>* インスタンステンプレートとマネージドインスタンスグループの活用
>* 監視・ログ管理・コスト最適化の実装パターン

## Google Compute Engineとは？

Google Compute Engine は、Googleが提供する**高性能な仮想マシンサービス**です。

**1. 圧倒的な性能と柔軟性**

>* カスタムマシンタイプで CPU・メモリを1GB単位で調整可能
>* 最新のIntel/AMD CPUと高性能GPU（NVIDIA Tesla、A100）をサポート
>* プリエンプティブルインスタンスで最大80%のコスト削減

**2. Google品質のインフラストラクチャ**

>* 99.99%の稼働率SLA保証
>* 世界中26リージョン・78ゾーンでのグローバル展開
>* Googleの専用光ファイバーネットワークで超高速データ転送

**3. DevOps フレンドリーな運用環境**

>* 自動スケーリングとロードバランシングの標準対応
>* コンテナ最適化OSでDocker/Kubernetesとシームレス連携
>* 豊富なAPIとSDKによるプログラマブルな管理

**⚖️ 他のクラウドVMサービスとの比較**

| サービス | カスタマイズ性 | 料金体系 | エコシステム | 適用場面 |
|---------|---------------|----------|-------------|---------|
| **Compute Engine** | ⭐⭐⭐ | 秒単位課金 | GCP統合 | 高負荷・本格運用 |
| **AWS EC2** | ⭐⭐⭐ | 秒単位課金 | AWS統合 | 豊富な選択肢重視 |
| **Azure VM** | ⭐⭐ | 秒単位課金 | Microsoft統合 | 企業向けハイブリッド |

**📌 Compute Engineを選ぶべきシーン**

>* 機械学習・データ分析で高性能GPUが必要
>* BigQuery・Cloud Storage等のGCPサービスとの連携が多い
>* コスト効率を重視したい（プリエンプティブルインスタンス活用）

## 開発環境のセットアップ

Compute Engine をTypeScriptで管理するための開発環境を構築しましょう。

**📦 必要なツールのインストール**

```bash
# Google Cloud SDK のインストール（公式サイトからダウンロード）
# https://cloud.google.com/sdk/docs/install

# Cloud SDK の初期化とログイン
gcloud init
gcloud auth login
gcloud auth application-default login

# プロジェクトの作成と設定
gcloud projects create your-gce-project --name="My GCE Project"
gcloud config set project your-gce-project

# Compute Engine API の有効化
gcloud services enable compute.googleapis.com
```

**🚀 TypeScriptプロジェクトの初期化**

```bash
# プロジェクトディレクトリの作成
mkdir gce-typescript-manager && cd gce-typescript-manager

# Node.jsプロジェクトの初期化
npm init -y

# Google Cloud Client Library のインストール
npm install @google-cloud/compute
npm install -D typescript @types/node ts-node nodemon

# 設定管理・ユーティリティライブラリ
npm install dotenv winston
npm install -D @types/dotenv
```

**📝 TypeScript設定ファイルの作成**

```json:tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020"],
    "module": "commonjs",
    "moduleResolution": "node",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "sourceMap": true,
    "removeComments": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## Compute Engine インスタンス管理の実装

実用的なVM管理システムをTypeScriptで構築してみましょう。

### 1\. 基本的なインスタンス操作クラス

```typescript:src/services/ComputeEngineManager.ts
import { InstancesClient, ZoneOperationsClient } from '@google-cloud/compute';
import { logger } from '../utils/logger';

export interface InstanceConfig {
  name: string;
  zone: string;
  machineType: string;
  diskImage: string;
  diskSize?: number;
  tags?: string[];
  labels?: Record<string, string>;
  metadata?: Record<string, string>;
}

export interface InstanceInfo {
  id: string;
  name: string;
  status: string;
  machineType: string;
  zone: string;
  internalIP?: string;
  externalIP?: string;
  createdAt: string;
}

export class ComputeEngineManager {
  private instancesClient: InstancesClient;
  private operationsClient: ZoneOperationsClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.instancesClient = new InstancesClient();
    this.operationsClient = new ZoneOperationsClient();
  }

  async createInstance(config: InstanceConfig): Promise<InstanceInfo> {
    const {
      name,
      zone,
      machineType,
      diskImage,
      diskSize = 10,
      tags = [],
      labels = {},
      metadata = {}
    } = config;

    const instanceResource = {
      name,
      machineType: `zones/${zone}/machineTypes/${machineType}`,
      disks: [
        {
          boot: true,
          autoDelete: true,
          initializeParams: {
            sourceImage: `projects/ubuntu-os-cloud/global/images/${diskImage}`,
            diskSizeGb: diskSize.toString(),
          },
        },
      ],
      networkInterfaces: [
        {
          network: 'global/networks/default',
          accessConfigs: [
            {
              type: 'ONE_TO_ONE_NAT',
              name: 'External NAT',
            },
          ],
        },
      ],
      tags: {
        items: tags,
      },
      labels,
      metadata: {
        items: Object.entries(metadata).map(([key, value]) => ({
          key,
          value,
        })),
      },
    };

    try {
      logger.info(`Creating instance: ${name} in zone: ${zone}`);
      
      const [operation] = await this.instancesClient.insert({
        project: this.projectId,
        zone,
        instanceResource,
      });

      // オペレーション完了を待つ
      await this.waitForOperation(operation.name!, zone);
      
      // 作成されたインスタンスの詳細を取得
      const instanceInfo = await this.getInstance(name, zone);
      
      logger.info(`Successfully created instance: ${name}`);
      return instanceInfo;

    } catch (error) {
      logger.error(`Failed to create instance ${name}:`, error);
      throw new Error(`Instance creation failed: ${error.message}`);
    }
  }

  async getInstance(name: string, zone: string): Promise<InstanceInfo> {
    try {
      const [instance] = await this.instancesClient.get({
        project: this.projectId,
        zone,
        instance: name,
      });

      const networkInterface = instance.networkInterfaces?.[0];
      const accessConfig = networkInterface?.accessConfigs?.[0];

      return {
        id: instance.id!.toString(),
        name: instance.name!,
        status: instance.status!,
        machineType: instance.machineType!.split('/').pop()!,
        zone,
        internalIP: networkInterface?.networkIP,
        externalIP: accessConfig?.natIP,
        createdAt: instance.creationTimestamp!,
      };

    } catch (error) {
      logger.error(`Failed to get instance ${name}:`, error);
      throw new Error(`Instance retrieval failed: ${error.message}`);
    }
  }

  async listInstances(zone: string): Promise<InstanceInfo[]> {
    try {
      const [instances] = await this.instancesClient.list({
        project: this.projectId,
        zone,
      });

      return instances.map(instance => {
        const networkInterface = instance.networkInterfaces?.[0];
        const accessConfig = networkInterface?.accessConfigs?.[0];

        return {
          id: instance.id!.toString(),
          name: instance.name!,
          status: instance.status!,
          machineType: instance.machineType!.split('/').pop()!,
          zone,
          internalIP: networkInterface?.networkIP,
          externalIP: accessConfig?.natIP,
          createdAt: instance.creationTimestamp!,
        };
      });

    } catch (error) {
      logger.error(`Failed to list instances in zone ${zone}:`, error);
      throw new Error(`Instance listing failed: ${error.message}`);
    }
  }

  async deleteInstance(name: string, zone: string): Promise<void> {
    try {
      logger.info(`Deleting instance: ${name} in zone: ${zone}`);
      
      const [operation] = await this.instancesClient.delete({
        project: this.projectId,
        zone,
        instance: name,
      });

      await this.waitForOperation(operation.name!, zone);
      
      logger.info(`Successfully deleted instance: ${name}`);

    } catch (error) {
      logger.error(`Failed to delete instance ${name}:`, error);
      throw new Error(`Instance deletion failed: ${error.message}`);
    }
  }

  async startInstance(name: string, zone: string): Promise<void> {
    try {
      logger.info(`Starting instance: ${name}`);
      
      const [operation] = await this.instancesClient.start({
        project: this.projectId,
        zone,
        instance: name,
      });

      await this.waitForOperation(operation.name!, zone);
      
      logger.info(`Successfully started instance: ${name}`);

    } catch (error) {
      logger.error(`Failed to start instance ${name}:`, error);
      throw new Error(`Instance start failed: ${error.message}`);
    }
  }

  async stopInstance(name: string, zone: string): Promise<void> {
    try {
      logger.info(`Stopping instance: ${name}`);
      
      const [operation] = await this.instancesClient.stop({
        project: this.projectId,
        zone,
        instance: name,
      });

      await this.waitForOperation(operation.name!, zone);
      
      logger.info(`Successfully stopped instance: ${name}`);

    } catch (error) {
      logger.error(`Failed to stop instance ${name}:`, error);
      throw new Error(`Instance stop failed: ${error.message}`);
    }
  }

  private async waitForOperation(operationName: string, zone: string): Promise<void> {
    let operation;
    do {
      [operation] = await this.operationsClient.get({
        project: this.projectId,
        zone,
        operation: operationName,
      });

      if (operation.status === 'DONE') {
        if (operation.error) {
          throw new Error(`Operation failed: ${JSON.stringify(operation.error)}`);
        }
        break;
      }

      // 2秒間待ってから再チェック
      await new Promise(resolve => setTimeout(resolve, 2000));
    } while (operation.status !== 'DONE');
  }
}
```

### 2\. インスタンステンプレートの管理

大規模な運用では、インスタンステンプレートを使った標準化が重要です。

```typescript:src/services/InstanceTemplateManager.ts
import { InstanceTemplatesClient, GlobalOperationsClient } from '@google-cloud/compute';
import { logger } from '../utils/logger';

export interface TemplateConfig {
  name: string;
  description?: string;
  machineType: string;
  diskImage: string;
  diskSize?: number;
  networkTags?: string[];
  labels?: Record<string, string>;
  startupScript?: string;
  serviceAccount?: string;
  scopes?: string[];
}

export class InstanceTemplateManager {
  private templatesClient: InstanceTemplatesClient;
  private operationsClient: GlobalOperationsClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.templatesClient = new InstanceTemplatesClient();
    this.operationsClient = new GlobalOperationsClient();
  }

  async createTemplate(config: TemplateConfig): Promise<void> {
    const {
      name,
      description = '',
      machineType,
      diskImage,
      diskSize = 10,
      networkTags = [],
      labels = {},
      startupScript = '',
      serviceAccount,
      scopes = ['https://www.googleapis.com/auth/cloud-platform']
    } = config;

    const templateResource = {
      name,
      description,
      properties: {
        machineType,
        disks: [
          {
            boot: true,
            autoDelete: true,
            initializeParams: {
              sourceImage: `projects/ubuntu-os-cloud/global/images/${diskImage}`,
              diskSizeGb: diskSize.toString(),
            },
          },
        ],
        networkInterfaces: [
          {
            network: 'global/networks/default',
            accessConfigs: [
              {
                type: 'ONE_TO_ONE_NAT',
                name: 'External NAT',
              },
            ],
          },
        ],
        tags: {
          items: networkTags,
        },
        labels,
        metadata: {
          items: startupScript ? [
            {
              key: 'startup-script',
              value: startupScript,
            },
          ] : [],
        },
        serviceAccounts: serviceAccount ? [
          {
            email: serviceAccount,
            scopes,
          },
        ] : [],
      },
    };

    try {
      logger.info(`Creating instance template: ${name}`);
      
      const [operation] = await this.templatesClient.insert({
        project: this.projectId,
        instanceTemplateResource: templateResource,
      });

      await this.waitForGlobalOperation(operation.name!);
      
      logger.info(`Successfully created instance template: ${name}`);

    } catch (error) {
      logger.error(`Failed to create instance template ${name}:`, error);
      throw new Error(`Template creation failed: ${error.message}`);
    }
  }

  async deleteTemplate(name: string): Promise<void> {
    try {
      logger.info(`Deleting instance template: ${name}`);
      
      const [operation] = await this.templatesClient.delete({
        project: this.projectId,
        instanceTemplate: name,
      });

      await this.waitForGlobalOperation(operation.name!);
      
      logger.info(`Successfully deleted instance template: ${name}`);

    } catch (error) {
      logger.error(`Failed to delete instance template ${name}:`, error);
      throw new Error(`Template deletion failed: ${error.message}`);
    }
  }

  async listTemplates(): Promise<any[]> {
    try {
      const [templates] = await this.templatesClient.list({
        project: this.projectId,
      });

      return templates.map(template => ({
        name: template.name,
        description: template.description,
        createdAt: template.creationTimestamp,
        machineType: template.properties?.machineType,
      }));

    } catch (error) {
      logger.error('Failed to list instance templates:', error);
      throw new Error(`Template listing failed: ${error.message}`);
    }
  }

  private async waitForGlobalOperation(operationName: string): Promise<void> {
    let operation;
    do {
      [operation] = await this.operationsClient.get({
        project: this.projectId,
        operation: operationName,
      });

      if (operation.status === 'DONE') {
        if (operation.error) {
          throw new Error(`Operation failed: ${JSON.stringify(operation.error)}`);
        }
        break;
      }

      await new Promise(resolve => setTimeout(resolve, 2000));
    } while (operation.status !== 'DONE');
  }
}
```

### 3\. 実践的な使用例

```typescript:src/examples/basicUsage.ts
import { ComputeEngineManager, InstanceTemplateManager } from '../services';
import { logger } from '../utils/logger';

async function createWebServerInstance() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const gceManager = new ComputeEngineManager(projectId);

  const webServerConfig = {
    name: 'web-server-01',
    zone: 'asia-northeast1-a',
    machineType: 'e2-medium',
    diskImage: 'ubuntu-2204-lts',
    diskSize: 20,
    tags: ['http-server', 'https-server'],
    labels: {
      environment: 'production',
      application: 'web-server',
      team: 'backend'
    },
    metadata: {
      'startup-script': `#!/bin/bash
      apt-get update
      apt-get install -y nginx
      systemctl start nginx
      systemctl enable nginx
      `
    }
  };

  try {
    const instance = await gceManager.createInstance(webServerConfig);
    logger.info('Web server instance created:', instance);

    // インスタンス一覧の確認
    const instances = await gceManager.listInstances('asia-northeast1-a');
    logger.info('Current instances:', instances);

  } catch (error) {
    logger.error('Failed to create web server instance:', error);
  }
}

async function createInstanceTemplate() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const templateManager = new InstanceTemplateManager(projectId);

  const templateConfig = {
    name: 'web-server-template',
    description: 'Standard web server template with Nginx',
    machineType: 'e2-medium',
    diskImage: 'ubuntu-2204-lts',
    diskSize: 20,
    networkTags: ['http-server', 'https-server'],
    labels: {
      environment: 'production',
      type: 'web-server'
    },
    startupScript: `#!/bin/bash
    apt-get update
    apt-get install -y nginx nodejs npm
    systemctl start nginx
    systemctl enable nginx
    
    # アプリケーションのセットアップ
    cd /opt
    git clone https://github.com/your-org/your-app.git
    cd your-app
    npm install
    npm run build
    pm2 start ecosystem.config.js
    `
  };

  try {
    await templateManager.createTemplate(templateConfig);
    logger.info('Instance template created successfully');

    // テンプレート一覧の確認
    const templates = await templateManager.listTemplates();
    logger.info('Current templates:', templates);

  } catch (error) {
    logger.error('Failed to create instance template:', error);
  }
}

// 実行
createWebServerInstance();
createInstanceTemplate();
```

## 自動スケーリングとロードバランシング

**🔧 マネージドインスタンスグループの実装**

```typescript:src/services/ManagedInstanceGroupManager.ts
import { 
  InstanceGroupManagersClient, 
  RegionOperationsClient,
  RegionAutoscalersClient 
} from '@google-cloud/compute';
import { logger } from '../utils/logger';

export interface MIGConfig {
  name: string;
  region: string;
  instanceTemplate: string;
  targetSize: number;
  baseInstanceName: string;
  zones: string[];
}

export interface AutoscalingConfig {
  minReplicas: number;
  maxReplicas: number;
  cpuTargetUtilization: number;
  coolDownPeriod?: number;
}

export class ManagedInstanceGroupManager {
  private migClient: InstanceGroupManagersClient;
  private operationsClient: RegionOperationsClient;
  private autoscalerClient: RegionAutoscalersClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.migClient = new InstanceGroupManagersClient();
    this.operationsClient = new RegionOperationsClient();
    this.autoscalerClient = new RegionAutoscalersClient();
  }

  async createManagedInstanceGroup(config: MIGConfig): Promise<void> {
    const {
      name,
      region,
      instanceTemplate,
      targetSize,
      baseInstanceName,
      zones
    } = config;

    const migResource = {
      name,
      instanceTemplate: `projects/${this.projectId}/global/instanceTemplates/${instanceTemplate}`,
      targetSize,
      baseInstanceName,
      distributionPolicy: {
        zones: zones.map(zone => ({
          zone: `projects/${this.projectId}/zones/${zone}`,
        })),
      },
    };

    try {
      logger.info(`Creating managed instance group: ${name} in region: ${region}`);
      
      const [operation] = await this.migClient.insert({
        project: this.projectId,
        region,
        instanceGroupManagerResource: migResource,
      });

      await this.waitForRegionOperation(operation.name!, region);
      
      logger.info(`Successfully created managed instance group: ${name}`);

    } catch (error) {
      logger.error(`Failed to create MIG ${name}:`, error);
      throw new Error(`MIG creation failed: ${error.message}`);
    }
  }

  async setupAutoscaling(migName: string, region: string, config: AutoscalingConfig): Promise<void> {
    const {
      minReplicas,
      maxReplicas,
      cpuTargetUtilization,
      coolDownPeriod = 60
    } = config;

    const autoscalerResource = {
      name: `${migName}-autoscaler`,
      target: `projects/${this.projectId}/regions/${region}/instanceGroupManagers/${migName}`,
      autoscalingPolicy: {
        minNumReplicas: minReplicas,
        maxNumReplicas: maxReplicas,
        coolDownPeriodSec: coolDownPeriod,
        cpuUtilization: {
          utilizationTarget: cpuTargetUtilization / 100,
        },
      },
    };

    try {
      logger.info(`Setting up autoscaling for MIG: ${migName}`);
      
      const [operation] = await this.autoscalerClient.insert({
        project: this.projectId,
        region,
        autoscalerResource,
      });

      await this.waitForRegionOperation(operation.name!, region);
      
      logger.info(`Successfully configured autoscaling for MIG: ${migName}`);

    } catch (error) {
      logger.error(`Failed to setup autoscaling for MIG ${migName}:`, error);
      throw new Error(`Autoscaling setup failed: ${error.message}`);
    }
  }

  async resizeManagedInstanceGroup(migName: string, region: string, size: number): Promise<void> {
    try {
      logger.info(`Resizing MIG ${migName} to ${size} instances`);
      
      const [operation] = await this.migClient.resize({
        project: this.projectId,
        region,
        instanceGroupManager: migName,
        size,
      });

      await this.waitForRegionOperation(operation.name!, region);
      
      logger.info(`Successfully resized MIG ${migName} to ${size} instances`);

    } catch (error) {
      logger.error(`Failed to resize MIG ${migName}:`, error);
      throw new Error(`MIG resize failed: ${error.message}`);
    }
  }

  private async waitForRegionOperation(operationName: string, region: string): Promise<void> {
    let operation;
    do {
      [operation] = await this.operationsClient.get({
        project: this.projectId,
        region,
        operation: operationName,
      });

      if (operation.status === 'DONE') {
        if (operation.error) {
          throw new Error(`Operation failed: ${JSON.stringify(operation.error)}`);
        }
        break;
      }

      await new Promise(resolve => setTimeout(resolve, 2000));
    } while (operation.status !== 'DONE');
  }
}
```

## 4. 監視とログ管理

**📊 Cloud Monitoring との連携**

```typescript:src/services/MonitoringService.ts
import { MetricServiceClient } from '@google-cloud/monitoring';
import { logger } from '../utils/logger';

export interface MetricData {
  instanceName: string;
  zone: string;
  metricType: string;
  value: number;
  timestamp?: Date;
}

export class MonitoringService {
  private client: MetricServiceClient;
  private projectId: string;
  private projectPath: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.client = new MetricServiceClient();
    this.projectPath = this.client.projectPath(projectId);
  }

  async getInstanceMetrics(instanceName: string, zone: string, hours = 1): Promise<any[]> {
    const filter = `resource.type="gce_instance" AND resource.labels.instance_name="${instanceName}" AND resource.labels.zone="${zone}"`;
    
    const now = new Date();
    const startTime = new Date(now.getTime() - hours * 60 * 60 * 1000);

    const request = {
      name: this.projectPath,
      filter,
      interval: {
        endTime: { seconds: Math.floor(now.getTime() / 1000) },
        startTime: { seconds: Math.floor(startTime.getTime() / 1000) },
      },
      view: 'FULL',
    };

    try {
      const [timeSeries] = await this.client.listTimeSeries(request);
      
      return timeSeries.map(series => ({
        metricType: series.metric?.type,
        resource: series.resource?.labels,
        points: series.points?.map(point => ({
          value: point.value?.doubleValue || point.value?.int64Value,
          timestamp: point.interval?.endTime,
        })),
      }));

    } catch (error) {
      logger.error(`Failed to get metrics for instance ${instanceName}:`, error);
      throw new Error(`Metrics retrieval failed: ${error.message}`);
    }
  }

  async createCustomMetric(metricData: MetricData): Promise<void> {
    const { instanceName, zone, metricType, value, timestamp = new Date() } = metricData;

    const dataPoint = {
      interval: {
        endTime: {
          seconds: Math.floor(timestamp.getTime() / 1000),
        },
      },
      value: {
        doubleValue: value,
      },
    };

    const timeSeries = {
      metric: {
        type: `custom.googleapis.com/${metricType}`,
        labels: {
          instance_name: instanceName,
        },
      },
      resource: {
        type: 'gce_instance',
        labels: {
          instance_id: instanceName,
          zone: zone,
          project_id: this.projectId,
        },
      },
      points: [dataPoint],
    };

    const request = {
      name: this.projectPath,
      timeSeries: [timeSeries],
    };

    try {
      await this.client.createTimeSeries(request);
      logger.info(`Custom metric created: ${metricType} for instance ${instanceName}`);

    } catch (error) {
      logger.error(`Failed to create custom metric:`, error);
      throw new Error(`Custom metric creation failed: ${error.message}`);
    }
  }
}
```

**💰 コスト管理とリソース最適化**

```typescript:src/services/CostOptimizationService.ts
import { ComputeEngineManager } from './ComputeEngineManager';
import { logger } from '../utils/logger';

export interface OptimizationRule {
  name: string;
  condition: (instance: any) => boolean;
  action: 'stop' | 'delete' | 'resize' | 'warn';
  description: string;
}

export class CostOptimizationService {
  private gceManager: ComputeEngineManager;

  constructor(projectId: string) {
    this.gceManager = new ComputeEngineManager(projectId);
  }

  // 使用率の低いインスタンスを特定する最適化ルール
  getDefaultOptimizationRules(): OptimizationRule[] {
    return [
      {
        name: 'idle-instances',
        condition: (instance) => {
          // インスタンスが7日間以上停止している場合
          const lastStopTime = new Date(instance.lastStopTimestamp || 0);
          const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
          return instance.status === 'TERMINATED' && lastStopTime < sevenDaysAgo;
        },
        action: 'delete',
        description: '7日間以上停止しているインスタンスを削除対象とする'
      },
      {
        name: 'oversized-instances',
        condition: (instance) => {
          // 大きなマシンタイプで低負荷のインスタンス
          const largeTypes = ['n2-standard-4', 'n2-standard-8', 'n2-standard-16'];
          return largeTypes.includes(instance.machineType) && 
                 instance.cpuUtilization < 0.2; // CPU使用率20%未満
        },
        action: 'resize',
        description: '大型インスタンスで低負荷の場合はリサイズを提案'
      },
      {
        name: 'non-preemptible-dev',
        condition: (instance) => {
          // 開発環境で通常インスタンスを使用している場合
          const isDevEnv = instance.labels?.environment === 'development' || 
                          instance.labels?.env === 'dev';
          return isDevEnv && !instance.scheduling?.preemptible;
        },
        action: 'warn',
        description: '開発環境でプリエンプティブルインスタンスの使用を推奨'
      }
    ];
  }

  async analyzeAndOptimize(zones: string[]): Promise<void> {
    const rules = this.getDefaultOptimizationRules();
    
    for (const zone of zones) {
      try {
        const instances = await this.gceManager.listInstances(zone);
        
        logger.info(`Analyzing ${instances.length} instances in zone: ${zone}`);

        for (const instance of instances) {
          for (const rule of rules) {
            if (rule.condition(instance)) {
              await this.executeOptimizationAction(instance, rule, zone);
            }
          }
        }

      } catch (error) {
        logger.error(`Failed to analyze instances in zone ${zone}:`, error);
      }
    }
  }

  private async executeOptimizationAction(
    instance: any, 
    rule: OptimizationRule, 
    zone: string
  ): Promise<void> {
    const { name, action, description } = rule;
    const instanceName = instance.name;

    logger.info(`Applying rule '${name}' to instance '${instanceName}': ${description}`);

    try {
      switch (action) {
        case 'stop':
          if (instance.status === 'RUNNING') {
            await this.gceManager.stopInstance(instanceName, zone);
            logger.info(`Stopped instance: ${instanceName}`);
          }
          break;

        case 'delete':
          // 本番環境では慎重に実装する
          logger.warn(`DELETE action suggested for instance: ${instanceName} (not executed)`);
          break;

        case 'resize':
          logger.warn(`RESIZE action suggested for instance: ${instanceName} (manual review required)`);
          break;

        case 'warn':
          logger.warn(`Optimization suggestion for ${instanceName}: ${description}`);
          break;
      }

    } catch (error) {
      logger.error(`Failed to execute optimization action for instance ${instanceName}:`, error);
    }
  }

  async generateCostReport(zones: string[]): Promise<any> {
    const report = {
      totalInstances: 0,
      runningInstances: 0,
      stoppedInstances: 0,
      instancesByMachineType: {} as Record<string, number>,
      recommendedActions: [] as any[],
      estimatedMonthlySavings: 0
    };

    for (const zone of zones) {
      try {
        const instances = await this.gceManager.listInstances(zone);
        report.totalInstances += instances.length;

        for (const instance of instances) {
          if (instance.status === 'RUNNING') {
            report.runningInstances++;
          } else {
            report.stoppedInstances++;
          }

          // マシンタイプ別の統計
          const machineType = instance.machineType;
          report.instancesByMachineType[machineType] = 
            (report.instancesByMachineType[machineType] || 0) + 1;
        }

      } catch (error) {
        logger.error(`Failed to generate cost report for zone ${zone}:`, error);
      }
    }

    logger.info('Cost optimization report generated:', report);
    return report;
  }
}
```

## ユーティリティとヘルパー関数

**🛠️ ログ設定**

```typescript:src/utils/logger.ts
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.colorize(),
    winston.format.simple()
  ),
  defaultMeta: { service: 'gce-typescript-manager' },
  transports: [
    new winston.transports.File({ 
      filename: 'logs/error.log', 
      level: 'error' 
    }),
    new winston.transports.File({ 
      filename: 'logs/combined.log' 
    }),
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      )
    })
  ],
});

export { logger };
```

**📋 設定管理**

```typescript:src/config/index.ts
import dotenv from 'dotenv';

dotenv.config();

export const config = {
  projectId: process.env.GOOGLE_CLOUD_PROJECT || '',
  defaultZone: process.env.DEFAULT_ZONE || 'asia-northeast1-a',
  defaultRegion: process.env.DEFAULT_REGION || 'asia-northeast1',
  logLevel: process.env.LOG_LEVEL || 'info',
  
  // デフォルトのマシンタイプ設定
  defaultMachineTypes: {
    small: 'e2-micro',
    medium: 'e2-medium',
    large: 'e2-standard-4',
  },

  // デフォルトのディスクイメージ
  defaultImages: {
    ubuntu: 'ubuntu-2204-lts',
    debian: 'debian-11',
    centos: 'centos-stream-9',
  },

  // コスト最適化設定
  optimization: {
    checkIntervalHours: 24,
    enableAutoStop: false,
    enableAutoDelete: false,
  }
};

// 必須設定の検証
if (!config.projectId) {
  throw new Error('GOOGLE_CLOUD_PROJECT environment variable is required');
}
```

## まとめ

Google Compute Engine × TypeScript の組み合わせで、VM管理の複雑さから解放された効率的なインフラ管理が実現できました。

**✅ この記事で身についたスキル**

>* Compute Engine の基本操作とインスタンスライフサイクル管理
>* TypeScript + Google Cloud Client Library での実用的なVM自動化
>* インスタンステンプレートとマネージドインスタンスグループの活用
>* 監視・ログ管理・コスト最適化の実装パターン
>* Infrastructure as Code によるスケーラブルな運用基盤構築

**🎯 次のステップ**

>* Kubernetes Engine（GKE）との連携でコンテナオーケストレーション
>* Terraform や Pulumi を使った宣言的なインフラ管理
>* CI/CD パイプラインとの統合で完全自動化されたデプロイメント

Compute Engine なら、従来の物理サーバー運用の煩わしさを一切感じることなく、必要な時に必要なだけのコンピューティングリソースを活用できます。あなたも今日からプログラマブルなインフラ管理を始めてみませんか？