## もうコンテナ管理で迷わない！Google Kubernetes Engine (GKE) をTypeScriptで完全制御する実践ガイド

## はじめに

「Kubernetesのクラスター管理は複雑すぎて、手作業でのデプロイは時間がかかりすぎる」

そんな悩みを抱えるDevOpsエンジニアや開発者の方に朗報です。Google Kubernetes Engine（GKE）なら、**TypeScript**を使ってクラスターの作成からアプリケーションデプロイ、スケーリングまで全てをプログラマブルに管理できます。

この記事では、**Google Cloud SDK** と **TypeScript** を組み合わせてGKEクラスターを効率的に管理し、実際の運用で使える実践的なコードパターンを解説します。記事を読み終える頃には、あなたもKubernetes as Code の真髄を体感できているはずです 🚀

>* Google Kubernetes Engine の基本概念とメリット・デメリットの理解
>* TypeScript + Google Cloud Client Library でのクラスター管理自動化
>* Node Pool、Workload、Service の作成・管理の実装
>* 監視・ログ管理・オートスケーリングの設定パターン

## Google Kubernetes Engine (GKE) とは？

Google Kubernetes Engine は、Googleが提供する**フルマネージドなKubernetesサービス**です。

**1. Google品質のKubernetesクラスター**

>* Kubernetesの開発元であるGoogleによる最新版サポート
>* マスターノードの管理・アップデートが完全自動化
>* 99.95%の稼働率SLA保証（リージョナルクラスター）

**2. スケーラビリティとパフォーマンス**

>* Cluster Autoscaler による自動ノードスケーリング
>* Horizontal Pod Autoscaler（HPA）とVertical Pod Autoscaler（VPA）対応
>* GPUやTPU対応でML/AI ワークロードにも最適

**3. エンタープライズレディな運用機能**

>* Google Cloud の IAM・セキュリティとの統合
>* Cloud Logging・Cloud Monitoring による包括的な可視化
>* Binary Authorization、Pod Security Policy による強固なセキュリティ

**⚖️ 他のマネージドKubernetesとの比較**

| サービス | 管理負荷 | 料金体系 | エコシステム | 適用場面 |
|---------|---------|----------|------------|---------|
| **GKE** | 低 | マスター無料 | GCP統合 | 高パフォーマンス重視 |
| **EKS** | 中 | マスター有料 | AWS統合 | AWS既存環境 |
| **AKS** | 中 | マスター無料 | Azure統合 | エンタープライズ |

**📌 GKEを選ぶべきシーン**

>* 機械学習・データ分析でGPU/TPUが必要
>* BigQuery・Cloud Storage等のGCPサービスとの連携が多い
>* Kubernetes運用の管理負荷を最小限に抑えたい

## 開発環境のセットアップ

GKEをTypeScriptで管理するための開発環境を構築しましょう。

**📦 必要なツールのインストール**

```bash
# Google Cloud SDK のインストール（公式サイトからダウンロード）
# https://cloud.google.com/sdk/docs/install

# Cloud SDK の初期化とログイン
gcloud init
gcloud auth login
gcloud auth application-default login

# プロジェクトの作成と設定
gcloud projects create your-gke-project --name="My GKE Project"
gcloud config set project your-gke-project

# 必要なAPIの有効化
gcloud services enable container.googleapis.com
gcloud services enable compute.googleapis.com
gcloud services enable monitoring.googleapis.com

# kubectlのインストール
gcloud components install kubectl
```

**🚀 TypeScriptプロジェクトの初期化**

```bash
# プロジェクトディレクトリの作成
mkdir gke-typescript-manager && cd gke-typescript-manager

# Node.jsプロジェクトの初期化
npm init -y

# Google Cloud Client Library のインストール
npm install @google-cloud/container
npm install @kubernetes/client-node
npm install -D typescript @types/node ts-node nodemon

# 設定管理・ユーティリティライブラリ
npm install dotenv winston yaml
npm install -D @types/dotenv @types/js-yaml
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

## GKE クラスター管理の実装

実用的なGKEクラスター管理システムをTypeScriptで構築してみましょう。

### 1\. 基本的なクラスター操作クラス

```typescript:src/services/GKEClusterManager.ts
import { ClusterManagerClient } from '@google-cloud/container';
import { logger } from '../utils/logger';

export interface ClusterConfig {
  name: string;
  zone: string;
  location: string;
  nodeCount: number;
  machineType?: string;
  diskSize?: number;
  oauthScopes?: string[];
  kubernetesVersion?: string;
  enableAutopilot?: boolean;
}

export interface NodePoolConfig {
  name: string;
  clusterName: string;
  nodeCount: number;
  machineType: string;
  diskSize?: number;
  preemptible?: boolean;
  autoUpgrade?: boolean;
  autoRepair?: boolean;
}

export interface ClusterInfo {
  name: string;
  status: string;
  location: string;
  kubernetesVersion: string;
  nodeCount: number;
  endpoint: string;
  createdAt: string;
  nodePools: any[];
}

export class GKEClusterManager {
  private client: ClusterManagerClient;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.client = new ClusterManagerClient();
  }

  async createCluster(config: ClusterConfig): Promise<ClusterInfo> {
    const {
      name,
      location,
      nodeCount,
      machineType = 'e2-medium',
      diskSize = 100,
      oauthScopes = [
        'https://www.googleapis.com/auth/compute',
        'https://www.googleapis.com/auth/devstorage.read_only',
        'https://www.googleapis.com/auth/logging.write',
        'https://www.googleapis.com/auth/monitoring'
      ],
      kubernetesVersion,
      enableAutopilot = false
    } = config;

    const parent = `projects/${this.projectId}/locations/${location}`;
    
    const cluster = {
      name,
      initialNodeCount: nodeCount,
      nodeConfig: {
        machineType,
        diskSizeGb: diskSize,
        oauthScopes,
        imageType: 'COS_CONTAINERD',
      },
      ...(kubernetesVersion && { initialClusterVersion: kubernetesVersion }),
      ...(enableAutopilot && { autopilot: { enabled: true } }),
      // セキュリティ設定
      networkPolicy: {
        enabled: true,
        provider: 'CALICO',
      },
      ipAllocationPolicy: {
        useIpAliases: true,
      },
      // オートスケーリング設定
      clusterAutoscaling: {
        enabled: true,
        resourceLimits: [
          {
            resourceType: 'cpu',
            minimum: '1',
            maximum: '100',
          },
          {
            resourceType: 'memory',
            minimum: '1',
            maximum: '1000',
          },
        ],
      },
      // アドオン設定
      addonsConfig: {
        httpLoadBalancing: { disabled: false },
        horizontalPodAutoscaling: { disabled: false },
        kubernetesDashboard: { disabled: true }, // セキュリティのため無効
        networkPolicyConfig: { disabled: false },
      },
    };

    try {
      logger.info(`Creating GKE cluster: ${name} in location: ${location}`);
      
      const [operation] = await this.client.createCluster({
        parent,
        cluster,
      });

      logger.info(`Cluster creation operation started: ${operation.name}`);
      
      // オペレーション完了を待つ
      await this.waitForOperation(operation.name!, location);
      
      // 作成されたクラスターの詳細を取得
      const clusterInfo = await this.getCluster(name, location);
      
      logger.info(`Successfully created cluster: ${name}`);
      return clusterInfo;

    } catch (error) {
      logger.error(`Failed to create cluster ${name}:`, error);
      throw new Error(`Cluster creation failed: ${error.message}`);
    }
  }

  async getCluster(name: string, location: string): Promise<ClusterInfo> {
    try {
      const clusterName = `projects/${this.projectId}/locations/${location}/clusters/${name}`;
      
      const [cluster] = await this.client.getCluster({
        name: clusterName,
      });

      return {
        name: cluster.name!,
        status: cluster.status!,
        location: cluster.location!,
        kubernetesVersion: cluster.currentMasterVersion!,
        nodeCount: cluster.currentNodeCount!,
        endpoint: cluster.endpoint!,
        createdAt: cluster.createTime!,
        nodePools: cluster.nodePools || [],
      };

    } catch (error) {
      logger.error(`Failed to get cluster ${name}:`, error);
      throw new Error(`Cluster retrieval failed: ${error.message}`);
    }
  }

  async listClusters(location: string): Promise<ClusterInfo[]> {
    try {
      const parent = `projects/${this.projectId}/locations/${location}`;
      
      const [response] = await this.client.listClusters({ parent });
      const clusters = response.clusters || [];

      return clusters.map(cluster => ({
        name: cluster.name!,
        status: cluster.status!,
        location: cluster.location!,
        kubernetesVersion: cluster.currentMasterVersion!,
        nodeCount: cluster.currentNodeCount!,
        endpoint: cluster.endpoint!,
        createdAt: cluster.createTime!,
        nodePools: cluster.nodePools || [],
      }));

    } catch (error) {
      logger.error(`Failed to list clusters in location ${location}:`, error);
      throw new Error(`Cluster listing failed: ${error.message}`);
    }
  }

  async deleteCluster(name: string, location: string): Promise<void> {
    try {
      const clusterName = `projects/${this.projectId}/locations/${location}/clusters/${name}`;
      
      logger.info(`Deleting cluster: ${name} in location: ${location}`);
      
      const [operation] = await this.client.deleteCluster({
        name: clusterName,
      });

      await this.waitForOperation(operation.name!, location);
      
      logger.info(`Successfully deleted cluster: ${name}`);

    } catch (error) {
      logger.error(`Failed to delete cluster ${name}:`, error);
      throw new Error(`Cluster deletion failed: ${error.message}`);
    }
  }

  async createNodePool(clusterName: string, location: string, config: NodePoolConfig): Promise<void> {
    const {
      name,
      nodeCount,
      machineType,
      diskSize = 100,
      preemptible = false,
      autoUpgrade = true,
      autoRepair = true,
    } = config;

    const parent = `projects/${this.projectId}/locations/${location}/clusters/${clusterName}`;
    
    const nodePool = {
      name,
      initialNodeCount: nodeCount,
      config: {
        machineType,
        diskSizeGb: diskSize,
        oauthScopes: [
          'https://www.googleapis.com/auth/compute',
          'https://www.googleapis.com/auth/devstorage.read_only',
          'https://www.googleapis.com/auth/logging.write',
          'https://www.googleapis.com/auth/monitoring'
        ],
        preemptible,
        imageType: 'COS_CONTAINERD',
      },
      autoscaling: {
        enabled: true,
        minNodeCount: 1,
        maxNodeCount: nodeCount * 3,
      },
      management: {
        autoUpgrade,
        autoRepair,
      },
    };

    try {
      logger.info(`Creating node pool: ${name} in cluster: ${clusterName}`);
      
      const [operation] = await this.client.createNodePool({
        parent,
        nodePool,
      });

      await this.waitForOperation(operation.name!, location);
      
      logger.info(`Successfully created node pool: ${name}`);

    } catch (error) {
      logger.error(`Failed to create node pool ${name}:`, error);
      throw new Error(`Node pool creation failed: ${error.message}`);
    }
  }

  private async waitForOperation(operationName: string, location: string): Promise<void> {
    const operationPath = `projects/${this.projectId}/locations/${location}/operations/${operationName.split('/').pop()}`;
    
    let operation;
    do {
      [operation] = await this.client.getOperation({
        name: operationPath,
      });

      if (operation.status === 'DONE') {
        if (operation.statusMessage && operation.statusMessage !== 'SUCCESS') {
          throw new Error(`Operation failed: ${operation.statusMessage}`);
        }
        break;
      }

      logger.info(`Operation ${operationName} in progress... Status: ${operation.status}`);
      
      // 30秒間待ってから再チェック
      await new Promise(resolve => setTimeout(resolve, 30000));
    } while (operation.status !== 'DONE');
  }
}
```

### 2\. Kubernetes API との連携

```typescript:src/services/KubernetesManager.ts
import * as k8s from '@kubernetes/client-node';
import { logger } from '../utils/logger';

export interface DeploymentConfig {
  name: string;
  namespace: string;
  image: string;
  replicas: number;
  containerPort: number;
  env?: Record<string, string>;
  resources?: {
    requests?: { cpu: string; memory: string };
    limits?: { cpu: string; memory: string };
  };
}

export interface ServiceConfig {
  name: string;
  namespace: string;
  selector: Record<string, string>;
  ports: Array<{ port: number; targetPort: number; protocol?: string }>;
  type?: 'ClusterIP' | 'NodePort' | 'LoadBalancer';
}

export class KubernetesManager {
  private kc: k8s.KubeConfig;
  private k8sApi: k8s.AppsV1Api;
  private coreApi: k8s.CoreV1Api;

  constructor(clusterEndpoint: string, clusterCaCertificate: string, accessToken: string) {
    this.kc = new k8s.KubeConfig();
    
    // GKEクラスター接続情報を設定
    this.kc.loadFromOptions({
      clusters: [{
        name: 'gke-cluster',
        server: `https://${clusterEndpoint}`,
        caData: clusterCaCertificate,
      }],
      users: [{
        name: 'gke-user',
        token: accessToken,
      }],
      contexts: [{
        name: 'gke-context',
        cluster: 'gke-cluster',
        user: 'gke-user',
      }],
      currentContext: 'gke-context',
    });

    this.k8sApi = this.kc.makeApiClient(k8s.AppsV1Api);
    this.coreApi = this.kc.makeApiClient(k8s.CoreV1Api);
  }

  async createNamespace(name: string): Promise<void> {
    const namespace = {
      metadata: {
        name,
      },
    };

    try {
      await this.coreApi.createNamespace(namespace);
      logger.info(`Created namespace: ${name}`);
    } catch (error) {
      if (error.response?.statusCode !== 409) { // Already exists
        logger.error(`Failed to create namespace ${name}:`, error);
        throw new Error(`Namespace creation failed: ${error.message}`);
      }
      logger.info(`Namespace ${name} already exists`);
    }
  }

  async createDeployment(config: DeploymentConfig): Promise<void> {
    const {
      name,
      namespace,
      image,
      replicas,
      containerPort,
      env = {},
      resources,
    } = config;

    const deployment = {
      metadata: {
        name,
        namespace,
      },
      spec: {
        replicas,
        selector: {
          matchLabels: {
            app: name,
          },
        },
        template: {
          metadata: {
            labels: {
              app: name,
            },
          },
          spec: {
            containers: [{
              name,
              image,
              ports: [{
                containerPort,
              }],
              env: Object.entries(env).map(([key, value]) => ({
                name: key,
                value,
              })),
              ...(resources && { resources }),
            }],
          },
        },
      },
    };

    try {
      await this.k8sApi.createNamespacedDeployment(namespace, deployment);
      logger.info(`Created deployment: ${name} in namespace: ${namespace}`);
    } catch (error) {
      logger.error(`Failed to create deployment ${name}:`, error);
      throw new Error(`Deployment creation failed: ${error.message}`);
    }
  }

  async createService(config: ServiceConfig): Promise<void> {
    const {
      name,
      namespace,
      selector,
      ports,
      type = 'ClusterIP',
    } = config;

    const service = {
      metadata: {
        name,
        namespace,
      },
      spec: {
        type,
        selector,
        ports: ports.map(port => ({
          port: port.port,
          targetPort: port.targetPort,
          protocol: port.protocol || 'TCP',
        })),
      },
    };

    try {
      await this.coreApi.createNamespacedService(namespace, service);
      logger.info(`Created service: ${name} in namespace: ${namespace}`);
    } catch (error) {
      logger.error(`Failed to create service ${name}:`, error);
      throw new Error(`Service creation failed: ${error.message}`);
    }
  }

  async scaleDeployment(name: string, namespace: string, replicas: number): Promise<void> {
    try {
      const patch = [
        {
          op: 'replace',
          path: '/spec/replicas',
          value: replicas,
        },
      ];

      await this.k8sApi.patchNamespacedDeployment(
        name, 
        namespace,
        patch,
        undefined,
        undefined,
        undefined,
        undefined,
        undefined,
        {
          headers: { 'Content-Type': 'application/json-patch+json' },
        }
      );

      logger.info(`Scaled deployment ${name} to ${replicas} replicas`);
    } catch (error) {
      logger.error(`Failed to scale deployment ${name}:`, error);
      throw new Error(`Deployment scaling failed: ${error.message}`);
    }
  }

  async getDeploymentStatus(name: string, namespace: string): Promise<any> {
    try {
      const response = await this.k8sApi.readNamespacedDeployment(name, namespace);
      const deployment = response.body;

      return {
        name: deployment.metadata?.name,
        namespace: deployment.metadata?.namespace,
        replicas: deployment.spec?.replicas,
        readyReplicas: deployment.status?.readyReplicas || 0,
        updatedReplicas: deployment.status?.updatedReplicas || 0,
        availableReplicas: deployment.status?.availableReplicas || 0,
      };
    } catch (error) {
      logger.error(`Failed to get deployment status for ${name}:`, error);
      throw new Error(`Deployment status retrieval failed: ${error.message}`);
    }
  }

  async getPodLogs(namespace: string, podName: string, containerName?: string): Promise<string> {
    try {
      const response = await this.coreApi.readNamespacedPodLog(
        podName,
        namespace,
        containerName,
        undefined,
        undefined,
        undefined,
        undefined,
        undefined,
        undefined,
        100 // 最新100行
      );

      return response.body;
    } catch (error) {
      logger.error(`Failed to get logs for pod ${podName}:`, error);
      throw new Error(`Pod log retrieval failed: ${error.message}`);
    }
  }
}
```

### 3\. 実践的な使用例

```typescript:src/examples/gkeUsage.ts
import { GKEClusterManager, KubernetesManager } from '../services';
import { logger } from '../utils/logger';

async function createProductionCluster() {
  const projectId = process.env.GOOGLE_CLOUD_PROJECT!;
  const gkeManager = new GKEClusterManager(projectId);

  const clusterConfig = {
    name: 'production-cluster',
    location: 'asia-northeast1', // リージョナルクラスター
    zone: 'asia-northeast1-a',
    nodeCount: 3,
    machineType: 'e2-standard-4',
    diskSize: 100,
    kubernetesVersion: '1.27',
    enableAutopilot: false,
  };

  try {
    // クラスター作成
    const cluster = await gkeManager.createCluster(clusterConfig);
    logger.info('Production cluster created:', cluster);

    // ワーカーノード用のNode Pool作成
    await gkeManager.createNodePool(clusterConfig.name, clusterConfig.location, {
      name: 'worker-pool',
      clusterName: clusterConfig.name,
      nodeCount: 5,
      machineType: 'e2-standard-2',
      diskSize: 50,
      preemptible: false,
      autoUpgrade: true,
      autoRepair: true,
    });

    // 開発用のプリエンプティブルNode Pool作成
    await gkeManager.createNodePool(clusterConfig.name, clusterConfig.location, {
      name: 'dev-pool',
      clusterName: clusterConfig.name,
      nodeCount: 2,
      machineType: 'e2-medium',
      diskSize: 30,
      preemptible: true,
      autoUpgrade: true,
      autoRepair: true,
    });

    logger.info('All node pools created successfully');

  } catch (error) {
    logger.error('Failed to create production cluster:', error);
  }
}

async function deployWebApplication(
  clusterEndpoint: string, 
  clusterCaCertificate: string, 
  accessToken: string
) {
  const k8sManager = new KubernetesManager(
    clusterEndpoint,
    clusterCaCertificate,
    accessToken
  );

  try {
    // Namespace作成
    await k8sManager.createNamespace('web-app');

    // Web アプリケーションのDeployment作成
    await k8sManager.createDeployment({
      name: 'web-app',
      namespace: 'web-app',
      image: 'gcr.io/your-project/web-app:latest',
      replicas: 3,
      containerPort: 3000,
      env: {
        NODE_ENV: 'production',
        PORT: '3000',
        DATABASE_URL: 'postgresql://...',
      },
      resources: {
        requests: { cpu: '100m', memory: '128Mi' },
        limits: { cpu: '500m', memory: '512Mi' },
      },
    });

    // LoadBalancer Service作成
    await k8sManager.createService({
      name: 'web-app-service',
      namespace: 'web-app',
      selector: { app: 'web-app' },
      ports: [{ port: 80, targetPort: 3000 }],
      type: 'LoadBalancer',
    });

    // APIサーバーのDeployment作成
    await k8sManager.createDeployment({
      name: 'api-server',
      namespace: 'web-app',
      image: 'gcr.io/your-project/api-server:latest',
      replicas: 2,
      containerPort: 8080,
      env: {
        NODE_ENV: 'production',
        PORT: '8080',
        REDIS_URL: 'redis://redis-service:6379',
      },
      resources: {
        requests: { cpu: '200m', memory: '256Mi' },
        limits: { cpu: '1000m', memory: '1Gi' },
      },
    });

    // API Service作成（ClusterIP）
    await k8sManager.createService({
      name: 'api-service',
      namespace: 'web-app',
      selector: { app: 'api-server' },
      ports: [{ port: 8080, targetPort: 8080 }],
      type: 'ClusterIP',
    });

    logger.info('Web application deployed successfully');

    // デプロイメントのステータス確認
    const webAppStatus = await k8sManager.getDeploymentStatus('web-app', 'web-app');
    const apiServerStatus = await k8sManager.getDeploymentStatus('api-server', 'web-app');
    
    logger.info('Deployment statuses:', { webAppStatus, apiServerStatus });

  } catch (error) {
    logger.error('Failed to deploy web application:', error);
  }
}

async function scaleApplicationBasedOnLoad() {
  const clusterEndpoint = process.env.CLUSTER_ENDPOINT!;
  const clusterCaCertificate = process.env.CLUSTER_CA_CERT!;
  const accessToken = process.env.ACCESS_TOKEN!;

  const k8sManager = new KubernetesManager(
    clusterEndpoint,
    clusterCaCertificate,
    accessToken
  );

  try {
    // 現在のデプロイメント状態を確認
    const status = await k8sManager.getDeploymentStatus('web-app', 'web-app');
    logger.info('Current deployment status:', status);

    // 負荷に応じてスケーリング（実際の実装では監視メトリクスを参照）
    const targetReplicas = Math.max(3, Math.min(10, status.replicas * 2));
    
    await k8sManager.scaleDeployment('web-app', 'web-app', targetReplicas);
    logger.info(`Scaled web-app to ${targetReplicas} replicas`);

  } catch (error) {
    logger.error('Failed to scale application:', error);
  }
}

// 実行例
createProductionCluster();

// クラスター作成後に実行
// deployWebApplication('cluster-endpoint', 'ca-certificate', 'access-token');
// scaleApplicationBasedOnLoad();
```

##  Cloud Monitoring との連携

```typescript:src/services/GKEMonitoringService.ts
import { MetricServiceClient } from '@google-cloud/monitoring';
import { logger } from '../utils/logger';

export interface ClusterMetrics {
  clusterName: string;
  location: string;
  cpuUtilization: number;
  memoryUtilization: number;
  nodeCount: number;
  podCount: number;
  timestamp: Date;
}

export class GKEMonitoringService {
  private client: MetricServiceClient;
  private projectId: string;
  private projectPath: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.client = new MetricServiceClient();
    this.projectPath = this.client.projectPath(projectId);
  }

  async getClusterMetrics(clusterName: string, location: string, hours = 1): Promise<ClusterMetrics[]> {
    const now = new Date();
    const startTime = new Date(now.getTime() - hours * 60 * 60 * 1000);

    const filter = `
      resource.type="k8s_cluster" AND 
      resource.labels.cluster_name="${clusterName}" AND 
      resource.labels.location="${location}"
    `;

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
      
      const metrics: ClusterMetrics[] = [];
      
      for (const series of timeSeries) {
        if (series.points) {
          for (const point of series.points) {
            metrics.push({
              clusterName,
              location,
              cpuUtilization: this.extractCPUUtilization(series),
              memoryUtilization: this.extractMemoryUtilization(series),
              nodeCount: this.extractNodeCount(series),
              podCount: this.extractPodCount(series),
              timestamp: new Date(point.interval?.endTime?.seconds! * 1000),
            });
          }
        }
      }

      return metrics;

    } catch (error) {
      logger.error(`Failed to get cluster metrics:`, error);
      throw new Error(`Metrics retrieval failed: ${error.message}`);
    }
  }

  async createCustomAlert(clusterName: string, location: string): Promise<void> {
    // カスタムアラートポリシーの作成例
    const alertPolicy = {
      displayName: `GKE Cluster ${clusterName} High CPU Alert`,
      conditions: [
        {
          displayName: 'CPU usage high',
          conditionThreshold: {
            filter: `
              resource.type="k8s_cluster" AND 
              resource.labels.cluster_name="${clusterName}" AND 
              metric.type="kubernetes.io/container/cpu/core_usage_time"
            `,
            comparison: 'COMPARISON_GREATER_THAN',
            thresholdValue: 0.8, // 80%
            duration: {
              seconds: 300, // 5分間
            },
            aggregations: [
              {
                alignmentPeriod: {
                  seconds: 60,
                },
                perSeriesAligner: 'ALIGN_RATE',
                crossSeriesReducer: 'REDUCE_MEAN',
                groupByFields: ['resource.labels.cluster_name'],
              },
            ],
          },
        },
      ],
      enabled: true,
    };

    try {
      // アラートポリシーの作成（実際の実装では@google-cloud/monitoringを使用）
      logger.info(`Created alert policy for cluster: ${clusterName}`);
    } catch (error) {
      logger.error('Failed to create alert policy:', error);
      throw new Error(`Alert policy creation failed: ${error.message}`);
    }
  }

  private extractCPUUtilization(series: any): number {
    // メトリクスからCPU使用率を抽出
    return series.points?.[0]?.value?.doubleValue || 0;
  }

  private extractMemoryUtilization(series: any): number {
    // メトリクスからメモリ使用率を抽出
    return series.points?.[0]?.value?.doubleValue || 0;
  }

  private extractNodeCount(series: any): number {
    // メトリクスからノード数を抽出
    return series.points?.[0]?.value?.int64Value || 0;
  }

  private extractPodCount(series: any): number {
    // メトリクスからPod数を抽出
    return series.points?.[0]?.value?.int64Value || 0;
  }
}
```

## GitHub Actions との連携

```typescript:src/services/CICDIntegrationService.ts
import { GKEClusterManager, KubernetesManager } from './index';
import { logger } from '../utils/logger';

export interface DeploymentPipeline {
  environment: 'development' | 'staging' | 'production';
  clusterName: string;
  location: string;
  namespace: string;
  imageTag: string;
  appName: string;
}

export class CICDIntegrationService {
  private gkeManager: GKEClusterManager;
  private projectId: string;

  constructor(projectId: string) {
    this.projectId = projectId;
    this.gkeManager = new GKEClusterManager(projectId);
  }

  async deployToEnvironment(pipeline: DeploymentPipeline): Promise<void> {
    const { 
      environment, 
      clusterName, 
      location, 
      namespace, 
      imageTag, 
      appName 
    } = pipeline;

    try {
      logger.info(`Starting deployment to ${environment} environment`);

      // クラスターの存在確認
      const cluster = await this.gkeManager.getCluster(clusterName, location);
      if (cluster.status !== 'RUNNING') {
        throw new Error(`Cluster ${clusterName} is not in RUNNING state`);
      }

      // Kubernetes接続情報を取得（実際の実装では認証情報を適切に取得）
      const accessToken = await this.getAccessToken();
      const k8sManager = new KubernetesManager(
        cluster.endpoint,
        '', // CA証明書は実際の実装で取得
        accessToken
      );

      // Namespace作成
      await k8sManager.createNamespace(namespace);

      // 環境別設定の取得
      const deploymentConfig = this.getEnvironmentConfig(environment, appName, imageTag);

      // デプロイメント更新
      await this.updateDeployment(k8sManager, namespace, deploymentConfig);

      // デプロイメント完了の確認
      await this.waitForDeploymentReady(k8sManager, namespace, appName);

      logger.info(`Successfully deployed ${appName}:${imageTag} to ${environment}`);

    } catch (error) {
      logger.error(`Deployment failed:`, error);
      throw new Error(`Deployment failed: ${error.message}`);
    }
  }

  async rollbackDeployment(
    clusterName: string, 
    location: string, 
    namespace: string, 
    appName: string
  ): Promise<void> {
    try {
      const cluster = await this.gkeManager.getCluster(clusterName, location);
      const accessToken = await this.getAccessToken();
      
      const k8sManager = new KubernetesManager(
        cluster.endpoint,
        '',
        accessToken
      );

      // 前のリビジョンにロールバック（kubectl rollout undo相当）
      logger.info(`Rolling back deployment: ${appName} in namespace: ${namespace}`);
      
      // 実際の実装ではkubectl相当のAPI呼び出しを行う
      await this.performRollback(k8sManager, namespace, appName);

      logger.info(`Successfully rolled back deployment: ${appName}`);

    } catch (error) {
      logger.error(`Rollback failed:`, error);
      throw new Error(`Rollback failed: ${error.message}`);
    }
  }

  private getEnvironmentConfig(environment: string, appName: string, imageTag: string): any {
    const baseConfig = {
      name: appName,
      image: `gcr.io/${this.projectId}/${appName}:${imageTag}`,
      containerPort: 3000,
    };

    switch (environment) {
      case 'development':
        return {
          ...baseConfig,
          replicas: 1,
          resources: {
            requests: { cpu: '50m', memory: '64Mi' },
            limits: { cpu: '200m', memory: '256Mi' },
          },
          env: {
            NODE_ENV: 'development',
            LOG_LEVEL: 'debug',
          },
        };

      case 'staging':
        return {
          ...baseConfig,
          replicas: 2,
          resources: {
            requests: { cpu: '100m', memory: '128Mi' },
            limits: { cpu: '500m', memory: '512Mi' },
          },
          env: {
            NODE_ENV: 'staging',
            LOG_LEVEL: 'info',
          },
        };

      case 'production':
        return {
          ...baseConfig,
          replicas: 3,
          resources: {
            requests: { cpu: '200m', memory: '256Mi' },
            limits: { cpu: '1000m', memory: '1Gi' },
          },
          env: {
            NODE_ENV: 'production',
            LOG_LEVEL: 'warn',
          },
        };

      default:
        throw new Error(`Unknown environment: ${environment}`);
    }
  }

  private async updateDeployment(
    k8sManager: KubernetesManager, 
    namespace: string, 
    config: any
  ): Promise<void> {
    try {
      // 既存のDeploymentがあるかチェック
      const status = await k8sManager.getDeploymentStatus(config.name, namespace);
      
      if (status) {
        // 既存のDeploymentを更新（イメージタグの更新など）
        await this.updateExistingDeployment(k8sManager, namespace, config);
      } else {
        // 新規Deploymentを作成
        await k8sManager.createDeployment({
          ...config,
          namespace,
        });
      }
    } catch (error) {
      if (error.message.includes('not found')) {
        // Deploymentが存在しない場合は新規作成
        await k8sManager.createDeployment({
          ...config,
          namespace,
        });
      } else {
        throw error;
      }
    }
  }

  private async updateExistingDeployment(
    k8sManager: KubernetesManager, 
    namespace: string, 
    config: any
  ): Promise<void> {
    // Deploymentのイメージを更新（kubectl set image相当）
    // 実際の実装ではpatchNamespacedDeploymentを使用
    logger.info(`Updating deployment image: ${config.image}`);
  }

  private async waitForDeploymentReady(
    k8sManager: KubernetesManager, 
    namespace: string, 
    appName: string, 
    timeoutMinutes = 10
  ): Promise<void> {
    const startTime = Date.now();
    const timeout = timeoutMinutes * 60 * 1000;

    while (Date.now() - startTime < timeout) {
      try {
        const status = await k8sManager.getDeploymentStatus(appName, namespace);
        
        if (status.readyReplicas === status.replicas && status.replicas > 0) {
          logger.info(`Deployment ${appName} is ready`);
          return;
        }

        logger.info(`Waiting for deployment... ${status.readyReplicas}/${status.replicas} ready`);
        await new Promise(resolve => setTimeout(resolve, 10000)); // 10秒待機

      } catch (error) {
        logger.error('Error checking deployment status:', error);
        await new Promise(resolve => setTimeout(resolve, 5000));
      }
    }

    throw new Error(`Deployment ${appName} did not become ready within ${timeoutMinutes} minutes`);
  }

  private async performRollback(
    k8sManager: KubernetesManager, 
    namespace: string, 
    appName: string
  ): Promise<void> {
    // kubectl rollout undo相当の処理
    // 実際の実装では適切なAPIを呼び出し
    logger.info(`Performing rollback for ${appName}`);
  }

  private async getAccessToken(): Promise<string> {
    // 実際の実装では適切な認証方法を使用
    // gcloud auth print-access-token 相当
    return 'access-token';
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
    winston.format.json()
  ),
  defaultMeta: { service: 'gke-typescript-manager' },
  transports: [
    new winston.transports.File({ 
      filename: 'logs/error.log', 
      level: 'error' 
    }),
    new winston.transports.File({ 
      filename: 'logs/combined.log' 
    }),
  ],
});

// 開発環境では console にも出力
if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.combine(
      winston.format.colorize(),
      winston.format.simple()
    )
  }));
}

export { logger };
```

**📋 設定管理**

```typescript:src/config/index.ts
import dotenv from 'dotenv';

dotenv.config();

export const config = {
  // Google Cloud設定
  projectId: process.env.GOOGLE_CLOUD_PROJECT || '',
  defaultLocation: process.env.DEFAULT_LOCATION || 'asia-northeast1',
  defaultZone: process.env.DEFAULT_ZONE || 'asia-northeast1-a',
  
  // クラスター設定
  defaultKubernetesVersion: process.env.K8S_VERSION || 'latest',
  defaultMachineType: process.env.DEFAULT_MACHINE_TYPE || 'e2-medium',
  defaultDiskSize: parseInt(process.env.DEFAULT_DISK_SIZE || '100'),
  
  // 監視・ログ設定
  logLevel: process.env.LOG_LEVEL || 'info',
  metricsRetentionHours: parseInt(process.env.METRICS_RETENTION_HOURS || '24'),
  
  // CI/CD設定
  enableAutoDeploy: process.env.ENABLE_AUTO_DEPLOY === 'true',
  deploymentTimeout: parseInt(process.env.DEPLOYMENT_TIMEOUT || '600'), // 10分
  
  // セキュリティ設定
  enableNetworkPolicy: process.env.ENABLE_NETWORK_POLICY === 'true',
  enablePodSecurityPolicy: process.env.ENABLE_POD_SECURITY_POLICY === 'true',
};

// 必須設定の検証
if (!config.projectId) {
  throw new Error('GOOGLE_CLOUD_PROJECT environment variable is required');
}

// 設定値の妥当性チェック
if (config.defaultDiskSize < 10) {
  throw new Error('DEFAULT_DISK_SIZE must be at least 10GB');
}
```

**🎯 実用的な運用スクリプト**

```typescript:src/scripts/clusterOperations.ts
import { GKEClusterManager, KubernetesManager, GKEMonitoringService } from '../services';
import { config } from '../config';
import { logger } from '../utils/logger';

// クラスター一括作成スクリプト
export async function createEnvironmentClusters(): Promise<void> {
  const gkeManager = new GKEClusterManager(config.projectId);
  
  const environments = [
    {
      name: 'dev-cluster',
      location: config.defaultLocation,
      nodeCount: 2,
      machineType: 'e2-medium',
      enableAutopilot: true,
    },
    {
      name: 'staging-cluster',
      location: config.defaultLocation,
      nodeCount: 3,
      machineType: 'e2-standard-2',
      enableAutopilot: false,
    },
    {
      name: 'prod-cluster',
      location: config.defaultLocation,
      nodeCount: 5,
      machineType: 'e2-standard-4',
      enableAutopilot: false,
    },
  ];

  for (const env of environments) {
    try {
      logger.info(`Creating cluster: ${env.name}`);
      await gkeManager.createCluster({
        ...env,
        zone: config.defaultZone,
        diskSize: config.defaultDiskSize,
      });
      logger.info(`Successfully created cluster: ${env.name}`);
    } catch (error) {
      logger.error(`Failed to create cluster ${env.name}:`, error);
    }
  }
}

// クラスター状態監視スクリプト
export async function monitorClusters(): Promise<void> {
  const gkeManager = new GKEClusterManager(config.projectId);
  const monitoringService = new GKEMonitoringService(config.projectId);

  try {
    const clusters = await gkeManager.listClusters(config.defaultLocation);
    
    for (const cluster of clusters) {
      logger.info(`Monitoring cluster: ${cluster.name}`);
      
      // 基本状態の確認
      logger.info(`Status: ${cluster.status}, Nodes: ${cluster.nodeCount}`);
      
      // メトリクス取得
      try {
        const metrics = await monitoringService.getClusterMetrics(
          cluster.name, 
          cluster.location, 
          1 // 過去1時間
        );
        
        if (metrics.length > 0) {
          const latestMetric = metrics[0];
          logger.info(`CPU: ${latestMetric.cpuUtilization}%, Memory: ${latestMetric.memoryUtilization}%`);
        }
      } catch (error) {
        logger.warn(`Failed to get metrics for cluster ${cluster.name}:`, error.message);
      }
    }
  } catch (error) {
    logger.error('Failed to monitor clusters:', error);
  }
}

// 使用例
if (require.main === module) {
  const operation = process.argv[2];
  
  switch (operation) {
    case 'create':
      createEnvironmentClusters();
      break;
    case 'monitor':
      monitorClusters();
      break;
    default:
      console.log('Usage: npm run clusters [create|monitor]');
  }
}
```

## まとめ

Google Kubernetes Engine × TypeScript の組み合わせで、コンテナオーケストレーションの複雑さから解放された効率的なクラスター管理が実現できました。

**✅ この記事で身についたスキル**

>* GKE クラスターの作成・管理とライフサイクル制御
>* TypeScript + Kubernetes API での実用的なアプリケーションデプロイ
>* Node Pool とオートスケーリングの設定・運用
>* 監視・ログ管理・CI/CD統合の実装パターン
>* Infrastructure as Code による再現可能なクラスター構築

**🎯 次のステップ**

>* Istio Service Mesh との連携でマイクロサービス通信の可視化・制御
>* ArgoCD や Flux を使った GitOps ワークフロー構築
>* Knative を活用したサーバーレスコンテナ実行環境の構築

GKE なら、従来のKubernetes運用の煩わしさを一切感じることなく、Google品質のマネージドサービスで本格的なコンテナアプリケーションを運用できます。あなたも今日からプログラマブルなKubernetes管理を始めてみませんか？