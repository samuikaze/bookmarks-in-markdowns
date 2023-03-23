# Using Action Runner Controller to Host Self-Hosted GitHub Runner in Kubernetes

以下將會說明如何利用 Action Runner Controller 將 GitHub Actions Runner 架到 Kubernetes 中

## Action Runner Controller 安裝

1. 安裝 `cert-manager` 可參考[此文章](./install-k8s-on-rocky-linux.md)中安裝 cert-manager 的部分
2. 建立 GitHub Personal Access Token (PAT)
    > 可以[點此](https://github.com/settings/tokens/new)直接進行 PAT 建立，若不信任網址，也可遵循以下流程進入頁面建立 PAT

    > PAT 建立後僅會顯示一次權杖值，請好好保存，若在完成 GitHub Actions Runner 建置前遺失，則需要再重新申請一個

    - 登入 GitHub
    - 打開頁面左上角帳號選單中選擇 `Settings`
    - 左側選單選擇 `Developer settings`
    - 左側選單選擇 `Personal access tokens`
    - 左側選單選擇 `Tokens (classic)`
    - 右上角選擇 `Generate new token` > `Generate new token (classic)`
    - `Note` 輸入這個權杖的說明
    - `Expiration` 選擇 `No expiration` (無期限)
    - `Scope` 選擇整個 `repo`
    - 點選 `Generate token` 建立權杖
    - 下一頁會顯示產生的權杖值，請將之複製下來，後續會使用到
3. 安裝 Action Runner Controller
    > 不論使用什麼方式安裝，請務必將 `REPLACE_YOUR_TOKEN_HERE` 取代為剛剛申請的 PAT
    - Helm 安裝

        ```console
        $ helm repo add actions-runner-controller https://actions-runner-controller.github.io/actions-runner-controller
        $ helm upgrade --install --namespace actions-runner-system --create-namespace \
            --set=authSecret.create=true \
            --set=authSecret.github_token="REPLACE_YOUR_TOKEN_HERE" \
            --wait actions-runner-controller actions-runner-controller/actions-runner-controller
        ```

    - Kubectl 安裝

        ```console
        $ kubectl apply -f \
            https://github.com/actions/actions-runner-controller/ \
            releases/download/v0.22.0/actions-runner-controller.yaml
        $ kubectl create secret generic controller-manager \
            -n actions-runner-system \
            --from-literal=github_token=REPLACE_YOUR_TOKEN_HERE
        ```

4. 建立 GitHub self hosted runners，並將其指定給儲存庫使用

    > 請注意，儲存庫的名稱 `<YOUR_REPOSITORY_NAME>` 必須包含帳號與斜線，例如: `gitacc/test-repo`

    ```yaml
    apiVersion: actions.summerwind.dev/v1alpha1
    kind: RunnerDeployment
    metadata:
      name: <RUNNER_DEPLOYMENT_NAME>
      namespace: <DEPLOY_TARGET_NAMESPACE>
    spec:
      # 若有設定自動擴縮容，這邊可以不需要指定 replicas
      # replicas: 1
      template:
        spec:
          repository: <YOUR_REPOSITORY_NAME>
          ephemeral: true
          # 你可以指定這個 runner 的名稱，如此一來就可以在 workflow 中指定
          # labels:
          #   - <RUNNER_NAME>
    ```

5. 設定 Runner 自動擴縮容

    > 擴縮容最小值可以設為 0，但是 Pull Driven Scaling 有些限制，詳見[官方文件](https://github.com/actions/actions-runner-controller/blob/master/docs/automatically-scaling-runners.md#autoscaling-tofrom-0)

    > 自動擴縮容建議使用 Webhook Driven Scaling 方式

    1. 使用 Pull Driven Scaling
        > `scaleUpFactor` 和 `scaleUpAdjustment` 同時僅有一方能存在於 yaml 檔中，同理 `scaleDownFactor` 與 `scaleDownAdjustment` 也一樣

        > 使用 Pull Driven Scaling 的缺點是擴縮容不即時

        ```yaml
        # runner-pull-driven-autoscaler.yaml
        apiVersion: actions.summerwind.dev/v1alpha1
        kind: HorizontalRunnerAutoscaler
        metadata:
          name: <RUNNER_DEPLOYMENT_AUTOSCALER_NAME>
          namespace: <DEPLOY_TARGET_NAMESPACE>
        spec:
          # 設定 Runner 自動縮容的時間 (秒)，預設為 10 分鐘
          scaleDownDelaySecondsAfterScaleOut: 300
          scaleTargetRef:
            kind: RunnerDeployment
            # 如果是 RunnerSet 請打開以下註解
            # kind: RunnerSet
            name: <RUNNER_DEPLOYMENT_NAME>
          # 最小 Runner 數量
          minReplicas: 1
          # 最多 Runner 數量
          maxReplicas: 2
          metrics:
            # 各 type 僅能存在一種
            # 指定擴縮容的閥值
            - type: PercentageRunnersBusy
                # 擴容閥值
                scaleUpThreshold: '0.75'
                # 縮容閥值
                scaleDownThreshold: '0.3'
                # 以下兩種方式僅能存在一種
                # 以乘數進行擴容 (擴縮容時為當下 pod 數量乘上此值)
                # 擴容乘數
                scaleUpFactor: '2'
                # 縮容乘數
                scaleDownFactor: '0.5'

                # 以指定數量方式擴縮容 (擴縮容後的 pod 數量)
                # 擴容目標數量
                scaleUpAdjustment: 2
                # 縮容目標數量
                scaleDownAdjustment: 1
            # 以目前 CI/CD 數量決定要不要擴縮容
            - type: TotalNumberOfQueuedAndInProgressWorkflowRuns
              # 儲存庫名稱
              # 儲存庫的名稱不需要包含 GitHub 帳號前綴
              repositoryNames:
                - <REPOSITORY_NAME_1>
                - <REPOSITORY_NAME_2>
                ...
        ```

    2. 使用 Webhook Driven Scaling

        > 此方法需要額外以 Helm 安裝 webhook 伺服器，詳見 2. 開始的文件

        1. 套用設定到 Kubernetes 中

            ```yaml
            # runner-webhook-driven-autoscaler.yaml
            apiVersion: actions.summerwind.dev/v1alpha1
            kind: HorizontalRunnerAutoscaler
            metadata:
              name: <RUNNER_DEPLOYMENT_AUTOSCALER_NAME>
              namespace: <DEPLOY_TARGET_NAMESPACE>
            spec:
              # 最小 Runner 數量
              minReplicas: 1
              # 最多 Runner 數量
              maxReplicas: 2
              scaleTargetRef:
                kind: RunnerDeployment
                # 如果是 RunnerSet 請打開以下註解
                # kind: RunnerSet
                name: <RUNNER_DEPLOYMENT_NAME>
              scaleUpTriggers:
                - githubEvent:
                    workflowJob: {}
                  # 擴縮容有效時間
                  duration: "30m"
            ```

        2. 安裝 GitHub webhook 伺服器

            ```console
            $ helm upgrade --install --namespace actions-runner-system --create-namespace \
                --wait actions-runner-controller actions-runner-controller/actions-runner-controller \
                --set "githubWebhookServer.enabled=true"
            ```

        3. 設定 GitHub webhook 伺服器的 Ingress

            ```yaml
            # github-wwebhook-ingress.yaml
            apiVersion: networking.k8s.io/v1
            kind: Ingress
            metadata:
              name: actions-runner-controller-github-webhook-server
              namespace: actions-runner-system
              annotations:
                kubernetes.io/ingress.class: nginx
                # 指定 ingress-nginx 與後端溝通的協定
                nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
            spec:
              tls:
                - hosts:
                  - <YOUR_DOMAIN>
                  secretName: <YOUR_DOMAIL_TLS_SECRET>
              rules:
                # 若要限制僅有特定域名可以存取，請打開下面的註解
                # - host: <YOUR_DOMAIN>
                - http:
                    paths:
                      - path: <PATH_TO_GITHUB_WEBHOOK_SERVER>
                        pathType: Prefix
                        backend:
                          service:
                            name: actions-runner-controller-github-webhook-server
                            port:
                              number: 80
            ```

        4. GitHub 上找到該專案，進 Settings 中新增 Webhook 伺服器
            - 登入 GitHub 並找到目標儲存庫
            - 點進儲存庫的 `Settings`
            - 選擇左邊選單的 `Webhooks`
            - 點選右上角的 `Add webhook`
            - `Payload URL` 填入剛剛設定在 Kubernetes 中的域名 + 路徑
            - `Content type` 選擇 `application/json`
            - `Which events would you like to trigger this webhook?` 選擇 `Let me select individual events.`
            - 將所有勾選取消後，僅選擇 `Workflow runs`
            - 點選 `Add webhook` 確認新增
            - GitHub 進行伺服器連線驗證，沒問題會顯示打勾
            - 切換到左側選單的 `Actions` > `Runners` 確認有沒有剛剛設定的 runner 出現

6. 測試 Runner

    > 尚待測試

## 本文主要參考文章

- [Actions Runner Controller](https://github.com/actions/actions-runner-controller)

### 其它參考文章

- [myoung34/docker-github-actions-runner](https://github.com/myoung34/docker-github-actions-runner)
- [How to host your own GitHub Actions pipelines](https://www.youtube.com/watch?v=d3isYUrPN7s)
- [Running self-hosted GitHub Actions runners in your Kubernetes cluster](https://sanderknape.com/2020/03/self-hosted-github-actions-runner-kubernetes/)
- [evryfs/github-actions-runner-operator](https://github.com/evryfs/github-actions-runner-operator)
- [關於從 GitHub Actions Self-Hosted Runner 中偷 Secrets/Credentials 的一些安全研究](https://nova.moe/steal-credentials-from-ci-agents/)
- [extracting a variable's value from text file using bash](https://stackoverflow.com/questions/35296827/extracting-a-variables-value-from-text-file-using-bash)
- [nikic/php-parser](https://github.com/nikic/PHP-Parser)
- [C 編譯器入門 ～想懂低階系統從自幹編譯器開始～](https://koshizuow.gitbook.io/compilerbook/intoduction)
- [How to write a very basic compiler](https://softwareengineering.stackexchange.com/questions/165543/how-to-write-a-very-basic-compiler)
- [PHP Swagger OpenAPI Generic Type](https://stackoverflow.com/questions/71901795/php-swagger-openapi-generic-type)
- [How to set up file permissions for Laravel?](https://stackoverflow.com/a/37266353)
- [Convert Laravel Route::any into Lumen Route](https://stackoverflow.com/questions/57863502/convert-laravel-routeany-into-lumen-route)
