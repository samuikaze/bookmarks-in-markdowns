# Generate Kubernetes kubeconfig file for service account

此文件將會說明如何產出 kubeconfig 給其他使用者使用

## 建立服務帳號與叢集角色

1. 若要建立在非 default 命名空間，請利用以下 yaml 建立命名空間

    ```yaml
    # namespace.yaml
    apiVersion: v1
    kind: Namespace
    metadata:
      name: <NAMESPACE>
    ```

2. 建立服務帳號的 secret
    利用以下 yaml 檔建立服務帳號的 secret

    ```yaml
    # service-account-token-secret.yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: <SERVICE_ACCOUNT_SECRET_NAME>
      namespace: <NAMESPACE>
    annotations:
      kubernetes.io/service-account.name: <SERVICE_ACCOUNT_NAME>
    type: kubernetes.io/service-account-token
    ```

3. 建立服務帳號
    利用以下 yaml 檔建立服務帳號

    ```yaml
    # service-account.yaml
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: <SERVICE_ACCOUNT_NAME>
      namespace: <NAMESPACE>
    secrets:
      - name: <SERVICE_ACCOUNT_SECRET_NAME>
    ```

4. 建立叢集角色
    利用以下 yaml 檔建立叢集角色

    > `apiGroups`、`resources` 和 `verbs` 可以利用指令 `clear && kubectl api-resources --sort-by name -o wide` 查詢

    ```yaml
    # cluster-role.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: <CLUSTER_ROLE_NAME>
    rules:
      - apiGroups:
          - ''
          - apps
          - batch
          - networking.k8s.io
        resources:
          - cronjobs
          - configmaps
          - daemonsets
          - deployments
          - endpoints
          - ingresses
          - jobs
          - namespaces
          - nodes
          - pods
          - podtemplates
          - replicasets
          - replicationcontrollers
          - resourcequotas
          - services
          - statefulsets
        verbs:
          - get
          - list
          - watch
    ```

5. 建立服務帳號與叢集角色的連結
    利用以下 yaml 檔建立服務帳號與叢集角色的連結

    ```yaml
    # cluster-role-binding.yaml
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: <CLUSTER_ROLE_BINDING_NAME>
    subjects:
    - kind: ServiceAccount
      name: <SERVICE_ACCOUNT_NAME>
      namespace: <NAMESPACE>
    roleRef:
      kind: ClusterRole
      name: <CLUSTER_ROLE_NAME>
      apiGroup: rbac.authorization.k8s.io
    ```

## 建立 kubeconfig 檔案

將以下內容存成 `kubeconfig-generator.sh` 檔後，執行 `bash kubeconfig-generator.sh` 就會自動在目前的資料夾產生一支 kubeconfig 檔案

> 請記得修改以 `<` 和 `>` 包起來的大寫文字為正確的值

> `<KUBERNETES_API_SERVER_URL>` 請修改為**外網**可以連線的網址

> 產生出來的 kubeconfig 檔案名稱可以隨意修改

```shell
#!/bin/bash

clusterName='<YOUR_CLUSTER_NAME>'
server='https://<KUBERNETES_API_SERVER_URL>:6443'
namespace='<NAMESPACE>'
serviceAccount='<SERVICE_ACCOUNT_NAME>'
secretName='<SERVICE_ACCOUNT_SECRET_NAME>'

set -o errexit

ca=$(kubectl --namespace="$namespace" get secret/"$secretName" -o=jsonpath='{.data.ca\.crt}')
token=$(kubectl --namespace="$namespace" get secret/"$secretName" -o=jsonpath='{.data.token}' | base64 --decode)

echo "
---
apiVersion: v1
kind: Config
clusters:
  - name: ${clusterName}
    cluster:
      certificate-authority-data: ${ca}
      # 如 TLS 憑證為自簽憑證，請將以下的設定打開
      # insecure-skip-tls-verify: true
      server: ${server}
contexts:
  - name: ${serviceAccount}@${clusterName}
    context:
      cluster: ${clusterName}
      namespace: ${namespace}
      user: ${serviceAccount}
users:
  - name: ${serviceAccount}
    user:
      token: ${token}
current-context: ${serviceAccount}@${clusterName}
" > generated.kubeconfig
```

## 參考資料

- [How to create a kubectl config file for serviceaccount](https://stackoverflow.com/a/47776588)
- [kubeadm kubeconfig](https://kubernetes.io/zh-cn/docs/reference/setup-tools/kubeadm/kubeadm-kubeconfig/)
- [使用 kubeadm 進行憑證管理](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/#kubeconfig-additional-users)
- [使用 RBAC 鑑權](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/)
- [Kubernetes RBAC Port Forward](https://medium.com/@ManagedKube/kubernetes-rbac-port-forward-4c7eb3951e28)
- [List of Kubernetes RBAC rule verbs](https://stackoverflow.com/a/65245307)
