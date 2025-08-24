## もう性能で迷わない！GCP Cloud TPU・GPUをTypeScriptで完全制御する機械学習基盤構築ガイド

## はじめに

「機械学習モデルの学習が遅すぎて開発が進まない」「TPUとGPUの使い分けがよくわからない」

そんな悩みを抱える機械学習エンジニアの方に朗報です。Google Cloud Platform の **Cloud TPU** と **GPU インスタンス** なら、TypeScript を使って高性能な機械学習基盤を効率的に構築・管理できます。

この記事では、**Google Cloud TPU・GPU** の基本概念から、**TypeScript による実装例** まで、実際の運用で使える実践的なコードパターンを解説します。記事を読み終える頃には、あなたも高性能なAI基盤をプログラマブルに管理できるようになっているはずです 🚀

>* Cloud TPU と GPU の特徴・使い分けの理解
>* TypeScript + Google Cloud Client Library での TPU・GPU 管理自動化
>* 機械学習ワークフロー最適化の実装パターン
>* コスト効率を考慮したリソース管理戦略

## Cloud TPU と GPUとは

Google Cloud Platform では、機械学習ワークロードに応じて最適化された2つの選択肢を提供しています。

### 1\. Cloud TPUの特徴

**Cloud TPU（Tensor Processing Unit）** は、Googleが機械学習専用に設計したカスタムASICです。

**💡 TPUが最適な用途**

>* 大規模言語モデル（LLM）の学習・推論
>* 大量の行列計算を含む深層学習モデル
>* 128x128のチャンクに分割できる密度の高い計算処理
>* TensorFlowやPyTorchでの標準的な演算

**📊 最新のTPU性能（2025年版）**

| TPUバージョン | 発表年 | 性能向上 | 特徴 |
|-------------|-------|----------|------|
| **TPU v6** | 2024年5月 | v5eの4.7倍 | 256x256 MXU搭載 |
| **TPU v7 (Ironwood)** | 2025年4月 | 大幅性能向上 | 256チップ・9,216チップ構成 |
| **TPU v6e** | 2024年 | 高コスパ | 256 x 256 multiply-accumulators |

### 2\. GPU インスタンスの特徴

**GPU インスタンス** は、NVIDIA製の汎用GPU を搭載した仮想マシンです。

**💡 GPUが最適な用途**

>* 迅速なプロトタイピングと最大限の柔軟性が必要な場合
>* カスタム PyTorch/JAX オペレーションを使用するモデル
>* Cloud TPU で利用できない TensorFlow 演算子を使用するモデル
>* 並列処理が可能な多様な計算ワークロード

**📊 最新のGPU選択肢（2025年版）**

| マシンタイプ | GPU | メモリ | 適用場面 |
|-------------|-----|--------|---------|
| **A4X** | NVIDIA GB200 Grace Blackwell | 超大容量 | 基盤モデルの学習・推論 |
| **A3 Ultra** | NVIDIA H200 SXM | 141GB HBM3e | 最高性能ML処理 |
| **A3** | NVIDIA H100 SXM | 80GB HBM3 | 大規模モデル学習 |
| **G2** | NVIDIA L4 | 24GB GDDR6 | 推論・軽量学習 |

**⚖️ TPU vs GPU：使い分けの判断基準**

| 比較項目 | Cloud TPU | GPU インスタンス |
|---------|-----------|----------------|
| **計算特性** | 行列計算特化 | 汎用並列処理 |
| **開発柔軟性** | 制約あり | 高い柔軟性 |
| **コスト効率** | 大規模処理で優秀 | 用途により変動 |
| **セットアップ** | シンプル | 複雑な設定が必要 |
| **フレームワーク** | TensorFlow/PyTorch | 多数サポート |

## 開発環境のセットアップ

Cloud TPU と GPU をTypeScriptで管理するための開発環境を構築しましょう。

**📦 必要なツールのインストール**

```bash
# Google Cloud SDK のインストール
# https://cloud.google.com/sdk/docs/install からダウンロード

# Cloud SDK の初期化とログイン
gcloud init
gcloud auth login
gcloud auth application-default login

# プロジェクトの作成と設定
gcloud projects create your-ml-project --name="ML Platform Project"
gcloud config set project your-ml-project

# 必要なAPIの有効化
gcloud services enable compute.googleapis.com
gcloud services enable tpu.googleapis.com
gcloud services enable container.googleapis.com
```

**🚀 TypeScriptプロジェクトの初期化**

```bash
# プロジェクトディレクトリの作成
mkdir gcp-ml-platform && cd gcp-ml-platform

# Node.jsプロジェクトの初期化
npm init -y

# Google Cloud Client Library のインストール
npm install @google-cloud/tpu @google-cloud/compute
npm install -D typescript @types/node ts-node nodemon

# ユーティリティライブラリ
npm install dotenv winston googleapis
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

## Cloud TPU管理の実装

### 1\. TPU ノード管理クラス

```typescript:src/services/TPUManager.ts
import { TpuClient } from '@google-cloud/tpu';
import { logger } from '../utils/logger';

export interface TPUConfig {
  name: string;
  zone: string;
  acceleratorType: string;
  runtimeVersion: string;
  networkConfig?: {
    network?: string;
    subnetwork?: string;
  };
  labels?: Record<string, string>;
}

export interface TPUInfo {
  name: string;
  state: string;
  acceleratorType: string;
  runtimeVersion: string;
  networkEndpoints?: Array<{
    ipAddress: string;
    port: number;
  }>;
  createTime: string;
}

export class TPUManager {
  private tpuClient: TpuClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.tpuClient = new TpuClient();
  }

  async createTPUNode(config: TPUConfig): Promise<TPUInfo> {
    const {
      name,
      zone,
      acceleratorType,
      runtimeVersion,
      networkConfig = {},
      labels = {}
    } = config;

    const parent = `projects/${this.projectId}/locations/${zone}`;
    
    const nodeResource = {
      acceleratorType,
      runtimeVersion,
      networkConfig: {
        network: networkConfig.network || 'default',
        subnetwork: networkConfig.subnetwork || 'default',
      },
      labels: {
        ...labels,
        'created-by': 'typescript-ml-platform',
        'environment': process.env.NODE_ENV || 'development'
      },
    };

    try {
      logger.info(`Creating TPU node: ${name} in zone: ${zone}`);
      
      const [operation] = await this.tpuClient.createNode({
        parent,
        nodeId: name,
        node: nodeResource,
      });

      // オペレーション完了を待つ
      const [response] = await operation.promise();
      
      logger.info(`Successfully created TPU node: ${name}`);
      return this.formatTPUInfo(response);

    } catch (error) {
      logger.error(`Failed to create TPU node ${name}:`, error);
      throw new Error(`TPU node creation failed: ${error.message}`);
    }
  }

  async getTPUNode(name: string, zone: string): Promise<TPUInfo> {
    try {
      const nodeName = `projects/${this.projectId}/locations/${zone}/nodes/${name}`;
      
      const [node] = await this.tpuClient.getNode({
        name: nodeName,
      });

      return this.formatTPUInfo(node);

    } catch (error) {
      logger.error(`Failed to get TPU node ${name}:`, error);
      throw new Error(`TPU node retrieval failed: ${error.message}`);
    }
  }

  async listTPUNodes(zone: string): Promise<TPUInfo[]> {
    try {
      const parent = `projects/${this.projectId}/locations/${zone}`;
      
      const [nodes] = await this.tpuClient.listNodes({
        parent,
      });

      return nodes.map(node => this.formatTPUInfo(node));

    } catch (error) {
      logger.error(`Failed to list TPU nodes in zone ${zone}:`, error);
      throw new Error(`TPU node listing failed: ${error.message}`);
    }
  }

  async deleteTPUNode(name: string, zone: string): Promise<void> {
    try {
      logger.info(`Deleting TPU node: ${name} in zone: ${zone}`);
      
      const nodeName = `projects/${this.projectId}/locations/${zone}/nodes/${name}`;
      
      const [operation] = await this.tpuClient.deleteNode({
        name: nodeName,
      });

      await operation.promise();
      
      logger.info(`Successfully deleted TPU node: ${name}`);

    } catch (error) {
      logger.error(`Failed to delete TPU node ${name}:`, error);
      throw new Error(`TPU node deletion failed: ${error.message}`);
    }
  }

  async startTPUNode(name: string, zone: string): Promise<void> {
    try {
      logger.info(`Starting TPU node: ${name}`);
      
      const nodeName = `projects/${this.projectId}/locations/${zone}/nodes/${name}`;
      
      const [operation] = await this.tpuClient.startNode({
        name: nodeName,
      });

      await operation.promise();
      
      logger.info(`Successfully started TPU node: ${name}`);

    } catch (error) {
      logger.error(`Failed to start TPU node ${name}:`, error);
      throw new Error(`TPU node start failed: ${error.message}`);
    }
  }

  async stopTPUNode(name: string, zone: string): Promise<void> {
    try {
      logger.info(`Stopping TPU node: ${name}`);
      
      const nodeName = `projects/${this.projectId}/locations/${zone}/nodes/${name}`;
      
      const [operation] = await this.tpuClient.stopNode({
        name: nodeName,
      });

      await operation.promise();
      
      logger.info(`Successfully stopped TPU node: ${name}`);

    } catch (error) {
      logger.error(`Failed to stop TPU node ${name}:`, error);
      throw new Error(`TPU node stop failed: ${error.message}`);
    }
  }

  async getAvailableAcceleratorTypes(zone: string): Promise<string[]> {
    try {
      const parent = `projects/${this.projectId}/locations/${zone}`;
      
      const [acceleratorTypes] = await this.tpuClient.listAcceleratorTypes({
        parent,
      });

      return acceleratorTypes.map(type => type.name!.split('/').pop()!);

    } catch (error) {
      logger.error(`Failed to get accelerator types for zone ${zone}:`, error);
      throw new Error(`Accelerator types retrieval failed: ${error.message}`);
    }
  }

  async getAvailableRuntimeVersions(zone: string): Promise<string[]> {
    try {
      const parent = `projects/${this.projectId}/locations/${zone}`;
      
      const [runtimeVersions] = await this.tpuClient.listRuntimeVersions({
        parent,
      });

      return runtimeVersions.map(version => version.version!);

    } catch (error) {
      logger.error(`Failed to get runtime versions for zone ${zone}:`, error);
      throw new Error(`Runtime versions retrieval failed: ${error.message}`);
    }
  }

  private formatTPUInfo(node: any): TPUInfo {
    return {
      name: node.name!.split('/').pop()!,
      state: node.state!,
      acceleratorType: node.acceleratorType!,
      runtimeVersion: node.runtimeVersion!,
      networkEndpoints: node.networkEndpoints?.map((endpoint: any) => ({
        ipAddress: endpoint.ipAddress,
        port: endpoint.port || 8470,
      })),
      createTime: node.createTime!,
    };
  }
}
```

### 2\. GPU インスタンス管理の実装

```typescript:src/services/GPUManager.ts
import { InstancesClient, ZoneOperationsClient } from '@google-cloud/compute';
import { logger } from '../utils/logger';

export interface GPUConfig {
  name: string;
  zone: string;
  machineType: string;
  gpuType: string;
  gpuCount: number;
  diskImage: string;
  diskSize?: number;
  tags?: string[];
  labels?: Record<string, string>;
  metadata?: Record<string, string>;
  preemptible?: boolean;
}

export interface GPUInstanceInfo {
  name: string;
  status: string;
  machineType: string;
  gpuInfo: {
    type: string;
    count: number;
  };
  zone: string;
  internalIP?: string;
  externalIP?: string;
  preemptible: boolean;
  createdAt: string;
}

export class GPUManager {
  private instancesClient: InstancesClient;
  private operationsClient: ZoneOperationsClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.instancesClient = new InstancesClient();
    this.operationsClient = new ZoneOperationsClient();
  }

  async createGPUInstance(config: GPUConfig): Promise<GPUInstanceInfo> {
    const {
      name,
      zone,
      machineType,
      gpuType,
      gpuCount,
      diskImage,
      diskSize = 50,
      tags = [],
      labels = {},
      metadata = {},
      preemptible = false
    } = config;

    const instanceResource = {
      name,
      machineType: `zones/${zone}/machineTypes/${machineType}`,
      disks: [
        {
          boot: true,
          autoDelete: true,
          initializeParams: {
            sourceImage: `projects/ml-images/global/images/${diskImage}`,
            diskSizeGb: diskSize.toString(),
            diskType: `zones/${zone}/diskTypes/pd-ssd`,
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
      guestAccelerators: [
        {
          acceleratorType: `zones/${zone}/acceleratorTypes/${gpuType}`,
          acceleratorCount: gpuCount,
        },
      ],
      scheduling: {
        onHostMaintenance: 'TERMINATE',
        preemptible,
      },
      tags: {
        items: ['gpu-instance', 'ml-workload', ...tags],
      },
      labels: {
        ...labels,
        'instance-type': 'gpu',
        'created-by': 'typescript-ml-platform',
      },
      metadata: {
        items: [
          {
            key: 'startup-script',
            value: this.getGPUStartupScript(),
          },
          ...Object.entries(metadata).map(([key, value]) => ({
            key,
            value,
          })),
        ],
      },
    };

    try {
      logger.info(`Creating GPU instance: ${name} with ${gpuCount}x ${gpuType}`);
      
      const [operation] = await this.instancesClient.insert({
        project: this.projectId,
        zone,
        instanceResource,
      });

      // オペレーション完了を待つ
      await this.waitForOperation(operation.name!, zone);
      
      // 作成されたインスタンスの詳細を取得
      const instanceInfo = await this.getGPUInstance(name, zone);
      
      logger.info(`Successfully created GPU instance: ${name}`);
      return instanceInfo;

    } catch (error) {
      logger.error(`Failed to create GPU instance ${name}:`, error);
      throw new Error(`GPU instance creation failed: ${error.message}`);
    }
  }

  async getGPUInstance(name: string, zone: string): Promise<GPUInstanceInfo> {
    try {
      const [instance] = await this.instancesClient.get({
        project: this.projectId,
        zone,
        instance: name,
      });

      const networkInterface = instance.networkInterfaces?.[0];
      const accessConfig = networkInterface?.accessConfigs?.[0];
      const gpu = instance.guestAccelerators?.[0];

      return {
        name: instance.name!,
        status: instance.status!,
        machineType: instance.machineType!.split('/').pop()!,
        gpuInfo: {
          type: gpu?.acceleratorType?.split('/').pop()! || 'none',
          count: gpu?.acceleratorCount || 0,
        },
        zone,
        internalIP: networkInterface?.networkIP,
        externalIP: accessConfig?.natIP,
        preemptible: instance.scheduling?.preemptible || false,
        createdAt: instance.creationTimestamp!,
      };

    } catch (error) {
      logger.error(`Failed to get GPU instance ${name}:`, error);
      throw new Error(`GPU instance retrieval failed: ${error.message}`);
    }
  }

  async listGPUInstances(zone: string): Promise<GPUInstanceInfo[]> {
    try {
      const [instances] = await this.instancesClient.list({
        project: this.projectId,
        zone,
        filter: 'labels.instance-type=gpu',
      });

      return instances.map(instance => {
        const networkInterface = instance.networkInterfaces?.[0];
        const accessConfig = networkInterface?.accessConfigs?.[0];
        const gpu = instance.guestAccelerators?.[0];

        return {
          name: instance.name!,
          status: instance.status!,
          machineType: instance.machineType!.split('/').pop()!,
          gpuInfo: {
            type: gpu?.acceleratorType?.split('/').pop()! || 'none',
            count: gpu?.acceleratorCount || 0,
          },
          zone,
          internalIP: networkInterface?.networkIP,
          externalIP: accessConfig?.natIP,
          preemptible: instance.scheduling?.preemptible || false,
          createdAt: instance.creationTimestamp!,
        };
      });

    } catch (error) {
      logger.error(`Failed to list GPU instances in zone ${zone}:`, error);
      throw new Error(`GPU instance listing failed: ${error.message}`);
    }
  }

  async deleteGPUInstance(name: string, zone: string): Promise<void> {
    try {
      logger.info(`Deleting GPU instance: ${name} in zone: ${zone}`);
      
      const [operation] = await this.instancesClient.delete({
        project: this.projectId,
        zone,
        instance: name,
      });

      await this.waitForOperation(operation.name!, zone);
      
      logger.info(`Successfully deleted GPU instance: ${name}`);

    } catch (error) {
      logger.error(`Failed to delete GPU instance ${name}:`, error);
      throw new Error(`GPU instance deletion failed: ${error.message}`);
    }
  }

  private getGPUStartupScript(): string {
    return `#!/bin/bash
    # NVIDIA Driver and CUDA installation
    curl -O https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.0-1_all.deb
    sudo dpkg -i cuda-keyring_1.0-1_all.deb
    sudo apt-get update
    sudo apt-get -y install cuda
    
    # Docker installation for containerized ML workloads
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
    sudo apt-get update
    sudo apt-get -y install docker-ce docker-ce-cli containerd.io
    
    # NVIDIA Container Toolkit
    curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
    curl -s -L https://nvidia.github.io/nvidia-docker/ubuntu20.04/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
    sudo apt-get update && sudo apt-get -y install nvidia-docker2
    sudo systemctl restart docker
    
    # Create ML workspace
    sudo mkdir -p /opt/ml-workspace
    sudo chmod 777 /opt/ml-workspace
    
    echo "GPU instance setup completed" | sudo tee /var/log/startup-script.log
    `;
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

      // 3秒間待ってから再チェック
      await new Promise(resolve => setTimeout(resolve, 3000));
    } while (operation.status !== 'DONE');
  }
}
```

##  機械学習ワークフロー管理

**統合プラットフォームクラス**

```typescript:src/services/MLPlatformManager.ts
import { TPUManager, TPUConfig } from './TPUManager';
import { GPUManager, GPUConfig } from './GPUManager';
import { logger } from '../utils/logger';

export interface MLWorkloadConfig {
  workloadType: 'training' | 'inference' | 'research';
  model: {
    name: string;
    framework: 'tensorflow' | 'pytorch' | 'jax';
    size: 'small' | 'medium' | 'large' | 'xlarge';
  };
  compute: {
    preferredType: 'tpu' | 'gpu' | 'auto';
    zone: string;
    budget?: number;
  };
  storage: {
    datasetSize?: number;
    modelOutputSize?: number;
  };
}

export interface ComputeRecommendation {
  recommendedType: 'tpu' | 'gpu';
  reasoning: string[];
  estimatedCost: number;
  config: TPUConfig | GPUConfig;
}

export class MLPlatformManager {
  private tpuManager: TPUManager;
  private gpuManager: GPUManager;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.tpuManager = new TPUManager(projectId);
    this.gpuManager = new GPUManager(projectId);
  }

  async recommendCompute(workloadConfig: MLWorkloadConfig): Promise<ComputeRecommendation> {
    const { workloadType, model, compute, storage } = workloadConfig;
    const reasoning: string[] = [];
    let recommendedType: 'tpu' | 'gpu';
    let estimatedCost = 0;

    // ワークロードタイプによる初期判定
    if (compute.preferredType !== 'auto') {
      recommendedType = compute.preferredType;
      reasoning.push(`ユーザー指定により${compute.preferredType.toUpperCase()}を選択`);
    } else {
      // 自動判定ロジック
      if (model.framework === 'tensorflow' && model.size === 'xlarge') {
        recommendedType = 'tpu';
        reasoning.push('大規模TensorFlowモデルのためTPUが最適');
      } else if (workloadType === 'research' && model.framework === 'pytorch') {
        recommendedType = 'gpu';
        reasoning.push('研究用途でPyTorchを使用するためGPUが適している');
      } else if (model.size === 'large' || model.size === 'xlarge') {
        recommendedType = 'tpu';
        reasoning.push('大規模モデルのためTPUがコスト効率的');
      } else {
        recommendedType = 'gpu';
        reasoning.push('中小規模モデルでGPUが柔軟性とコストのバランスが良い');
      }
    }

    // 設定とコスト見積もりの生成
    let config: TPUConfig | GPUConfig;
    
    if (recommendedType === 'tpu') {
      const acceleratorType = this.getRecommendedTPUType(model.size);
      config = {
        name: `${model.name}-tpu-${Date.now()}`,
        zone: compute.zone,
        acceleratorType,
        runtimeVersion: model.framework === 'tensorflow' ? 'tpu-vm-tf-2.15.0-v4' : 'tpu-vm-pt-2.1-v4',
        labels: {
          workload: workloadType,
          model: model.name,
          framework: model.framework,
        },
      };
      estimatedCost = this.estimateTPUCost(acceleratorType, workloadType);
    } else {
      const { machineType, gpuType, gpuCount } = this.getRecommendedGPUConfig(model.size);
      config = {
        name: `${model.name}-gpu-${Date.now()}`,
        zone: compute.zone,
        machineType,
        gpuType,
        gpuCount,
        diskImage: 'c0-deeplearning-common-cu121-v20240922-ubuntu-2204',
        diskSize: storage?.datasetSize ? Math.max(50, storage.datasetSize * 2) : 100,
        preemptible: workloadType !== 'inference',
        labels: {
          workload: workloadType,
          model: model.name,
          framework: model.framework,
        },
        metadata: {
          'ml-framework': model.framework,
          'model-size': model.size,
        },
      };
      estimatedCost = this.estimateGPUCost(machineType, gpuType, gpuCount, workloadType);
    }

    // 予算チェック
    if (compute.budget && estimatedCost > compute.budget) {
      reasoning.push(`⚠️ 推定コスト${estimatedCost}円が予算${compute.budget}円を超過`);
      // 予算内の代替案を提案
      if (recommendedType === 'tpu') {
        recommendedType = 'gpu';
        reasoning.push('予算の制約によりGPUインスタンスを提案');
        // GPUの再計算...
      }
    }

    return {
      recommendedType,
      reasoning,
      estimatedCost,
      config,
    };
  }

  async deployWorkload(recommendation: ComputeRecommendation): Promise<string> {
    const { recommendedType, config } = recommendation;

    try {
      if (recommendedType === 'tpu') {
        const tpuInfo = await this.tpuManager.createTPUNode(config as TPUConfig);
        logger.info(`TPU workload deployed: ${tpuInfo.name}`);
        return tpuInfo.name;
      } else {
        const gpuInfo = await this.gpuManager.createGPUInstance(config as GPUConfig);
        logger.info(`GPU workload deployed: ${gpuInfo.name}`);
        return gpuInfo.name;
      }
    } catch (error) {
      logger.error('Failed to deploy ML workload:', error);
      throw new Error(`Workload deployment failed: ${error.message}`);
    }
  }

  async getWorkloadStatus(workloadName: string, zone: string, computeType: 'tpu' | 'gpu') {
    try {
      if (computeType === 'tpu') {
        return await this.tpuManager.getTPUNode(workloadName, zone);
      } else {
        return await this.gpuManager.getGPUInstance(workloadName, zone);
      }
    } catch (error) {
      logger.error(`Failed to get workload status for ${workloadName}:`, error);
      throw new Error(`Status retrieval failed: ${error.message}`);
    }
  }

  async cleanupWorkload(workloadName: string, zone: string, computeType: 'tpu' | 'gpu'): Promise<void> {
    try {
      if (computeType === 'tpu') {
        await this.tpuManager.deleteTPUNode(workloadName, zone);
      } else {
        await this.gpuManager.deleteGPUInstance(workloadName, zone);
      }
      logger.info(`Successfully cleaned up workload: ${workloadName}`);
    } catch (error) {
      logger.error(`Failed to cleanup workload ${workloadName}:`, error);
      throw new Error(`Workload cleanup failed: ${error.message}`);
    }
  }

  private getRecommendedTPUType(modelSize: string): string {
    switch (modelSize) {
      case 'small':
        return 'v2-8';
      case 'medium':
        return 'v3-8';
      case 'large':
        return 'v4-8';
      case 'xlarge':
        return 'v5p-8';
      default:
        return 'v3-8';
    }
  }

  private getRecommendedGPUConfig(modelSize: string): {
    machineType: string;
    gpuType: string;
    gpuCount: number;
  } {
    switch (modelSize) {
      case 'small':
        return {
          machineType: 'g2-standard-4',
          gpuType: 'nvidia-l4',
          gpuCount: 1,
        };
      case 'medium':
        return {
          machineType: 'a2-highgpu-1g',
          gpuType: 'nvidia-tesla-a100',
          gpuCount: 1,
        };
      case 'large':
        return {
          machineType: 'a2-highgpu-2g',
          gpuType: 'nvidia-tesla-a100',
          gpuCount: 2,
        };
      case 'xlarge':
        return {
          machineType: 'a3-highgpu-8g',
          gpuType: 'nvidia-h100-80gb',
          gpuCount: 8,
        };
      default:
        return {
          machineType: 'g2-standard-4',
          gpuType: 'nvidia-l4',
          gpuCount: 1,
        };
    }
  }

  private estimateTPUCost(acceleratorType: string, workloadType: string): number {
    // 簡略化されたコスト計算（実際の料金は公式ドキュメントを参照）
    const baseHourlyCost = {
      'v2-8': 4.5,
      'v3-8': 8.0,
      'v4-8': 16.0,
      'v5p-8': 24.0,
    }[acceleratorType] || 8.0;

    const estimatedHours = workloadType === 'training' ? 24 : 8;
    return Math.round(baseHourlyCost * estimatedHours);
  }

  private estimateGPUCost(machineType: string, gpuType: string, gpuCount: number, workloadType: string): number {
    // 簡略化されたコスト計算（実際の料金は公式ドキュメントを参照）
    const baseHourlyCost = {
      'nvidia-l4': 0.6,
      'nvidia-tesla-a100': 2.9,
      'nvidia-h100-80gb': 8.5,
    }[gpuType] || 2.0;

    const estimatedHours = workloadType === 'training' ? 24 : 8;
    const preemptibleDiscount = workloadType !== 'inference' ? 0.7 : 1.0;
    
    return Math.round(baseHourlyCost * gpuCount * estimatedHours * preemptibleDiscount);
  }
}
```

## 実践的な使用例

```typescript:src/examples/mlWorkflowExample.ts
import { MLPlatformManager, MLWorkloadConfig } from '../services/MLPlatformManager';
import { logger } from '../utils/logger';

async function trainLargeLanguageModel() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const mlPlatform = new MLPlatformManager(projectId);

  const workloadConfig: MLWorkloadConfig = {
    workloadType: 'training',
    model: {
      name: 'llama-7b-custom',
      framework: 'pytorch',
      size: 'large',
    },
    compute: {
      preferredType: 'auto',
      zone: 'us-central1-a',
      budget: 5000, // 月額予算（円）
    },
    storage: {
      datasetSize: 50, // GB
      modelOutputSize: 15, // GB
    },
  };

  try {
    // 1. 最適なコンピューティングリソースの推奨を取得
    const recommendation = await mlPlatform.recommendCompute(workloadConfig);
    
    logger.info('💡 コンピューティング推奨結果:');
    logger.info(`   推奨タイプ: ${recommendation.recommendedType.toUpperCase()}`);
    logger.info(`   推定コスト: ¥${recommendation.estimatedCost}`);
    logger.info(`   理由:`);
    recommendation.reasoning.forEach(reason => {
      logger.info(`     - ${reason}`);
    });

    // 2. ワークロードのデプロイ
    const workloadName = await mlPlatform.deployWorkload(recommendation);
    logger.info(`🚀 ワークロード「${workloadName}」をデプロイしました`);

    // 3. ステータスの監視
    const status = await mlPlatform.getWorkloadStatus(
      workloadName,
      workloadConfig.compute.zone,
      recommendation.recommendedType
    );
    logger.info(`📊 ワークロードステータス:`, status);

    // 4. 学習完了後のクリーンアップ（例：8時間後）
    setTimeout(async () => {
      await mlPlatform.cleanupWorkload(
        workloadName,
        workloadConfig.compute.zone,
        recommendation.recommendedType
      );
      logger.info(`✅ ワークロード「${workloadName}」をクリーンアップしました`);
    }, 8 * 60 * 60 * 1000); // 8時間

  } catch (error) {
    logger.error('ML workload failed:', error);
  }
}

async function runInferenceService() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const mlPlatform = new MLPlatformManager(projectId);

  const inferenceConfig: MLWorkloadConfig = {
    workloadType: 'inference',
    model: {
      name: 'bert-qa-service',
      framework: 'tensorflow',
      size: 'medium',
    },
    compute: {
      preferredType: 'gpu', // 推論サービスはGPUを指定
      zone: 'asia-northeast1-a',
    },
  };

  try {
    const recommendation = await mlPlatform.recommendCompute(inferenceConfig);
    const workloadName = await mlPlatform.deployWorkload(recommendation);
    
    logger.info(`🔍 推論サービス「${workloadName}」が稼働中です`);
    
    // 推論サービスは常駐するためクリーンアップは手動または別の仕組みで

  } catch (error) {
    logger.error('Inference service deployment failed:', error);
  }
}

async function researchExperiment() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const mlPlatform = new MLPlatformManager(projectId);

  const experiments = [
    {
      name: 'vision-transformer-exp1',
      framework: 'pytorch' as const,
      size: 'medium' as const,
    },
    {
      name: 'transformer-ablation-exp2',
      framework: 'pytorch' as const,
      size: 'small' as const,
    },
  ];

  // 複数実験を並列実行
  const deploymentPromises = experiments.map(async (experiment) => {
    const config: MLWorkloadConfig = {
      workloadType: 'research',
      model: experiment,
      compute: {
        preferredType: 'auto',
        zone: 'us-west1-b',
      },
    };

    try {
      const recommendation = await mlPlatform.recommendCompute(config);
      const workloadName = await mlPlatform.deployWorkload(recommendation);
      
      logger.info(`🧪 実験「${workloadName}」を開始しました`);
      return { workloadName, computeType: recommendation.recommendedType };
      
    } catch (error) {
      logger.error(`実験 ${experiment.name} の開始に失敗:`, error);
      return null;
    }
  });

  const deployedExperiments = await Promise.all(deploymentPromises);
  
  logger.info(`📈 ${deployedExperiments.filter(Boolean).length} 個の実験が稼働中です`);
}

// 実行例
trainLargeLanguageModel();
runInferenceService();
researchExperiment();
```

## 監視とコスト最適化

**📊 リソース監視クラス**

```typescript:src/services/ResourceMonitor.ts
import { MLPlatformManager } from './MLPlatformManager';
import { logger } from '../utils/logger';

export interface ResourceUsage {
  resourceName: string;
  computeType: 'tpu' | 'gpu';
  zone: string;
  status: string;
  runningTime: number; // minutes
  estimatedCost: number;
  utilizationScore?: number;
}

export interface CostAlert {
  severity: 'info' | 'warning' | 'critical';
  message: string;
  recommendedAction?: string;
  affectedResources: string[];
}

export class ResourceMonitor {
  private mlPlatform: MLPlatformManager;
  private alertThresholds = {
    dailyBudget: 10000, // 日額予算（円）
    idleTimeMinutes: 60, // アイドル状態の閾値（分）
    lowUtilizationThreshold: 0.3, // 低使用率の閾値
  };

  constructor(projectId: string) {
    this.mlPlatform = new MLPlatformManager(projectId);
  }

  async getAllResourceUsage(zones: string[]): Promise<ResourceUsage[]> {
    const allUsage: ResourceUsage[] = [];

    for (const zone of zones) {
      try {
        // TPUリソースの取得
        const tpuNodes = await this.mlPlatform['tpuManager'].listTPUNodes(zone);
        const tpuUsage = tpuNodes.map(node => ({
          resourceName: node.name,
          computeType: 'tpu' as const,
          zone,
          status: node.state,
          runningTime: this.calculateRunningTime(node.createTime),
          estimatedCost: this.estimateCurrentCost(node.acceleratorType, 'tpu', node.createTime),
        }));

        // GPUリソースの取得
        const gpuInstances = await this.mlPlatform['gpuManager'].listGPUInstances(zone);
        const gpuUsage = gpuInstances.map(instance => ({
          resourceName: instance.name,
          computeType: 'gpu' as const,
          zone,
          status: instance.status,
          runningTime: this.calculateRunningTime(instance.createdAt),
          estimatedCost: this.estimateCurrentCost(instance.gpuInfo.type, 'gpu', instance.createdAt),
        }));

        allUsage.push(...tpuUsage, ...gpuUsage);

      } catch (error) {
        logger.error(`Failed to get resource usage for zone ${zone}:`, error);
      }
    }

    return allUsage;
  }

  async generateCostAlerts(usage: ResourceUsage[]): Promise<CostAlert[]> {
    const alerts: CostAlert[] = [];

    // 総コストの計算
    const totalDailyCost = usage.reduce((sum, resource) => {
      if (resource.status === 'RUNNING' || resource.status === 'READY') {
        return sum + resource.estimatedCost;
      }
      return sum;
    }, 0);

    // 予算超過アラート
    if (totalDailyCost > this.alertThresholds.dailyBudget) {
      alerts.push({
        severity: 'critical',
        message: `日次予算を超過しています: ¥${totalDailyCost} (予算: ¥${this.alertThresholds.dailyBudget})`,
        recommendedAction: '使用率の低いリソースを停止してください',
        affectedResources: usage.map(r => r.resourceName),
      });
    }

    // 長時間実行リソースのアラート
    const longRunningResources = usage.filter(
      resource => resource.runningTime > this.alertThresholds.idleTimeMinutes * 60
    );

    if (longRunningResources.length > 0) {
      alerts.push({
        severity: 'warning',
        message: `${longRunningResources.length}個のリソースが長時間実行中です`,
        recommendedAction: '学習完了したリソースは停止してコストを削減してください',
        affectedResources: longRunningResources.map(r => r.resourceName),
      });
    }

    // アイドル状態のリソース検出
    const idleResources = usage.filter(
      resource => resource.status === 'READY' && 
                  resource.runningTime > this.alertThresholds.idleTimeMinutes
    );

    if (idleResources.length > 0) {
      alerts.push({
        severity: 'warning',
        message: `${idleResources.length}個のリソースがアイドル状態です`,
        recommendedAction: '使用していないリソースを停止してください',
        affectedResources: idleResources.map(r => r.resourceName),
      });
    }

    return alerts;
  }

  async executeAutomatedOptimization(alerts: CostAlert[]): Promise<void> {
    const criticalAlerts = alerts.filter(alert => alert.severity === 'critical');
    
    if (criticalAlerts.length === 0) {
      logger.info('📊 自動最適化が必要なアラートはありません');
      return;
    }

    logger.info('🤖 自動最適化を実行中...');

    for (const alert of criticalAlerts) {
      // 最もコストの高いリソースから順次停止
      const resourceUsage = await this.getAllResourceUsage(['us-central1-a', 'asia-northeast1-a']);
      const sortedByCost = resourceUsage
        .filter(r => alert.affectedResources.includes(r.resourceName))
        .sort((a, b) => b.estimatedCost - a.estimatedCost);

      for (const resource of sortedByCost.slice(0, 2)) { // 上位2つのリソースを停止
        try {
          if (resource.computeType === 'tpu') {
            await this.mlPlatform['tpuManager'].stopTPUNode(resource.resourceName, resource.zone);
          } else {
            // GPUインスタンスの場合は削除（停止機能がないため）
            logger.warn(`GPU instance ${resource.resourceName} needs manual attention`);
          }
          
          logger.info(`⏹️ リソース ${resource.resourceName} を自動停止しました`);
        } catch (error) {
          logger.error(`Failed to auto-optimize resource ${resource.resourceName}:`, error);
        }
      }
    }
  }

  async startMonitoring(zones: string[], intervalMinutes = 30): Promise<void> {
    logger.info(`📊 リソース監視を開始します（間隔: ${intervalMinutes}分）`);

    const monitoringInterval = setInterval(async () => {
      try {
        const usage = await this.getAllResourceUsage(zones);
        const alerts = await this.generateCostAlerts(usage);

        if (alerts.length > 0) {
          logger.warn(`⚠️ ${alerts.length}個のコストアラートが発生しました:`);
          alerts.forEach(alert => {
            logger.warn(`   ${alert.severity.toUpperCase()}: ${alert.message}`);
            if (alert.recommendedAction) {
              logger.warn(`   推奨アクション: ${alert.recommendedAction}`);
            }
          });

          // 自動最適化の実行
          await this.executeAutomatedOptimization(alerts);
        }

        // レポートの出力
        this.generateResourceReport(usage);

      } catch (error) {
        logger.error('監視サイクルでエラーが発生:', error);
      }
    }, intervalMinutes * 60 * 1000);

    // Graceful shutdown
    process.on('SIGINT', () => {
      clearInterval(monitoringInterval);
      logger.info('📊 リソース監視を終了しました');
      process.exit(0);
    });
  }

  private calculateRunningTime(createTime: string): number {
    const created = new Date(createTime);
    const now = new Date();
    return Math.floor((now.getTime() - created.getTime()) / (1000 * 60)); // minutes
  }

  private estimateCurrentCost(acceleratorType: string, computeType: string, createTime: string): number {
    const runningTimeHours = this.calculateRunningTime(createTime) / 60;
    
    if (computeType === 'tpu') {
      const hourlyRates = {
        'v2-8': 4.5,
        'v3-8': 8.0,
        'v4-8': 16.0,
        'v5p-8': 24.0,
      };
      return Math.round((hourlyRates[acceleratorType] || 8.0) * runningTimeHours);
    } else {
      const hourlyRates = {
        'nvidia-l4': 0.6,
        'nvidia-tesla-a100': 2.9,
        'nvidia-h100-80gb': 8.5,
      };
      return Math.round((hourlyRates[acceleratorType] || 2.0) * runningTimeHours);
    }
  }

  private generateResourceReport(usage: ResourceUsage[]): void {
    const totalCost = usage.reduce((sum, r) => sum + r.estimatedCost, 0);
    const runningCount = usage.filter(r => r.status === 'RUNNING' || r.status === 'READY').length;

    logger.info('📋 リソースサマリーレポート:');
    logger.info(`   総リソース数: ${usage.length}`);
    logger.info(`   稼働中リソース: ${runningCount}`);
    logger.info(`   推定総コスト: ¥${totalCost}`);
    logger.info(`   TPUリソース: ${usage.filter(r => r.computeType === 'tpu').length}`);
    logger.info(`   GPUリソース: ${usage.filter(r => r.computeType === 'gpu').length}`);
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
    winston.format.printf(({ timestamp, level, message, ...meta }) => {
      return `${timestamp} [${level}] ${message} ${Object.keys(meta).length ? JSON.stringify(meta, null, 2) : ''}`;
    })
  ),
  defaultMeta: { service: 'gcp-ml-platform' },
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
  
  // デフォルトのゾーン・リージョン設定
  defaultTPUZone: process.env.DEFAULT_TPU_ZONE || 'us-central1-a',
  defaultGPUZone: process.env.DEFAULT_GPU_ZONE || 'us-central1-a',
  monitoringZones: process.env.MONITORING_ZONES?.split(',') || ['us-central1-a', 'asia-northeast1-a'],
  
  // 監視・アラート設定
  monitoring: {
    intervalMinutes: parseInt(process.env.MONITORING_INTERVAL || '30'),
    dailyBudgetLimit: parseInt(process.env.DAILY_BUDGET_LIMIT || '10000'),
    idleTimeThresholdMinutes: parseInt(process.env.IDLE_THRESHOLD_MINUTES || '60'),
    enableAutomatedOptimization: process.env.ENABLE_AUTO_OPTIMIZATION === 'true',
  },
  
  // デフォルトのTPU設定
  defaultTPUConfig: {
    runtimeVersions: {
      tensorflow: 'tpu-vm-tf-2.15.0-v4',
      pytorch: 'tpu-vm-pt-2.1-v4',
      jax: 'tpu-vm-base',
    },
  },
  
  // デフォルトのGPU設定
  defaultGPUConfig: {
    diskImages: {
      tensorflow: 'c0-deeplearning-common-cu121-v20240922-ubuntu-2204',
      pytorch: 'c0-deeplearning-common-cu121-v20240922-ubuntu-2204',
      general: 'c0-deeplearning-common-cu121-v20240922-ubuntu-2204',
    },
    networkTags: ['ml-workload', 'gpu-instance'],
  },
};

// 必須設定の検証
if (!config.projectId) {
  throw new Error('GOOGLE_CLOUD_PROJECT environment variable is required');
}
```

**🎯 統合CLIツール**

```typescript:src/cli/ml-platform-cli.ts
#!/usr/bin/env node
import { Command } from 'commander';
import { MLPlatformManager } from '../services/MLPlatformManager';
import { ResourceMonitor } from '../services/ResourceMonitor';
import { config } from '../config';
import { logger } from '../utils/logger';

const program = new Command();
const mlPlatform = new MLPlatformManager(config.projectId);
const monitor = new ResourceMonitor(config.projectId);

program
  .name('ml-platform')
  .description('GCP ML Platform Management CLI')
  .version('1.0.0');

// ワークロードの推奨を取得
program
  .command('recommend')
  .description('Get compute recommendations for ML workload')
  .option('-t, --type <type>', 'workload type (training|inference|research)', 'training')
  .option('-f, --framework <framework>', 'ML framework (tensorflow|pytorch|jax)', 'tensorflow')
  .option('-s, --size <size>', 'model size (small|medium|large|xlarge)', 'medium')
  .option('-z, --zone <zone>', 'GCP zone', config.defaultTPUZone)
  .option('-b, --budget <budget>', 'budget limit (JPY)', '5000')
  .action(async (options) => {
    const workloadConfig = {
      workloadType: options.type as any,
      model: {
        name: `model-${Date.now()}`,
        framework: options.framework as any,
        size: options.size as any,
      },
      compute: {
        preferredType: 'auto' as const,
        zone: options.zone,
        budget: parseInt(options.budget),
      },
    };

    try {
      const recommendation = await mlPlatform.recommendCompute(workloadConfig);
      
      console.log('🤖 ML Platform 推奨結果:');
      console.log(`推奨コンピュート: ${recommendation.recommendedType.toUpperCase()}`);
      console.log(`推定コスト: ¥${recommendation.estimatedCost}`);
      console.log('理由:');
      recommendation.reasoning.forEach(reason => console.log(`  • ${reason}`));
      
    } catch (error) {
      logger.error('推奨の取得に失敗:', error);
      process.exit(1);
    }
  });

// ワークロードのデプロイ
program
  .command('deploy')
  .description('Deploy ML workload')
  .requiredOption('-n, --name <name>', 'workload name')
  .option('-t, --type <type>', 'compute type (tpu|gpu)', 'auto')
  .option('-z, --zone <zone>', 'GCP zone', config.defaultTPUZone)
  .action(async (options) => {
    // デプロイロジックの実装
    console.log(`🚀 ワークロード「${options.name}」をデプロイ中...`);
    // 実装省略...
  });

// リソース監視の開始
program
  .command('monitor')
  .description('Start resource monitoring')
  .option('-i, --interval <minutes>', 'monitoring interval in minutes', '30')
  .action(async (options) => {
    const intervalMinutes = parseInt(options.interval);
    await monitor.startMonitoring(config.monitoringZones, intervalMinutes);
  });

// リソースのリスト表示
program
  .command('list')
  .description('List all ML resources')
  .option('-z, --zone <zone>', 'specific zone to list', '')
  .action(async (options) => {
    try {
      const zones = options.zone ? [options.zone] : config.monitoringZones;
      const usage = await monitor.getAllResourceUsage(zones);
      
      console.log('📋 稼働中のMLリソース:');
      console.table(usage);
      
    } catch (error) {
      logger.error('リソース一覧の取得に失敗:', error);
      process.exit(1);
    }
  });

program.parse();
```

## まとめ

GCP Cloud TPU・GPU × TypeScript の組み合わせで、高性能な機械学習基盤の自動管理が実現できました。

**✅ この記事で身についたスキル**

>* Cloud TPU と GPU の特徴理解と適切な使い分け判断
>* TypeScript + Google Cloud Client Library での TPU・GPU 自動化
>* ワークロード最適化と推奨システムの実装パターン  
>* リソース監視とコスト最適化の自動化システム構築
>* 機械学習基盤をコードで完全制御する Infrastructure as Code

**🎯 次のステップ**

>* Kubernetes Engine（GKE）との連携でコンテナオーケストレーション
>* Vertex AI との統合で MLOps パイプライン構築
>* BigQuery・Cloud Storage との連携で データパイプライン自動化

2025年の最新TPU v7や NVIDIA GB200 Grace Blackwell を活用して、あなたも次世代の機械学習基盤をプログラマブルに構築してみませんか？コスト効率と性能を両立した AI 開発環境で、革新的なモデル開発を加速させましょう！