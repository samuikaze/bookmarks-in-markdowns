# Apply Custom Helm CHarts to Kubernetes

以下將會說明如何將自建的 Helm Charts 部署到 Kubernetes 中

## 部署說明

1. 先透過[官方網站提供的方式](https://helm.sh/docs/intro/install/)安裝 Helm
2. 打開終端機並切換到自建 Charts 的資料夾或其上一層的資料夾
3. 在終端機中執行 `helm install <CHART_NAME> <FOLDER_NAME_INCLUDES_CHARTS>/ --values <VALUES_YAML_RELATED_PATH>`

    > `<CHART_NAME>` 為 charts 的名稱
  
    > `<FOLDER_NAME_INCLUDES_CHARTS>` 為 charts 所在資料夾的名稱，若為當下資料夾，請輸入 `.`
  
    > `<VALUES_YAML_RELATED_PATH>` 為 charts 變數的 values.yaml 檔所在相對路徑

    > 範例: `helm install my-custom-chart custom-charts/ --values custom-charts/values.yaml`

4. 完成

## 參考資料

- [How To Create A Helm Chart](https://phoenixnap.com/kb/create-helm-chart)
- [How to make a Helm chart in 10 minutes](https://opensource.com/article/20/5/helm-charts)
- [Kubernetes 基礎教學（三）Helm 介紹與建立 Chart](https://cwhu.medium.com/kubernetes-helm-chart-tutorial-fbdad62a8b61)
- [Helm Install - Helm](https://helm.sh/docs/helm/helm_install/)
- [How to Use the helm install Command](https://phoenixnap.com/kb/helm-install-command)
