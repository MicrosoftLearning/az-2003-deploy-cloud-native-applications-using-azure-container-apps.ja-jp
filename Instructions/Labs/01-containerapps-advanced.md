---
lab:
  title: Azure Container Apps をデプロイして操作する
  description: Bicep を使用して Azure Container Apps インフラストラクチャをデプロイし、コンテナー イメージをビルドし、Azure DevOps で自動スケーリング、トラフィック分割、バッチ処理ジョブ、CI/CD を実装する
  level: 400
  duration: 150 minutes
  islab: true
  primarytopics:
    - Azure
    - Azure Container Apps
    - Azure DevOps
---

# Azure Container Apps をデプロイして操作する

この演習では、Azure Container Apps 上に構築されたイベントドリブン マイクロサービス アプリケーションである **Contoso Analytics** プラットフォームをデプロイして操作します。 インフラストラクチャのデプロイ、コンテナー イメージ管理、HTTP 自動スケーリング、ブルーグリーン デプロイのためのトラフィック分割、バッチ処理用の Container Apps ジョブを操作します。 すべてが稼働したら、Azure DevOps CI/CD パイプラインを使用して DevOps 戦略を統合します。

この演習の所要時間は約 **150** 分です。

---

## 目次

- [開始する前に](#before-you-start)
- [演習 1: ベースライン シナリオのデプロイ](#exercise-1-baseline-scenario-deployment)
  - [タスク 1: インフラストラクチャをデプロイする](#task-1-deploy-infrastructure)
- [演習 2: Azure Container Apps の管理と操作](#exercise-2-managing--operating-azure-container-apps)
  - [タスク 1: HTTP 自動スケーリングを実装する](#task-1-implement-http-auto-scaling)
  - [タスク 2: トラフィック分割を実装する](#task-2-implement-traffic-splitting)
  - [タスク 3: Container Apps ジョブを操作する](#task-3-work-with-container-apps-jobs)
- [演習 3: Azure DevOps CI/CD パイプライン](#exercise-3-azure-devops-cicd-pipelines)
  - [タスク 1: Azure DevOps パイプラインを使用してデプロイする](#task-1-deploy-with-azure-devops-pipelines)
  - [タスク 2: トラフィック分割を使用してパイプラインを拡張する (省略可能)](#task-2-enhance-the-pipeline-with-traffic-splitting-optional)

---

## 開始する前に

演習を開始する前に、以下が必要になります。

1. **共同作成者**のアクセス権を持つ Azure サブスクリプション
1. **[Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli?view=azure-cli-latest)** バージョン 2.50 以降がインストールされていること
1. **[Azure Developer CLI - AZD](https://learn.microsoft.com/azure/developer/azure-developer-cli/install-azd)** がインストールされていること (オプション B のデプロイ選択肢用)
1. **[Docker Desktop](https://www.docker.com/products/docker-desktop/)** がインストールされ、実行されていること
1. **共同作成者**のアクセス権を持つ **Azure DevOps** 組織とプロジェクト

1. ツールの準備ができていることを確認します。

    ```powershell
    # Check Azure CLI version (requires 2.50+)
    az version
    ```

    ```powershell
    # Check Docker is running; if it fails, start Docker Desktop and wait for the GUI to inform you it is ready to run containers
    docker info
    ```

    ```powershell
    # Login to Azure
    az login
    ```

    ```powershell
    # Set your subscription (replace with your subscription ID or name)
    az account set --subscription "<YOUR_SUBSCRIPTION_ID>"
    ```

    ```powershell
    # Verify current subscription
    az account show --query "{Name:name, SubscriptionId:id}" -o table

    ```

1. リポジトリを クローンする:

    ```powershell
    # Clone the repository
    git clone https://github.com/MicrosoftLearning/az-2003-deploy-cloud-native-applications-using-azure-container-apps.git
    ```

    ```powershell
    # Navigate to the project directory
    cd .\az-2003-deploy-cloud-native-applications-using-azure-container-apps

    ```

1. 演習全体を通して使用する環境変数を設定します。

    ```powershell
    # Set your environment name (use lowercase, no special characters)
    $ENV_NAME = "contapp"  # Example: "john-lab"
    ```
> **注**: `contapp` は一意の名前に置き換えます。 これを使用してグローバルに一意のリソース名が作成されます。

    ```powershell
    # Set the Azure region
    $LOCATION = "eastus2" # Choose your Azure region
    ```

    ```powershell
    # Get your user principal ID (for Key Vault access)
    $PRINCIPAL_ID = (az ad signed-in-user show --query id -o tsv)
    ```

    ```powershell
    # Verify the principal ID was retrieved
    Write-Host "Principal ID: $PRINCIPAL_ID"

    ```
---

## 演習 1: ベースライン シナリオのデプロイ

この演習では、Contoso Analytics プラットフォームの基本インフラストラクチャをデプロイします。 Azure CLI と Bicep テンプレート、または Azure Developer CLI (azd) を使用して、Azure Container Apps 環境、Container Registry、Key Vault、サポート リソースをプロビジョニングします。 この演習を終了すると、アプリケーションのデプロイ用に、完全に機能するコンテナー プラットフォームが準備できます。

### タスク 1: インフラストラクチャをデプロイする

このタスクでは、Azure Container Apps インフラストラクチャをデプロイします。 次の 2 つのデプロイ方法から選択できます。

| Method | 説明 | 最適な用途 |
|--------|-------------|----------|
| **オプション A: Azure CLI + Bicep** | Azure CLI と Bicep テンプレートを使用して、ステップバイステップでインフラストラクチャをデプロイする | Bicep の学習、きめ細かい制御、各コンポーネントの理解 |
| **オプション B: Azure Developer CLI (azd)** | インフラストラクチャとアプリケーション コードを 1 つのコマンドでデプロイする | クイック セットアップ、CI/CD ワークフロー、運用シナリオ |

> **重要**: 以下の**デプロイ オプションの 1 つ**を選択します。 コンテナーのビルド、デプロイ、管理といった Docker と Container Registry のタスクにあまり慣れていない場合は、すべてのステップを手動で進める**オプション A (az cli)** を使用することをお勧めします。 Docker、Azure Container Registry、Azure Bicep のデプロイに既に慣れている場合は、**オプション B (azd)** を選択してください。azd によってコンテナー イメージが自動的にビルドされてデプロイされます。

---

### オプション A: Azure CLI と Bicep を使用してデプロイする

このオプションは、Bicep テンプレートを理解し、ステップバイステップでインフラストラクチャをデプロイしたい場合に使用します。

#### Bicep テンプレートを確認する

1. デプロイする前に、Bicep のメイン テンプレートを調べて、どのようなリソースが作成されるかを理解します。

    ```powershell
    # Open the main Bicep file to review
    Get-Content .\infra\main.bicep | Select-Object -First 80

    ```

1. テンプレートによって作成されるリソースに注意してください。
    - リソース グループ
    - Log Analytics ワークスペース
    - ユーザー割り当てマネージド ID
    - Azure Container Registry
    - Container Apps 環境
    - Azure Service Bus 名前空間
    - Azure Key Vault
    - コンテナー アプリ (プレースホルダー イメージを使用)
    - Container Apps ジョブ (プレースホルダー イメージを使用)

#### インフラストラクチャをデプロイする

1. Azure CLI を使用して Bicep テンプレートをデプロイします。

    ```powershell
    # Deploy infrastructure at subscription scope
    az deployment sub create `
        --name "contapp-$ENV_NAME" `
        --location $LOCATION `
        --template-file .\infra\main.bicep `
        --parameters environmentName=$ENV_NAME `
        --parameters location=$LOCATION `
        --parameters principalId=$PRINCIPAL_ID `
        --query "properties.outputs" -o json

    ```

    > **注**: このデプロイが完了するまで、5 - 8 分ほどかかります。

#### デプロイの出力を取得する

1. デプロイが完了したら、後で使用するために出力値を取得します。

    ```powershell
    # Get deployment outputs
    $OUTPUTS = az deployment sub show `
        --name "contapp-$ENV_NAME" `
        --query "properties.outputs" -o json | ConvertFrom-Json
    ```

    ```powershell
    # Extract key values
    $RG = $OUTPUTS.AZURE_RESOURCE_GROUP.value
    $ACR = $OUTPUTS.CONTAINER_REGISTRY_LOGIN_SERVER.value
    $ACR_NAME = $OUTPUTS.CONTAINER_REGISTRY_NAME.value
    $CAE_NAME = $OUTPUTS.CONTAINER_APPS_ENVIRONMENT_NAME.value
    $DASHBOARD_URL = $OUTPUTS.DASHBOARD_URL.value
    $INGESTION_URL = $OUTPUTS.INGESTION_SERVICE_URL.value
    $HELLO_API_URL = $OUTPUTS.HELLO_API_URL.value

    # Display captured values
    Write-Host "========================================="
    Write-Host "Resource Group:        $RG"
    Write-Host "Container Registry:    $ACR"
    Write-Host "Environment:           $CAE_NAME"
    Write-Host "Dashboard URL:         $DASHBOARD_URL"
    Write-Host "Ingestion Service URL: $INGESTION_URL"
    Write-Host "Hello API URL:         $HELLO_API_URL"
    Write-Host "========================================="

    ```

#### 初期デプロイを検証する

1. すべてのリソースが作成されたことを確認します。

    ```powershell
    # List all resources in the resource group
    az resource list -g $RG --query "[].{Name:name, Type:type}" -o table
    ```
    ```powershell
    # List Container Apps
    az containerapp list -g $RG --query "[].{Name:name, URL:properties.configuration.ingress.fqdn}" -o table
    ```
    ```powershell
    # List Container Apps Jobs
    az containerapp job list -g $RG --query "[].{Name:name, TriggerType:properties.configuration.triggerType}" -o table

    ```

    > **注**: ingestion-service、hello-api、dashboard という 3 つのコンテナー アプリが表示されるはずです。 data-processor-parallel、-scheduled、-manual という 3 つの Container Apps ジョブが表示されるはずです

    > **注**: この時点で、コンテナー アプリはプレースホルダー イメージ (`containerapps-helloworld`) を使用して実行されています。 以下の「**コンテナー イメージをビルドする**」セクションに進みます。

#### コンテナー イメージをビルドする

このセクションでは、カスタム コンテナー イメージをビルドして Azure Container Registry にプッシュします。

1. コンテナー レジストリに対して認証します。

    ```powershell
    # Login to ACR
    az acr login --name $ACR_NAME

    ```

1. インジェスト サービス イメージをビルドしてプッシュします。

    ```powershell
    # Navigate to the ingestion-service directory
    cd src/ingestion-service
    ```
    ```powershell
    # Build the container image
    docker build -t "$ACR/ingestion-service:v1" .
    ```
    ```powershell
    # Push to Azure Container Registry
    docker push "$ACR/ingestion-service:v1"
    ```
    ```powershell
    # Return to project root
    cd ../..

    ```

1. ダッシュボード イメージをビルドしてプッシュします。

    ```powershell
    # Navigate to the dashboard directory
    cd src/dashboard
    ```
    ```powershell
    # Build the container image
    docker build -t "$ACR/dashboard:v1" .
    ```
    ```powershell
    # Push to Azure Container Registry
    docker push "$ACR/dashboard:v1"
    ```
    ```powershell
    # Return to project root
    cd ../..

    ```

1. Hello API イメージをビルドしてプッシュします。

    ```powershell
    # Navigate to the hello-api directory
    cd src/hello-api
    ```
    ```powershell
    # Build v1 of the Hello API (blue version)
    docker build -t "$ACR/hello-api:v1" --build-arg APP_VERSION=v1 .
    ```
    ```powershell
    # Push to Azure Container Registry
    docker push "$ACR/hello-api:v1"
    ```
    ```powershell
    # Return to project root
    cd ../..

    ```

1. デモ ジョブ イメージをビルドしてプッシュします。

    ```powershell
    # Navigate to the demo-job directory
    cd src/demo-job
    ```
    ```powershell
    # Build the job image
    docker build -t "$ACR/demo-job:v1" .
    ```
    ```powershell
    # Push to Azure Container Registry
    docker push "$ACR/demo-job:v1"
    ```
    ```powershell
    # Return to project root
    cd ../..

    ```

1. すべてのイメージがレジストリで使用可能であることを確認します。

    ```powershell
    # List all repositories in ACR
    az acr repository list --name $ACR_NAME -o table
    ```
    ```powershell
    # List tags for each repository
    az acr repository show-tags --name $ACR_NAME --repository ingestion-service -o table
    az acr repository show-tags --name $ACR_NAME --repository dashboard -o table
    az acr repository show-tags --name $ACR_NAME --repository hello-api -o table
    az acr repository show-tags --name $ACR_NAME --repository demo-job -o table

    ```

#### コンテナー イメージを Container Apps にデプロイする

次に、デプロイされたコンテナー アプリを更新して、カスタム イメージを使用するようにします。

1. カスタム イメージを使用して ingestion-service を更新します。

    ```powershell
    # Update the ingestion-service with the custom image
    az containerapp update `
        --name ingestion-service `
        --resource-group $RG `
        --image "$ACR/ingestion-service:v1"

    # Verify the update
    az containerapp show -n ingestion-service -g $RG `
        --query "{Name:name, Image:properties.template.containers[0].image, Status:properties.runningStatus}" -o table

    ```

1. カスタム イメージを使用して dashboard を更新します。

    ```powershell
    # Update the dashboard with the custom image
    az containerapp update `
        --name dashboard `
        --resource-group $RG `
        --image "$ACR/dashboard:v1"
    ```
    ```powershell
    # Verify the update
    az containerapp show -n dashboard -g $RG `
        --query "{Name:name, Image:properties.template.containers[0].image, Status:properties.runningStatus}" -o table

    ```

1. カスタム イメージを使用して hello-api を更新します。

    ```powershell
    # Update the hello-api with the custom image
    az containerapp update `
        --name hello-api `
        --resource-group $RG `
        --image "$ACR/hello-api:v1"
    ```
    ```powershell
    # Verify the update
    az containerapp show -n hello-api -g $RG `
        --query "{Name:name, Image:properties.template.containers[0].image, Status:properties.runningStatus}" -o table

    ```

1. デモ ジョブ イメージを使用して 3 つのジョブをすべて更新します。

    ```powershell
    # Update all three jobs with the demo-job image
    az containerapp job update -n data-processor-scheduled -g $RG --image "$ACR/demo-job:v1"
    az containerapp job update -n data-processor-manual -g $RG --image "$ACR/demo-job:v1"
    az containerapp job update -n data-processor-parallel -g $RG --image "$ACR/demo-job:v1"
    ```
    ```powershell
    # Verify the updates
    az containerapp job list -g $RG `
        --query "[].{Name:name, Image:properties.template.containers[0].image}" -o table

    ```

1. すべてのサービスが動作することを確認します。

    ```powershell
    # Check all Container Apps are running
    az containerapp list -g $RG `
        --query "[].{Name:name, Status:properties.runningStatus, Replicas:properties.template.scale.minReplicas}" -o table
    ```
    ```powershell
    # Test the endpoints
    Write-Host "`nTesting Dashboard..."
    Invoke-RestMethod -Uri "$DASHBOARD_URL/health" -TimeoutSec 30

    Write-Host "`nTesting Ingestion Service..."
    Invoke-RestMethod -Uri "$INGESTION_URL/health" -TimeoutSec 30

    Write-Host "`nTesting Hello API..."
    Invoke-RestMethod -Uri "$HELLO_API_URL/api/version" -TimeoutSec 30

    ```

    > **注**: Azure CLI と Bicep を使用して手動デプロイを完了しました。 **演習 2「タスク 1: HTTP 自動スケーリングを実装する」** に進みます。

---

### オプション B: Azure Developer CLI を使用してデプロイする (azd)

このオプションを使用すると、**すべてを自動的に**処理する、1 つのコマンドによる効率化されたデプロイが可能になります。

- Bicep テンプレートを使用してすべての Azure インフラストラクチャをプロビジョニングする
- Docker を使用してすべてのコンテナー イメージをローカルでビルドする
- Azure Container Registry にイメージをプッシュする
- すべてのサービスを Container Apps にデプロイする
- Container Apps ジョブを構成する

> **時間の節約**: このオプションを選択した場合は、デプロイが完了したら**演習 2「タスク 1: HTTP 自動スケーリングを実装する」** に直接進むことができます。

#### azd の前提条件

1. Azure Developer CLI がインストールされていることを確認します。

    ```powershell
    # Check if azd is installed
    azd version
    ```
    ```powershell
    # If not installed, install it (Windows winget)
    winget install microsoft.azd
    ```

    > **注**: Windows 以外を使っている場合や、Winget ではなく PowerShell を使う場合は、次のコマンドを使用します。 
    ```powershell
    powershell -ex AllSigned -c "Invoke-RestMethod 'https://aka.ms/install-azd.ps1' | Invoke-Expression"
    ```

#### azd を使用して初期化とデプロイを行う

1. azd を使用して Azure にログインします。

    ```powershell
    # Login to Azure
    azd auth login

    ```

1. 新しい azd 環境を初期化します。

    ```powershell
    # Start from the cloned repo folder
    cd .\az-2003-deploy-cloud-native-applications-using-azure-container-apps

    ```
    ```powershell
    # Create a new environment (use the same name as ENV_NAME)
    azd env new $ENV_NAME

    ```
    ```powershell
    # Set the Azure location
    azd env set AZURE_LOCATION $LOCATION

    ```

1. 1 つのコマンドを使用してすべてをデプロイします。

    ```powershell
    # Deploy infrastructure AND application code
    azd up

    ```

    > **注**: このコマンドで以下のことが行われます。
    > - Bicep テンプレートを使用してすべての Azure インフラストラクチャをプロビジョニングする
    > - Docker を使用してすべてのコンテナー イメージをローカルでビルドする
    > - Azure Container Registry にイメージをプッシュする
    > - すべてのサービスを Container Apps にデプロイする
    > - Container Apps ジョブを構成する
    >
    > プロセス全体で 10 - 15 分ほどかかります。

1. プロンプトが表示されたら、次を実行します。
    - お使いの "Azure サブスクリプション" を選択します
    - 場所を確認します (または、Enter キーを押して既定値をそのまま使用します)

#### azd でのデプロイの出力を取得する

1. デプロイが完了したら、環境変数を取得します。

    ```powershell
    # Get all environment values from azd and set them as PowerShell variables
    azd env get-values | ForEach-Object {
        if ($_ -match '^([^=]+)=(.*)$') {
            Set-Variable -Name $matches[1] -Value ($matches[2] -replace '^"|"$', '')
        }
    }
    ```
    ```powershell
    # The variables are now available directly (matching Bicep output names)
    $RG = $AZURE_RESOURCE_GROUP
    $ACR = $CONTAINER_REGISTRY_LOGIN_SERVER
    $ACR_NAME = $CONTAINER_REGISTRY_NAME
    $CAE_NAME = $CONTAINER_APPS_ENVIRONMENT_NAME
    ```
    ```powershell
    # Set aliases to match Option A variable names (for consistency in later tasks)
    $INGESTION_URL = $INGESTION_SERVICE_URL
    ```
    ```powershell
    # Display captured values
    Write-Host "========================================="
    Write-Host "Resource Group:        $RG"
    Write-Host "Container Registry:    $ACR"
    Write-Host "Environment:           $CAE_NAME"
    Write-Host "Dashboard URL:         $DASHBOARD_URL"
    Write-Host "Ingestion Service URL: $INGESTION_URL"
    Write-Host "Hello API URL:         $HELLO_API_URL"
    Write-Host "========================================="

    ```

#### azd でのデプロイを検証する

1. デプロイされたリソースを確認します。

    ```powershell
    # List all Container Apps (should show your custom images, not placeholders)
    az containerapp list -g $RG --query "[].{Name:name, Image:properties.template.containers[0].image}" -o table
    ```
    ```powershell
    # List Container Apps Jobs
    az containerapp job list -g $RG --query "[].{Name:name, TriggerType:properties.configuration.triggerType}" -o table
    ```
    ```powershell
    # Test the endpoints
    Write-Host "`nTesting Dashboard..."
    Invoke-RestMethod -Uri "$DASHBOARD_URL/health" -TimeoutSec 30

    Write-Host "`nTesting Ingestion Service..."
    Invoke-RestMethod -Uri "$INGESTION_URL/health" -TimeoutSec 30

    ```

    > **注**: azd では実際のアプリケーション イメージをデプロイしたので、**演習 2「タスク 1: HTTP 自動スケーリングを実装する」** に直接進みます。

---

## 演習 2: Azure Container Apps の管理と操作

この演習では、Azure Container Apps の運用機能を使って操作を行います。 変動するトラフィックの負荷を処理するために HTTP ベースの自動スケーリングを実装し、ブルーグリーン デプロイと A/B テスト用にトラフィック分割を構成し、バッチ処理ワークロード用の Container Apps ジョブを操作します。 これらのタスクは、コンテナ化された運用アプリケーションを管理するための実際のシナリオを示しています。

### タスク 1: HTTP 自動スケーリングを実装する

このタスクでは、Azure Container Apps で HTTP トラフィックに基づいて ingestion-service を自動的にスケーリングする方法を確認します。

### 初期レプリカ数を確認する

1. レプリカの現在の数を確認します。

    ```powershell
    # Check the current number of replicas
    az containerapp replica list -n ingestion-service -g $RG -o table
    ```

    > **注**: このコマンドの出力には 1 つのレプリカが表示されます (minReplicas が 1 に設定されています)

### スケーリング構成を確認する

1. ingestion-service 用に構成されたスケーリング ルールを表示します。

    ```powershell
    # View the scaling rules configured for the ingestion-service
    az containerapp show -n ingestion-service -g $RG `
        --query "properties.template.scale" -o json

    ```

    > **注**: `concurrentRequests` は `2` に設定されています。つまり、レプリカあたり 2 つを超える同時要求がある場合、サービスはスケールアップされます。

### dashboard コンテナー アプリを開く

1. ブラウザーで dashboard コンテナー アプリを開きます。

    ```powershell
    # Display the dashboard URL
    Write-Host "Open this URL in your browser: $DASHBOARD_URL"
    ```
    ```powershell
    # Or open directly (Windows)
    Start-Process $DASHBOARD_URL

    ```

### 負荷をトリガーしてスケーリングを観察する

1. dashboard で、**🔥 [Send 100 Events (Heavy Load)]** ボタンをクリックします
1. イベントが送信中であることを示すステータス メッセージを確認します
1. ロード テストの実行中、レプリカの数を監視します。

    ```powershell
    # Run this command multiple times during the load test to see scaling
    az containerapp replica list -n ingestion-service -g $RG -o table
    ```
    ```powershell
    # Or watch continuously (run in a separate terminal)
    while ($true) {
        $count = (az containerapp replica list -n ingestion-service -g $RG -o json | ConvertFrom-Json).Count
        Write-Host "$(Get-Date -Format 'HH:mm:ss') - Replica count: $count"
        Start-Sleep -Seconds 5
    }

    ```

### スケールダウンを観察する

1. ロード テストが完了したら、クールダウン期間 (既定では 5 分) が過ぎるのを待ち、レプリカがスケールダウンされて元に戻るのを確認します。

    ```powershell
    # Check replica count after load subsides
    az containerapp replica list -n ingestion-service -g $RG -o table

    ```

### Azure portal でスケーリング メトリックを表示する

1. [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) に移動します
1. **リソース グループ**に移動し、**ingestion-service** を選択します
1. **[メトリック]** をクリックします
1. **Replica Count** というメトリックを追加します
1. 時間の経過に伴うスケーリングの動作を観察します

    > **注**: ingestion-service が負荷時に 1 つのレプリカから複数のレプリカにスケールアップした後、クールダウン期間を経て元に戻ることが観察されたはずです。

### タスク 2: トラフィック分割を実装する

このタスクでは、2 番目のバージョンの Hello API をデプロイし、ブルーグリーン デプロイ用にトラフィック分割を実装します。

### バージョン 1 が実行されていることを確認する

1. 現在のバージョンがデプロイされていることを確認します。

    ```powershell
    # Open Hello API in browser
    Start-Process $HELLO_API_URL
    ```
    ```powershell
    # Or test via CLI
    Invoke-RestMethod -Uri "$HELLO_API_URL/api/version"

    ```

1. ページに **blue "v1"** バッジが表示されます。

### 複数リビジョン モードを有効にする

1. バージョン間でトラフィックを分割するために複数リビジョン モードを有効にします。

    ```powershell
    # Enable multiple revision mode
    az containerapp revision set-mode `
        --name hello-api `
        --resource-group $RG `
        --mode Multiple
    ```
    ```powershell
    # Verify the mode change
    az containerapp show -n hello-api -g $RG `
        --query "properties.configuration.activeRevisionsMode" -o tsv

    ```

### バージョン 2 をビルドしてプッシュする

1. Hello API のバージョン 2 をビルドしてプッシュします。

    ```powershell
    # Navigate to the hello-api directory
    cd src/hello-api
    ```
    ```powershell
    # Build v2 (green version)
    docker build -t "$ACR/hello-api:v2" --build-arg APP_VERSION=v2 .
    ```
    ```powershell
    # Push to ACR
    docker push "$ACR/hello-api:v2"
    ```
    ```powershell
    # Return to project root
    cd ../..

    ```

### バージョン 2 を新しいリビジョンとしてデプロイする

1. カスタム サフィックスを持つ新しいリビジョンとして v2 をデプロイします。

    ```powershell
    # Deploy v2 as a new revision with a custom suffix
    az containerapp update `
        --name hello-api `
        --resource-group $RG `
        --image "$ACR/hello-api:v2" `
        --revision-suffix v2 `
        --set-env-vars "APP_VERSION=v2"

    ```

### すべてのリビジョンを一覧表示する

1. 使用可能なすべてのリビジョンを表示します。

    ```powershell
    # List all revisions of hello-api
    az containerapp revision list -n hello-api -g $RG `
        --query "[].{Name:name, Active:properties.active, TrafficWeight:properties.trafficWeight, Created:properties.createdTime}" -o table

    ```

    > **注**: この時点で、トラフィックの 100% が最新のリビジョン (v2) に送信されます。

### トラフィックを 50/50 で分割してテストする

Azure portal から、バージョン間のトラフィック分割を構成します。

1. リソース グループに移動し、**hello-Api** コンテナー アプリを選択します
1. **[アプリケーション]** / **[リビジョンとレプリカ]** を選択します
1. **hello-api--v2** と **hello-api--00000** という 2 つのリビジョンがあります
1. 現在、**hello-api--v2** のトラフィック負荷は **100%** です。 これを **50%** に**更新**します
1. もう 1 つのリビジョンについて、同じ **50%** の重みに**更新**します
1. 変更を**保存**します
1. **hello-api** コンテナー アプリの **[概要]** セクションに戻り、**[アプリケーション URL]** を**選択**します
1. ブラウザーを何回か**更新**します。ブルーとグリーンのバージョンが交互に表示されます  

### 移行を完了する

1. トラフィックの 100% を v2 に移行します。

    ```powershell
    # Route all traffic to v2
    az containerapp ingress traffic set `
        --name hello-api `
        --resource-group $RG `
        --revision-weight "hello-api--v2=100"
    ```
    ```powershell
    # Verify
    az containerapp ingress traffic show -n hello-api -g $RG -o table

    ```

### (省略可能) v1 にロールバックする

1. 必要に応じて、すぐにロールバックできます。

    ```powershell
    # Rollback to v1
    az containerapp ingress traffic set `
        --name hello-api `
        --resource-group $RG `
        --revision-weight "$REV_V1=100"

    ```

    > **注**: トラフィック分割を使用してブルーグリーン デプロイを正常に実装しました。これにより、バージョン間でユーザーを徐々に移行できます。

### タスク 3: Container Apps ジョブを操作する

このタスクでは、バッチ処理タスクのための Container Apps ジョブを操作します。

### すべてのジョブを一覧表示する

1. すべての Container Apps ジョブを表示します。

    ```powershell
    # List all Container Apps Jobs
    az containerapp job list -g $RG `
        --query "[].{Name:name, TriggerType:properties.configuration.triggerType, Schedule:properties.configuration.scheduleTriggerConfig.cronExpression}" -o table

    ```

1. 3 つのジョブが表示されます。

    | Name | トリガーの種類 | スケジュール |
    |------|--------------|----------|
    | data-processor-scheduled | スケジュール | */2 * * * * |
    | data-processor-manual | 手動 | - |
    | data-processor-parallel | 手動 | - |

### スケジュールされたジョブの実行を表示する

1. スケジュールされたジョブは 2 分ごとに実行されます。 実行履歴を確認します。

    ```powershell
    # List executions for the scheduled job
    az containerapp job execution list `
        --name data-processor-scheduled `
        --resource-group $RG `
        --query "[].{Name:name, Status:properties.status, StartTime:properties.startTime}" -o table

    ```

    > **注**: 実行が表示されない場合は、スケジュールされた最初の実行まで 2 分待ちます。 一部のジョブは成功と表示され、他のジョブは失敗と表示される可能性があります。

### Azure portal でジョブのログを表示する

1. リソース グループから、**data-processor-scheduled** コンテナー アプリを選択します
1. **[概要]** セクションで、**[実行履歴]** の **[表示]** リンクを**選択**します
1. 約 2 分ごとに更新すると、状態が **[実行中]** の新しいジョブが表示されます
1. いずれかのジョブについて、**コンソールまたはシステム**のログのリンクを選択します
1. これにより、**Azure Log Analytics** の**より詳細な**ログ ビューがジョブの詳細とともに表示されます

> **注**: **[監視]** / **[実行履歴]** に移動して同じものを表示できます

### 手動ジョブをトリガーする

1. 手動ジョブを開始します。

    ```powershell
    # Start the manual job
    $JOB_EXECUTION = az containerapp job start `
        --name data-processor-manual `
        --resource-group $RG `
        --query "name" -o tsv
    ```
    ```powershell
    Write-Host "Started job execution: $JOB_EXECUTION"
    ```
    ```powershell
    # Wait for completion
    Write-Host "Waiting for job to complete..."
    Start-Sleep -Seconds 20
    ```
    ```powershell
    # Check execution status
    az containerapp job execution list `
        --name data-processor-manual `
        --resource-group $RG `
        --query "[0].{Name:name, Status:properties.status, StartTime:properties.startTime, EndTime:properties.endTime}" -o table

    ```

### 並列ジョブをトリガーする

1. 並列ジョブは、3 つのインスタンスを同時に実行します。

    ```powershell
    # Start the parallel job
    az containerapp job start `
        --name data-processor-parallel `
        --resource-group $RG
    ```
    ```powershell
    Write-Host "Started parallel job with 3 replicas"
    ```
    ```powershell
    # Wait and check status
    Start-Sleep -Seconds 25
    ```
    ```powershell
    # View execution details
    az containerapp job execution list `
        --name data-processor-parallel `
        --resource-group $RG `
        --query "[0].{Name:name, Status:properties.status}" -o table

    ```

### ポータルで並列実行を表示する

1. [Azure portal](https://portal.azure.com) (`https://portal.azure.com`) に移動します
1. **リソース グループ**に移動し、**data-processor-parallel** を選択します
1. **[実行履歴]** をクリックします
1. 最新の実行をクリックします
1. 3 つのレプリカが同時に実行されたことを確認します

    > **注**: スケジュールされたジョブ、手動トリガー、並列実行を含む Container Apps ジョブを正常に操作できました。

---

## 演習 3: Azure DevOps CI/CD パイプライン

この演習では、Azure DevOps を使用して継続的インテグレーションと継続的デプロイ (CI/CD) パイプラインを実装します。 コンテナー イメージのビルド、Azure Container Registry へのプッシュ、Azure Container Apps へのデプロイを行う、自動化されたパイプラインを作成します。 また、ダウンタイムなしのリリースのためにパイプラインで直接トラフィックを分割するなどの高度なデプロイ戦略を実装する方法も学習します。

### タスク 1: Azure DevOps パイプラインを使用してデプロイする

このタスクでは、コンテナー アプリのビルドとデプロイを自動化する Azure DevOps パイプラインを設定します。 これは、コンテナー化されたマイクロサービスの CI/CD を実装する方法を示しています。

### Azure DevOps の前提条件 (必要な場合)

1. Azure DevOps 組織にアクセスできることを確認します。 そうでない場合は、次のようにして作成します。
    - [Azure DevOps](https://aex.dev.azure.com) (`https://aex.dev.azure.com`) に移動します
    - Microsoft アカウントでサインインする
    - **[新しい組織]** をクリックします (まだない場合)

1. この演習用に新しいプロジェクトを作成します。

    ```powershell
    # Open Azure DevOps in your browser
    Start-Process "https://dev.azure.com"

    ```

1. Azure DevOps で、以下を行います。
    - **[+ 新しいプロジェクト]** をクリックします
    - プロジェクト名を入力します: `Contoso-ContainerApps`
    - 可視性を **[非公開]** に設定します
    - **[作成]** をクリックします。

### Azure サービス接続を作成する

1. サービス接続を作成して、Azure DevOps が Azure サブスクリプションにデプロイできるようにします。

    - Azure DevOps プロジェクトの **[プロジェクト設定]** (左下) に移動します
    - **[パイプライン]** で、**[サービス接続]** をクリックします
    - **[サービス コネクタの作成]** をクリックします
    - **[Azure Resource Manager]** を選択し、**[次へ]** をクリックします
    - 接続を構成します。
        - **ID の種類**: アプリの登録 (自動)
        - **Credential**:ワークロード ID フェデレーション
        - **[Scope level]\(スコープ レベル\)**:サブスクリプション
        - **[サブスクリプション]**: Azure サブスクリプションを選択します
        - **リソース グループ**: 空のままにします (サブスクリプション レベルのアクセス)
        - **サービス接続名**: `AzureServiceConnection`
    - **[保存]** をクリックします

    > **注**: サービス接続名は、パイプライン YAML ファイル内の `azureSubscription` 変数と一致している必要があります。

### Git リポジトリを初期化する

1. Git リポジトリを初期化し、コードを Azure DevOps にプッシュします。

    ```powershell
    # Navigate to your project directory
    cd c:\azd-contapp-demo-v2
    ```
    ```powershell
    # Initialize Git repository 
    git init
    ```
    ```powershell
    # Add all files
    git add .
    ```
    ```powershell
    # Create initial commit
    git commit -m "Initial commit: Contoso Analytics Container Apps"

    ```

### Azure DevOps をリモートとして追加し、プッシュする

1. Azure DevOps からリポジトリの URL を取得します。
    - Azure DevOps プロジェクトで、**[リポジトリ]** をクリックします
    - **[ファイル]** をクリックします
    - **クローン URL** (HTTPS) をコピーします

1. リモートを**更新**してプッシュします。

    ```powershell
    # Add Azure DevOps as remote (replace with your Azure DevOps Project and Repo URL)
    git remote set-url origin https://dev.azure.com/YOUR_ORG/YOUR_PROJECT/_git/YOUR_REPO
    ```
    ```powershell
    # Push to Azure DevOps
    git push -u origin main

    ```

    > **注**: 認証を求められる場合があります。 Azure DevOps 資格情報または個人用アクセス トークン (PAT) を使用します。

### パイプライン変数を更新する

1. パイプラインを実行する前に、**Azure DevOps セットアップ用の正しい YAML ファイル**内で、デプロイ変数をデプロイに合わせて更新します。

**Ubuntu ADO エージェント**:

```powershell
# Open the pipeline file
code .ado/azure-pipelines.yml
```

**Windows ADO エージェント**:

```powershell
# Open the pipeline file
code .ado/azure-pipelines-windows.yml
```

2. 次の変数を、正しい値を使って**変更**します。
    - **azureSubscription**: 前に設定した ADO サービス接続の名前 (default=AzureServiceConnection)
    - **resourceGroupName**: このプロジェクトに既に使用されている Azure リソースを含むリソース グループの名前
    - **ACR ログイン サーバー**: このプロジェクトで既に使用されている Azure Container Registry の FQDN
    - **ACR 名**: このプロジェクトで既に使用されている Azure コンテナー レジストリの短い名前
    - **Azure リージョン**: このプロジェクトに既に使用されている Azure リージョンの場所の名前

3. **[Azure DevOps エージェント プール](https://learn.microsoft.com/en-us/azure/devops/pipelines/agents/agents?view=azure-devops&tabs=yaml)のセットアップ**によっては、パイプライン プール設定構文を変更することも必要になる場合があります。

    - **セルフホステッド エージェントを使用する場合**

    ```yaml
    pool:
        name: $(agentPool)

    ```

    - **Azure ホステッド エージェントを使用する場合**

    ```yaml
    pool:
        vmImage: 'ubuntu-latest' #or 'windows-latest if you are on the azure-pipelines-windows.yml'

    ```

1. ファイルを保存し、変更をコミットします。

    ```powershell
    git add .
    git commit -m "Update environment name for pipeline"
    git push

    ```

### パイプラインを作成する

1. Azure DevOps で、以下を行います。
    - **[パイプライン]** > **[パイプライン]** に移動します
    - **[パイプラインの作成]** (または **[新しいパイプライン]**) をクリックします
    - **[Azure Repos Git]** を選択します
    - リポジトリの選択
    - **[既存の Azure Pipelines の YAML ファイル]** を選択します
    - セットアップに応じた正しいパイプラインを選択します。
        - **ブランチ**: main
        - **パス (path)**: 
            - `./ado/azure-pipelines.yml` (Ubuntu ADO エージェントの場合)
            - `./ado/azure-pipelines-windows.yml` (Windows ADO エージェントの場合)

1. **[実行]** をクリックして変更を保存し、パイプラインの実行を開始します

### パイプラインの承認

1. パイプラインの開始前に、**承認**を求められる場合があります。 これは **[環境]** を通じて管理され、DevOps チームが運用環境への実際のデプロイの制御を維持する方法をシミュレートします。 承認要求を**確認**します。

### パイプライン構造を確認する

1. パイプラインの実行中に、いくつかの詳細を確認します。 パイプラインには 3 つのステージがあります。

    | 段階 | 説明 |
    |-------|-------------|
    | **ビルドとテスト** | すべてのサービス (dashboard、ingestion-service、hello-api、demo-job) の Docker イメージをビルドし、Bicep テンプレートを検証します |
    | **イメージを ACR にプッシュ** | ビルドされたイメージを Azure Container Registry にプッシュします |
    | **Container Apps へのデプロイ** | すべてのコンテナー アプリとジョブを新しいイメージで更新します |

1. 主な機能は次のとおりです。
    - **並列ビルド**: すべてのサービスが同時にビルドされます
    - **成果物の公開**: Docker イメージがパイプライン成果物として保存されます
    - **動的出力**: リソース名が既存のデプロイから取得されます
    - **環境ゲート**: デプロイには承認が必要です (省略可能)

### パイプラインの進行状況を監視する

1. パイプライン ステージを監視します。
    - **ビルドとテスト**: 3 - 5 分で完了するはずです
    - **イメージを ACR にプッシュ**: 2 - 3 分で完了するはずです
    - **Container Apps へのデプロイ**: 2 - 3 分で完了するはずです

1. 任意のステージまたはジョブをクリックして詳細なログを表示します

### パイプライン実行履歴を表示する

1. パイプライン実行履歴を表示します。
    - **[パイプライン]** > **[パイプライン]** に移動します
    - パイプラインをクリックします
    - 状態、期間、トリガーの情報を含む実行の一覧が表示されます

    > **注**: コードがメイン ブランチにプッシュされたときにコンテナー アプリを自動的にビルドしてデプロイする CI/CD パイプラインが正常に設定されました。

### デプロイの検証

1. パイプラインが完了したら、デプロイを検証します。

- **コンテナー アプリが実行されていることを確認する**

    ```powershell
    # Check all Container Apps are running with the new images
    az containerapp list -g $RG `
        --query "[].{Name:name, Image:properties.template.containers[0].image}" -o table
    ```

- **Container Apps ジョブが実行されていることを確認する**

    ```powershell
    # Check the Jobs
    az containerapp job list -g $RG `
        --query "[].{Name:name, Image:properties.template.containers[0].image}" -o table

    ```

1. イメージ タグに Azure DevOps ビルド ID が含まれたことに注意してください

1. **Azure portal** から、さまざまな**コンテナー アプリ**間を移動し、ADO によってプッシュされたコンテナー イメージを反映した最新のリビジョンが **[アクティブなリビジョン]** として表示されることを確認します。 
1. **[非アクティブなリビジョン]** タブを選択すると、以前のものも引き続き**表示して使用する**ことができます。 
1. いずれかの非アクティブなリビジョン コンテナーの横にある **[アクティブ化]** を選択すると、そのバージョンがアクティブなリビジョンとして再び表示されます。
1. **[非アクティブなリビジョン]** タブから **[アクティブなリビジョン]** タブに戻ります。コンテナーが **[停止中]** 状態であることに注意してください。
1. 変更を**保存**して、切り替えたリビジョンのコンテナーを再び開始します。

### タスク 2: トラフィック分割を使用してパイプラインを拡張する (省略可能)

この省略可能なタスクではトラフィック分割を使用して、パイプラインを拡張してブルーグリーン デプロイを実装し、演習 2 のタスク 2 で手動で行ったことを自動化します。

### トラフィック分割パイプライン ステージを作成する

1. ブルーグリーン デプロイ用の新しいステージをパイプラインに追加します。 次の YAML をコピーし、`azure-pipelines.yml` の `DeployApps` ステージの後に追加します。

> **注**: ADO エージェント プールのセットアップに応じて、**pool** 構文を変更することを忘れないでください

    ```yaml
    # ============================================================================
    # Stage 4: Blue-Green Deployment (Optional - Manual Trigger)
    # ============================================================================
    - stage: BlueGreenDeploy
      displayName: 'Blue-Green Deployment'
      dependsOn: DeployApps
      condition: and(succeeded(), eq(variables['deployBlueGreen'], 'true'))
      variables:
        containerRegistry: $[ stageDependencies.PushImages.PushToACR.outputs['pushImages.containerRegistry'] ]
        resourceGroup: $[ stageDependencies.PushImages.PushToACR.outputs['pushImages.resourceGroup'] ]
      jobs:
        - deployment: BlueGreenHelloApi
          displayName: 'Blue-Green Hello API'
          pool:
            vmImage: 'ubuntu-latest'
          environment: 'production'
          strategy:
            runOnce:
              deploy:
                steps:
                  - task: AzureCLI@2
                    displayName: 'Enable Multiple Revision Mode'
                    inputs:
                      azureSubscription: $(azureSubscription)
                      scriptType: 'bash'
                      scriptLocation: 'inlineScript'
                      inlineScript: |
                        az containerapp revision set-mode \
                          --name hello-api \
                          --resource-group $(resourceGroup) \
                          --mode Multiple

                  - task: AzureCLI@2
                    displayName: 'Deploy New Revision'
                    inputs:
                      azureSubscription: $(azureSubscription)
                      scriptType: 'bash'
                      scriptLocation: 'inlineScript'
                      inlineScript: |
                        # Deploy new revision with v2 suffix
                        az containerapp update \
                          --name hello-api \
                          --resource-group $(resourceGroup) \
                          --image $(containerRegistry)/hello-api:$(imageTag) \
                          --revision-suffix "v2-$(Build.BuildId)" \
                          --set-env-vars "APP_VERSION=v2"

                  - task: AzureCLI@2
                    displayName: 'Configure 50/50 Traffic Split'
                    inputs:
                      azureSubscription: $(azureSubscription)
                      scriptType: 'bash'
                      scriptLocation: 'inlineScript'
                      inlineScript: |
                        # Get the previous revision (v1)
                        V1_REVISION=$(az containerapp revision list \
                          --name hello-api \
                          --resource-group $(resourceGroup) \
                          --query "[?properties.active && !contains(name, 'v2-')].name | [0]" -o tsv)
                        
                        # Get the new revision (v2)
                        V2_REVISION=$(az containerapp revision list \
                          --name hello-api \
                          --resource-group $(resourceGroup) \
                          --query "[?contains(name, 'v2-$(Build.BuildId)')].name | [0]" -o tsv)
                        
                        echo "V1 Revision: $V1_REVISION"
                        echo "V2 Revision: $V2_REVISION"
                        
                        # Split traffic 50/50
                        az containerapp ingress traffic set \
                          --name hello-api \
                          --resource-group $(resourceGroup) \
                          --revision-weight "$V1_REVISION=50" "$V2_REVISION=50"
                        
                        echo "Traffic split configured: 50% v1, 50% v2"

    ```

1. 更新されたパイプラインをコミットしてプッシュします。

    ```powershell
    git add azure-pipelines.yml
    git commit -m "Add blue-green deployment stage"
    git push

    ```

### ブルーグリーンの切り替え用にパイプライン変数を追加する

1. ブルーグリーン ステージを有効にするには、パイプライン変数を追加します。
    - **[パイプライン]** > **[パイプライン]** に移動します
    - パイプラインをクリックします
    - **[編集]** をクリックします
    - **[変数]** (右上) をクリックします
    - **[新しい変数]** をクリックします
    - 名前: `deployBlueGreen`
    - 値: `true`
    - **[OK]**、**[保存]** の順にクリックします

### 手動によるトラフィックの引き上げ

1. 50/50 分割をテストした後、パイプライン経由または手動で v2 を 100% のトラフィックに引き上げることができます。

    ```powershell
    # Promote v2 to 100% traffic
    $V2_REVISION = az containerapp revision list -n hello-api -g $RG `
        --query "[?contains(name, 'v2-')].name | [0]" -o tsv
    ```
    
    ```powershell
    az containerapp ingress traffic set `
        --name hello-api `
        --resource-group $RG `
        --revision-weight "${V2_REVISION}=100"
    
    Write-Host "V2 promoted to 100% traffic"

    ```

### パイプラインのトリガー

1. `azure-pipelines.yml` 内のパイプライン トリガーを確認します。

    ```yaml
    trigger:
      branches:
        include:
          - main
          - develop
      paths:
        include:
          - src/**
          - infra/**

    ```

1. この構成は、次のことを意味します。
    - パイプラインは、`main` または `develop` へのプッシュ時に自動的に実行されます
    - `src/` または `infra/` 内のファイルが変更された場合にのみトリガーされます
    - `main` トリガー検証ビルドへの pull request

### 継続的インテグレーションをテストする

1. パイプラインをトリガーするために小さな変更を行います。

    ```powershell
    # Make a change to the hello-api
    $content = Get-Content ".\src\hello-api\Program.cs"
    $content = $content -replace 'Contoso Analytics API', 'Contoso Analytics API v2'
    $content | Set-Content ".\src\hello-api\Program.cs"
    ```
    ```powershell
    # Commit and push
    git add .
    git commit -m "Update API title - trigger CI/CD"
    git push

    ```

1. パイプラインが自動的にトリガーされるのを確認します

1. デプロイが完了したことを確認します。

    ```powershell
    # Check the hello-api version
    Invoke-RestMethod -Uri "$HELLO_API_URL/api/version"

    ```

    > **注**: Azure DevOps Pipelines を使用して、自動化されたビルド、コンテナー レジストリ プッシュ、コンテナー アプリのデプロイなどの CI/CD が正常に実装されました。

## クリーンアップ

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。 デプロイ方法に合致するクリーンアップ方法を選択します。

### オプション A: Azure CLI を使用してクリーンアップする

このオプションは、演習 1 で**オプション A (Azure CLI + Bicep)** を使用してデプロイした場合に使用します。

1. リソース グループとその中のすべてのリソースを削除します。

    ```powershell
    # Delete the resource group and all resources within it
    az group delete --name $RG --yes --no-wait

    Write-Host "Resource group deletion initiated. This may take a few minutes."

    ```

1. 削除を確認します。

    ```powershell
    # Check if resource group still exists
    az group exists --name $RG

    ```

### オプション B: Azure Developer CLI (azd) を使用してクリーンアップする

このオプションは、演習 1 で**オプション B (azd)** を使用してデプロイした場合に使用します。

1. azd down コマンドを実行して、すべてのリソースを削除します。

    ```powershell
    # Delete all Azure resources provisioned by azd
    azd down --force --purge

    Write-Host "All resources have been deleted."

    ```

    > **注**: `--force` フラグは確認プロンプトをスキップし、`--purge` は、Key Vault シークレットなどの論理的に削除されたリソースを完全に削除します。

## まとめ

この演習では、以下の方法を学習しました。

| スキル | 行ったこと |
|-------|--------------|
| **インフラストラクチャのデプロイ** | Azure CLI で Bicep テンプレートを使用して Azure Container Apps インフラストラクチャをデプロイしました |
| **コンテナー管理** | コンテナー イメージをビルドして Azure Container Registry にプッシュし、デプロイしました |
| **自動スケーリング** | 追加の構成なしで負荷時の HTTP ベースの自動スケーリングを観察しました |
| **トラフィック分割** | パーセンテージベースのトラフィック ルーティングを使用してブルーグリーン デプロイを実装しました |
| **コンテナー ジョブ** | スケジュールされたジョブ、手動ジョブ、並列バッチ処理ジョブを操作しました |
| **Security** | セキュリティで保護されたシークレット管理のためにマネージド ID と Key Vault を使用しました |
| **CI/CD パイプライン** | コンテナー アプリの自動化されたビルドとデプロイ用に Azure DevOps Pipelines を設定しました |

### 重要なポイント

- **Azure Container Apps** により、Kubernetes の複雑さを伴わないサーバーレス コンテナー プラットフォームが提供されます
- **自動スケーリング**が組み込まれており、必要なのはシンプルなしきい値構成のみです
- **トラフィック分割**により、ダウンタイムなしのデプロイと A/B テストが可能になります
- **コンテナー ジョブ**により、Kubernetes CronJobs がシンプルな構成に置き換えられます
- **マネージド ID** により、セキュリティで保護されたパスワードレス認証が Azure サービスに提供されます
- **Azure DevOps Pipelines** により、コードのコミットから運用デプロイまでの CI/CD ワークフロー全体が自動化されます

## その他のリソース

- [Azure Container Apps のドキュメント](https://learn.microsoft.com/azure/container-apps/)
- [コンテナー アプリのスケーリング](https://learn.microsoft.com/azure/container-apps/scale-app)
- [トラフィック分割](https://learn.microsoft.com/azure/container-apps/revisions-manage)
- [Container Apps ジョブ](https://learn.microsoft.com/azure/container-apps/jobs)
- [マネージド ID](https://learn.microsoft.com/azure/container-apps/managed-identity)
- [Azure DevOps Pipelines](https://learn.microsoft.com/azure/devops/pipelines/)
- [Azure Pipelines から Azure Container Apps にデプロイする](https://learn.microsoft.com/azure/container-apps/azure-pipelines)
