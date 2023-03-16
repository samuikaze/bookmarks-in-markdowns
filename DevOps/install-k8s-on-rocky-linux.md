# Install Kubernetes On Rocky Linux 9

要在 Rocky Linux 9 下安裝 Kubernetes 請依據下方步驟進行安裝

> ※ 以下步驟僅適用於裸機安裝 (Bare-Metal)，雲服務不適用於本文章

> ※ 所有的指令前綴為 `$` 表不需要 root 權限， `#` 則需要 root 權限，`>` 表示繼續輸入。

## 全自動化懶人安裝法

- [使用 Ansible 在 Rocky Linux 上安裝 k8s](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/)

## 手動安裝

1. 設定機器域名與名稱

    > 這段其實非必要，只是方便測試時不用一直打 IP。

    > 執行指令前請先確認網路是否已經完成設定，特別是 IP 部分，請盡量不要使用 DHCP 自動派發

    > ** \<HOSTNAME\> 需符合 FQDN 的規範**

    ```console
    # hostnamectl set-hostname <HOSTNAME>
    # echo <NETWORK_IP> <HOSTNAME> <COMPUTER_NAME> >> /etc/hosts
    ```

2. 更新系統所有套件至最新

    ```console
    # dnf update --refresh -y
    ```

3. 關閉 SELinux (**不推薦**)

    > **非常不推薦**關閉 SELinux 可能會使伺服器暴露於危險之中，應當保持其開啟，並於需要時設定其策略

    ```console
    # setenforce 0
    # sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    ```

4. 關閉 firewalld (防火牆)

    > 由於 Calico 和 firewalld 都是相依於 iptables 的實作，保持 firewalld 啟用會導致 Kubernetes 中所有的 Pod 都無法利用 Cluster IP 互相連線，若要保持開啟，則 ingress 中必須要增加 `nginx.ingress.kubernetes.io/service-upstream=true` 的 annotation，但這僅適用於 nginx ingress，若非使用 nginx ingress 或必須使用 Pod IP 相互溝通，則必須關閉 firewalld。

    > Crio 本身有自帶 CNI，但由於文件過少，且未找到如何將防火牆設定帶進來，因此不使用其自帶的 CNI。

    > 後面會利用 Calico 網路策略去管理防火牆的策略

    ```console
    # systemctl stop firewalld.service
    # systemctl disable firewalld.service
    ```

5. 載入必要的 Linux 核心模組

    ```console
    # modprobe overlay
    # modprobe br_netfilter
    ```

    ```console
    # cat > /etc/modules-load.d/kubernetes.conf << EOF
    > overlay
    > br_netfilter
    > EOF
    ```

    ```console
    # cat > /etc/sysctl.d/kubernetes.conf << EOF
    > net.ipv4.ip_forward = 1
    > net.bridge.bridge-nf-call-ip6tables = 1
    > net.bridge.bridge-nf-call-iptables = 1
    > EOF
    ```

6. 套用變更

    ```console
    # sysctl --system
    ```

7. 關閉 Swap

    ```console
    # swapoff -a
    # sed -e '/swap/s/^/#/g' -i /etc/fstab
    ```

8. 驗證 Swap 是否已經關閉

    ```console
    # free -m
    ```

9. 安裝 crio (推薦第一種方式，可以直接安裝最新版的 crio)

    ```console
    # mkdir /usr/local/bin/runc
    # curl https://raw.githubusercontent.com/cri-o/cri-o/main/scripts/get | bash
    # systemctl enable --now crio
    ```

    或

    ```console
    # VERSION=<預計要安裝的 K8s 版本>
    # curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo
    # curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_8/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo
    # dnf install cri-o cri-tools -y
    # systemctl enable --now crio
    ```

10. 新增 Repo list

    ```console
    # cat > /etc/yum.repos.d/kubernetes.repo << EOF
    > [kubernetes]
    > name=Kubernetes
    > baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    > enabled=1
    > gpgcheck=1
    > gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    > exclude=kubelet kubeadm kubectl
    > EOF
    ```

11. 安裝並啟用 kubelet、kubectl 和 kubeadm

    ```console
    # dnf install -y {kubelet,kubeadm,kubectl} --disableexcludes=kubernetes
    # systemctl enable --now kubelet.service
    ```

12. 安裝 kubectl bash 指令自動完成

    ```console
    # source <(kubectl completion bash)
    # kubectl completion bash > /etc/bash_completion.d/kubectl
    ```

13. 拉取設定檔

    ```console
    # kubeadm config images pull --cri-socket unix:///var/run/crio/crio.sock
    ```

14. 初始化 kubeadm

    > ※ `<POD_NETWORK_CIDR>` 請更改為預計讓 pod 使用的網段，此設定會影響後續安裝 Calico 的相關設定

    ```console
    # kubeadm init \
        --pod-network-cidr=<POD_NETWORK_CIDR> \
        --cri-socket unix:///var/run/crio/crio.sock
    ```

15. 讓 kubectl 指令可以不用 sudo 執行

    ```console
    $ mkdir -p $HOME/.kube
    # cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
    # chown $(id -u):$(id -g) $HOME/.kube/config
    ```

16. 讓 Control Plane 機器也可以部署 Pod

    > ※ 若此指令無效，請執行 `kubectl describe nodes` 去看目前 Control Plane 那台 node 目前的汙點名稱為何，將 `node-role.kubernetes.io/control-plane:NoSchedule` 變更為正確的汙點名稱就可以了

    ```console
    kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
    ```

17. 安裝 calicoctl

    > 使用雲服務可以跳過此步驟，雲服務供應商會有自己的 CNI 介面

    > 建議每次安裝都到官方網站[複製最新版本的指令](https://docs.tigera.io/calico/3.25/operations/calicoctl/install#install-calicoctl-as-a-binary-on-a-single-host)

    ```console
    # cd /usr/local/bin
    # curl -L https://github.com/projectcalico/calico/releases/latest/download/calicoctl-linux-amd64 -o calicoctl
    # chmod +x ./calicoctl
    ```

18. 設定 calicoctl 對於 etcd 的連線 (這邊使用 Kubernetes 自帶的 etcd 資料庫)

    > 使用雲服務可以跳過此步驟，雲服務供應商會有自己的 CNI 介面

    ```console
    # mkdir /etc/calico
    # cat > /etc/calico/calicoctl.cfg << EOF
    > apiVersion: projectcalico.org/v3
    > kind: CalicoAPIConfig
    > metadata:
    > spec:
    >   datastoreType: 'kubernetes'
    >   kubeconfig: '/path/to/.kube/config'
    > EOF
    ```

19. 避免 NetworkManager 防止 Calico 修改路由表規則

    > 使用雲服務可以跳過此步驟，雲服務供應商會有自己的 CNI 介面

    ```console
    # cat > /etc/NetworkManager/conf.d/calico.conf << EOF
    > [keyfile]
    > unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
    > EOF
    ```

20. 安裝 Calico 當作 CNI

    > 使用雲服務可以跳過此步驟，雲服務供應商會有自己的 CNI 介面

    > 這邊都是使用「低於 50 台節點的設定檔」，如需大於 50 台節點的設定檔，請至[官方網站下載](https://docs.tigera.io/calico/3.25/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico)

    - 下載 kubectl 的 yaml 檔案

    ```console
    $ curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O
    ```

    - 若前面的 `<POD_NETWORK_CIDR>` 不是設定為 `192.168.0.0/16`，請在這邊打開文件搜尋 `CALICO_IPV4POOL_CIDR`，把該段 env 註解打開後修改為自己的 Pod 網段即可

    > 官方表示即便不改，Calico 也會自動偵測，如果不放心的話還是改一下比較妥當

    - 執行 `kubectl apply -f calico.yaml` 套用設定檔到 Kubernetes，待所有 Pod 都 Up 後就算安裝完成

    > 安裝完成後部分 Pod 的 IP 可能不是正確的，請利用 Deployment 等方式重啟 Pod 即可。

21. 設定 Calico 的網路策略

    > 目前尚在 POC 中，已知需開啟 6443/TCP、2379/TCP、2380/TCP、10250/TCP、10251/TCP、10252/TCP 連接埠

22. 部署 nginx ingress controller

    > ※ 建議每次都從[官方文件](https://kubernetes.github.io/ingress-nginx/deploy/)中複製 yaml 檔網址，以確保 ingress 版本是最新的穩定版本

    ```console
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
    ```

23. LoadBalancer 設定外部 IP

    > 使用雲服務 (例: AWS 等)可以跳過此項設定，K8s 會自動綁訂雲服務供應商的 Load Balancer IP

    - 若是使用裸機 (Bare-Metal) 安裝，有以下兩種方式可以設定對外 IP
        1. 使用 [MetalLB](https://metallb.universe.tf/)
            - 調整 kube-proxy 設定

                ```console
                $ kubectl get configmap kube-proxy -n kube-system -o yaml | \
                    sed -e "s/strictARP: false/strictARP: true/" | \
                    kubectl apply -f - -n kube-system
                ```

            - 使用下面指令或[到官方網站上複製指令](https://metallb.universe.tf/installation/#installation-by-manifest)安裝 MetalLB

                ```console
                $ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.9/config/manifests/metallb-native.yaml
                ```

            - 新增一份 yaml 檔，並以 kubectl 套用此設定，其中 yaml 檔需包含以下內容:

                ```yaml
                # external-ip-pool.yaml
                apiVersion: metallb.io/v1beta1
                kind: IPAddressPool
                metadata:
                  name: <IP_POOL_NAME>
                  namespace: metallb-system
                spec:
                  addresses:
                  - <YOUR_IP_POOL_1>
                  - <YOUR_IP_POOL_2>
                  - ...

                ---
                # l2-advertisement.yaml
                apiVersion: metallb.io/v1beta1
                kind: L2Advertisement
                metadata:
                  name: l2-advertisement
                  namespace: metallb-system
                spec:
                  ipAddressPools:
                  - <IP_POOL_NAME>
                ```

        2. 針對 Services 中 LoadBalancer 的 nginx 服務 spec 新增下列設定

            > 除非有必要，Nginx 官方不推薦使用此方式設定外部 IP

            > 建議每台 Kubernetes 的 node 都設定一組固定的 IP，避免不必要的麻煩

            ```console
            externalIPs:
            - <機器對外網卡上設定的固定 IP>
            ```

            或是使用指令直接套用

            ```console
            $ kubectl patch svc ingress-nginx-controller -n ingress-nginx -p '{"spec": {"externalIPs": ["<機器對外網卡上設定的固定 IP>"]}}'
            ```

24. 部署服務與相關設定 (Deployments、Service、Ingress、ConfigMap、SecretMap)

    > ※ 請記得先將映像 (image) 推到指定的 Registry 中，否則部署後 Pod 將無法正常運作

    > ※ 若 ingress-nginx 無法透過 Pod 的 IP 連上 Pod (未安裝 CNI 或未關閉防火牆)，請將下面這行加到該 App 的 Ingress 的 annotation 中

    ```yaml
    annotation:
      ...
      nginx.ingress.kubernetes.io/service-upstream=true
      ...
    ```

25. 設定 registry 的登入帳號密碼

    > 由於 Secrets 無法跨命名空間 (namespace) 使用，故如有多個命名空間，每個命名空間都需要部署一份 Secrets

    1. 執行以下指令以建立帳號密碼的 Secrets

        > `--docker-email` 為選填，若沒有設定電子郵件，可以省略此行

        > 請注意，命名空間 (Namespace) 請務必與 Pod 相同，否則無法讀到此 Secrets，Secrets 本身的名稱則無限制

        ```txt
        kubectl create secret docker-registry \
            -n <APP_NAMESPACE> <SECRET_NAME> \
            --docker-server=<YOUR_REGISTRY_SERVER> \
            --docker-username=<REGISTRY_USERNAME> \
            --docker-password=<REGISTRY_PASSWORD> \
            --docker-email=<YOUR_EMAIL>
        ```

    2. 套用 Secrets 至 Pod 中

        > 下面兩種方式擇一使用就可以了

        - 直接在 Deployment 中代入 Secrets:

            在 `spec.template.spec` 底下新增下面語句

            > 請注意，Secrets 請務必與 Deployment 部屬的 Pod 所屬的命名空間相同，否則會讀不到

            > 請將 `<YOUR_REGISTRY_CREDENTIALS_SECRETS>` 取代為正確的值

            ```txt
            imagePullSecrets:
              - name: <YOUR_REGISTRY_CREDENTIALS_SECRETS>
            ```

        - 使用 Service Account 方式

            - 打開終端機並執行 `kubectl patch serviceaccount default -n <NAMESPACE> -p '{"imagePullSecrets": [{"name": "<YOUR_REGISTRY_CREDENTIALS_SECRETS>"}]}'`

            > 請將 `<NAMESPACE>` 與 `<YOUR_REGISTRY_CREDENTIALS_SECRETS>` 取代為正確的值

            > 若 Service Account 有特別指定，請將 `default` 調整為該 Service Account 的名稱

    3. 部署應用程式，完成

## 其它資料

- [刪除不在預設命名空間的服務](https://stackoverflow.com/a/67517905)

## 參考資料

- [使用 Ansible 在 Rocky Linux 上安裝 k8s](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/)
- [手動在 Rocky Linux 上安裝 k8s](https://www.centlinux.com/2022/11/install-kubernetes-master-node-rocky-linux.html)
- [Install Kubernetes Cluster on Ubuntu 20.04 with kubeadm](https://computingforgeeks.com/deploy-kubernetes-cluster-on-ubuntu-with-kubeadm/)
- [在 Rocky Linux 上安裝 crio](https://computingforgeeks.com/install-cri-o-container-runtime-on-rocky-linux-almalinux/)
- [Crio GitHub](https://github.com/cri-o/cri-o#installing-cri-o)
- [nginx ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/)
- [讓 Control Plane 可以部署 Pod](https://blog.csdn.net/lisongyue123/article/details/108365127)
- [LoadBalancer 設定外部 IP](https://stackoverflow.com/questions/44110876/kubernetes-service-external-ip-pending/54168660#54168660)
- [Kubernetes: 502 Bad Gateway for some assets - with Nginx Ingress](https://serverfault.com/questions/954620/kubernetes-502-bad-gateway-for-some-assets-with-nginx-ingress)
- [Source of "service-upstream" annotation](https://github.com/kubernetes/ingress-nginx/issues/1120#issuecomment-418206748)
- [What are Kubernetes Secrets and Service Accounts?](https://tanzu.vmware.com/developer/guides/platform-security-secrets-sa-what-is/)
- [Sharing secret across namespaces](https://stackoverflow.com/a/46299290)
- [Service Account](https://kubernetes.io/docs/concepts/security/service-accounts/)
- [Using private registry docker images in Kubernetes when launched using docker stack deploy](https://stackoverflow.com/a/57831913)
- [Install Calico networking and network policy for on-premises deployments](https://docs.tigera.io/calico/3.25/getting-started/kubernetes/self-managed-onprem/onpremises)
- [Calico System requirements](https://docs.tigera.io/calico/3.25/getting-started/kubernetes/requirements)
- [Configure NetworkManager](https://docs.tigera.io/calico/3.25/operations/troubleshoot/troubleshooting#configure-networkmanager)
