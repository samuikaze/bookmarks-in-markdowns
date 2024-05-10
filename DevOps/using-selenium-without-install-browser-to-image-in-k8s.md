# Using Selenium without install browser to image in Kubernetes

使用 Python 爬取網頁資料是很常見的事情，但為了爬動態網頁裝了 selenium 套件後又得裝瀏覽器進 image，想必這 image 的大小會非常可觀
因此這邊會利用 Kubernetes 官方提供的範例說明如何把瀏覽器架在另一個容器中，而不是將整個瀏覽器塞到 image 中

> [!NOTE]
> 請注意，僅適用於 Selenium 4.x 版本

## 使用

1. 部署 Selenium Hub，新增以下內容到 `selenium-hub.yaml` 檔中，修改 `<>` 中的內容為正確的值，並執行 `kubectl apply -f selenium-hub.yaml` 套用設定到 Kubernetes 中

    > 註: Service 是方便使用 coredns 呼叫這個服務，如果想要暴露到網際網路上，你還需要再新增 Ingress 的宣告

    > 註: probe 的設定值可以參考[這篇](https://groups.google.com/g/selenium-users/c/vL7hjGyYRU4)的討論，解決 Selenium Hub 會一直被重啟的問題

    ```yaml
    # If you just want to deploy to default namespace, comment out namespace block.
    # selenium-hub-namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: <NAMESPACE>

    ---
    # selenium-hub-deployment.yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: selenium-hub
      # Specific the namespace or comment it out to deploy to default namespace
      namespace: <NAMESPACE>
      labels:
        app: selenium-hub
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: selenium-hub
      template:
        metadata:
          labels:
            app: selenium-hub
        spec:
          containers:
          - name: selenium-hub
            image: selenium/hub:4.0
            ports:
              - containerPort: 4444
              - containerPort: 4443
              - containerPort: 4442
            resources:
              limits:
                memory: "1000Mi"
                cpu: ".5"
            livenessProbe:
              httpGet:
                path: /wd/hub/status
                port: 4444
              initialDelaySeconds: 30
              timeoutSeconds: 30
              failureThreshold: 5
            readinessProbe:
              httpGet:
                path: /wd/hub/status
                port: 4444
              initialDelaySeconds: 30
              timeoutSeconds: 30
              failureThreshold: 5

    ---
    # selenium-hub-service.yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: selenium-hub
      # Specific the namespace or comment it out to deploy to default namespace
      namespace: <NAMESPACE>
      labels:
        app: selenium-hub
    spec:
      ports:
      - port: 4444
        targetPort: 4444
        name: port0
      - port: 4443
        targetPort: 4443
        name: port1
      - port: 4442
        targetPort: 4442
        name: port2
      selector:
        app: selenium-hub
      type: NodePort
      sessionAffinity: None

    ```

2. 確認服務是否正常運作，請將 `selenium-hub` 這個 Service 的其中一個連接埠進行轉發後，利用 `http://localhost:<PORT>` 網址去瀏覽其頁面，如正常顯示網頁則部署成功
    > `<PORT>` 為連接埠轉發後的埠號
3. 部署瀏覽器節點 (Firefox 或 Chrome)，新增以下 yaml 檔其一並執行 `kubectl create -f <YAML_FILE_NAME>` 進行部署

    > 請注意: 不管是哪一種瀏覽器，都支援 replicas 大於 1 的設定，請視需要設定這個數字，預設為 1

    - Chrome 節點

      ```yaml
      # chrome-node.yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        # Specific the namespace or comment it out to deploy to default namespace
        namespace: <NAMESPACE>
        name: selenium-node-chrome
        labels:
          app: selenium-node-chrome
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: selenium-node-chrome
        template:
          metadata:
            labels:
              app: selenium-node-chrome
          spec:
            volumes:
            - name: dshm
              emptyDir:
                medium: Memory
            containers:
            - name: selenium-node-chrome
              image: selenium/node-chrome:4.0
              ports:
                - containerPort: 5555
              volumeMounts:
                - mountPath: /dev/shm
                  name: dshm
              env:
                - name: SE_EVENT_BUS_HOST
                  value: "selenium-hub"
                - name: SE_EVENT_BUS_SUBSCRIBE_PORT
                  value: "4443"
                - name: SE_EVENT_BUS_PUBLISH_PORT
                  value: "4442"
              resources:
                limits:
                  memory: "1000Mi"
                  cpu: ".5"
      ```

    - Firefox 節點

      ```yaml
      # firefox-node.yaml
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: selenium-node-firefox
        # Specific the namespace or comment it out to deploy to default namespace
        namespace: <NAMESPACE>
        labels:
          app: selenium-node-firefox
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: selenium-node-firefox
        template:
          metadata:
            labels:
              app: selenium-node-firefox
          spec:
            volumes:
            - name: dshm
              emptyDir:
                medium: Memory
            containers:
            - name: selenium-node-firefox
              image: selenium/node-firefox:4.0
              ports:
                - containerPort: 5555
              volumeMounts:
                - mountPath: /dev/shm
                  name: dshm
              env:
                - name: SE_EVENT_BUS_HOST
                  value: "selenium-hub"
                - name: SE_EVENT_BUS_SUBSCRIBE_PORT
                  value: "4443"
                - name: SE_EVENT_BUS_PUBLISH_PORT
                  value: "4442"
              resources:
                limits:
                  memory: "1000Mi"
                  cpu: ".5"
      ```

4. Selenium 使用

    > 請務必注意，Selenium Hub 如果會重新啟動，表示 Selenium 執行過程中有程式掛掉了，請把問題抓出來，否則每拋出一次例外，Selenium Hub 就會重新啟動一次

    在建構 driver 時，除了要使用 `webdriver.Remote()` 進行建構外，帶入 `options` 後，需額外帶 `command_executor='http://selenium-hub.<NAMESPACE>:4444/wd/hub'` 參數給建構式，這樣 Selenium 才會去呼叫這個 URL 中提供的瀏覽器節點取得網頁資料，更多的說明或測試腳本可以在官方儲存庫中取得

## 節點重啟後無法正常爬取資料

發生原因目前不明，但已知遇到此問題時重新部署或重新啟動瀏覽器的 Pod 就可以解決

## 參考資料

- [Selenium on Kubernetes](https://github.com/kubernetes/examples/tree/master/staging/selenium)
- [Concurrent Web Scraping with Selenium Grid and Docker Swarm](https://testdriven.io/blog/concurrent-web-scraping-with-selenium-grid-and-docker-swarm/)
- [Building a Concurrent Web Scraper with Python and Selenium](https://testdriven.io/blog/building-a-concurrent-web-scraper-with-python-and-selenium/)
- [Selenium hub shutdowns when it receives SIGTERM signal from supervisord](https://groups.google.com/g/selenium-users/c/vL7hjGyYRU4)
