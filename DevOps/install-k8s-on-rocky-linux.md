# Install Kubernetes On Rocky Linux 9

要在 Rocky Linux 9 下安裝 Kubernetes 請依據下方步驟進行安裝

> ※ 以下步驟僅適用於裸機安裝 (Bare-Metal)，雲服務不適用於本文章

> ※ 所有的指令前綴為 `$` 表不需要 root 權限， `#` 則需要 root 權限，`>` 表示繼續輸入。

> 文件中以 `<` 和 `>` 包住且裡面的文字為**大寫英文以底線區隔單字者**，表該參數需要自行取代為正確的值。例如: `<NETWORK_IP>` 就需要取代成**正確的網路介面卡 IP**

## 全自動化懶人安裝法

- [使用 Ansible 在 Rocky Linux 上安裝 k8s](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/)

## 手動安裝

1. 設定機器域名與名稱

    > 這段其實非必要，只是方便測試時不用一直打 IP。

    > 執行指令前請先確認網路是否已經完成設定，特別是 IP 部分，請盡量不要使用 DHCP 自動派發

    > **&lt;HOSTNAME&gt; 需符合 RFC 1123 DNS 標籤的規範**

    ```console
    # hostnamectl set-hostname <HOSTNAME>
    # echo <NETWORK_IP> <HOSTNAME> <COMPUTER_NAME> >> /etc/hosts
    ```

2. 更新系統所有套件至最新

    ```console
    # dnf update --refresh -y
    ```

3. 關閉 SELinux (**不推薦**)

    > **非常不推薦**關閉 SELinux，這可能會使伺服器暴露於危險之中，應當保持其開啟，並於需要時設定其策略

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

9. 安裝 crio
    - 第一種方式 (推薦)

    ```console
    # mkdir /usr/local/bin/runc
    # curl https://raw.githubusercontent.com/cri-o/cri-o/main/scripts/get | bash
    # systemctl enable --now crio
    ```

    - 第二種方式

    ```console
    # VERSION=<預計要安裝的 K8s 版本>
    # curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo
    # curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_8/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo
    # dnf install cri-o cri-tools -y
    # systemctl enable --now crio
    ```

10. 新增 Repo list

    > 請將 `<KUBERNETS_VERSION>` 取代為您的 Kubernetes 版本號碼，例如: `1.26`

    ```console
    # cat > /etc/yum.repos.d/kubernetes.repo << EOF
    > [kubernetes]
    > name=Kubernetes
    > baseurl=https://pkgs.k8s.io/core:/stable:/v<KUBERNETS_VERSION>/rpm/
    > enabled=1
    > gpgcheck=1
    > gpgkey=https://pkgs.k8s.io/core:/stable:/v<KUBERNETS_VERSION>/rpm/repodata/repomd.xml.key
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

    > 網址中的 `latest` 為版本號碼，可以到[這邊](https://github.com/projectcalico/calico/releases)檢視目前的版號為何

    > 請注意，版號若與 Calico 版本不同，下指令會變得非常麻煩，甚至有部署設定失敗的可能性

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

    > 請注意，這邊修改完必須重新開機後才會生效

    ```console
    # cat > /etc/NetworkManager/conf.d/calico.conf << EOF
    > [keyfile]
    > unmanaged-devices=interface-name:cali*;interface-name:tunl*;interface-name:vxlan.calico;interface-name:vxlan-v6.calico;interface-name:wireguard.cali;interface-name:wg-v6.cali
    > EOF
    ```

20. 安裝 Calico 當作 CNI

    > 使用雲服務可以跳過此步驟，雲服務供應商會有自己的 CNI 介面

    > 這邊都是使用「低於 50 台節點的設定檔」，如需大於 50 台節點的設定檔，請至[官方網站下載](https://docs.tigera.io/calico/3.25/getting-started/kubernetes/self-managed-onprem/onpremises#install-calico)

    > `calicoctl` 與 Calico` 的版本請盡量一致，否則下指令時會變得非常麻煩，甚至有部署失敗的可能性

    > `v3.25.0` 這是安裝的版號，可以到[這邊](https://github.com/projectcalico/calico/releases)檢視可以使用的版號

    - 下載 kubectl 的 yaml 檔案

    ```console
    $ curl https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml -O
    ```

    - 若前面的 `<POD_NETWORK_CIDR>` 不是設定為 `192.168.0.0/16`，請在這邊打開文件搜尋 `CALICO_IPV4POOL_CIDR`，把該段 env 註解打開後修改為自己的 Pod 網段即可

    > 官方表示即便不改，Calico 也會自動偵測，如果不放心的話還是改一下比較妥當

    - 執行 `kubectl apply -f calico.yaml` 套用設定檔到 Kubernetes，待所有 Pod 都 Up 後就算安裝完成

    > 安裝完成後部分 Pod 的 IP 可能不是正確的，請利用 Deployment 等方式重啟 Pod 即可。

21. 安裝 Helm

    > 此步驟非必要步驟，如果只想使用 kubectl 工具進行 Kubernetes 相關管理，可以跳過此步驟

    > Helm 在更新 nginx-ingress 等功能時非常實用，比起使用 kubectl 去更新還出一堆錯誤，不如使用 Helm 請它幫你管理

    > 下面這行也可以到[官方文檔中複製](https://helm.sh/docs/intro/install/#from-script)

    ```console
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    ```

22. 部署 nginx ingress controller

    1. 使用 Helm 安裝
        > 利用 Helm 安裝的 Service 中 `externalTrafficPolicy` 預設為 `Cluster`，這會導致入站 IP 全部都是 Node 的 IP，若要看到外部 IP，請將其設為 `Local`

        > 另外 TCP 與 UDP Service 暴露設定也可以在 Helm 中直接設定

        - 使用 Bitnami 提供的 Repo 安裝

            > 使用 Repository 的方式未來升級版本會比較方便

            > 使用 Bitnami 的 Repo 進行安裝會額外安裝 `default-backend`，用以接受所有不符合 Ingress 規則的入站流量

            ```console
            $ helm repo add bitnami https://charts.bitnami.com/bitnami
            $ helm install ingress-nginx-controller bitnami/nginx-ingress-controller \
                --namespace ingress-nginx --create-namespace
            ```

        - 使用官方提供的指令

            > 這條指令如果已經安裝過 nginx-ingress，則它會進行更新，若未安裝過，則會進行安裝

            ```console
            $ helm upgrade --install ingress-nginx ingress-nginx \
                --repo https://kubernetes.github.io/ingress-nginx \
                --namespace ingress-nginx --create-namespace
            ```

    2. 使用 kubectl 安裝
        > ※ 建議每次都從[官方文件](https://kubernetes.github.io/ingress-nginx/deploy/)中複製 yaml 檔網址，以確保 ingress 版本是最新的穩定版本

        > kubectl 安裝方式在更新版本時非常麻煩，且不穩定，推薦使用 Helm 安裝

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

            - 安裝
                1. 使用 Helm 安裝
                    > 使用 Helm 安裝日後更新時會比較快速且穩定

                    ```console
                    $ helm repo add metallb https://metallb.github.io/metallb
                    $ helm install metallb metallb/metallb \
                        --namespace metallb-system --create-namespace
                    ```

                2. 使用 kubectl 安裝
                    > kubectl 安裝方式在更新版本時非常麻煩，且不穩定，推薦使用 Helm 安裝

                    > 使用下面指令或[到官方網站上複製指令](https://metallb.universe.tf/installation/#installation-by-manifest)安裝 MetalLB

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

24. 設定 Calico 的網路策略

    > 所有與 calico 相關的網路策略皆須使用 `calicoctl apply -f <FULL_PATH_TO_FILE>` 套用才會生效

    - 設定開啟的 Port 避免被意外切斷而無法提供服務，以下的埠號由[官方文件提供](https://docs.tigera.io/calico/3.25/network-policy/hosts/protect-hosts#avoid-accidentally-cutting-all-host-connectivity)

        > 10251 與 10252 連接埠可以看[這篇文章](https://stackoverflow.com/a/67397857)

        > 建議先以指令 `calicoctl get felixconfiguration default --export -o yaml > default-felix-config.yaml` 取出目前設定檔內容再進行修改

        ```yaml
        # default-felix-config.yaml
        apiVersion: projectcalico.org/v3
        kind: FelixConfiguration
        metadata:
          name: default
        spec:
          bpfLogLevel: ''
          floatingIPs: Disabled
          logSeverityScreen: Info
          reportingInterval: 0s
          ipipEnabled: true
          Ipv6Support: true
          # 如果需要開啟應用程式層級策略，請將下面的註解打開
          # 網路策略如果是用路徑 (path) 決定可不可以連入者須開啟
          # policySyncPathPrefix: '/var/run/nodeagent'
          # 如需允許容器到本機的流量，請將註解打開
          # defaultEndpointToHostAction: Accept
          # 允許入站流量 (in-bound)
          FailsafeInboundHostPorts:
            - protocol: tcp
              port: 22
            - protocol: udp
              port: 68
            - protocol: tcp
              port: 179
            - protocol: tcp
              port: 2379
            - protocol: tcp
              port: 2380
            # 若有 RDP 遠端桌面需求，請打開註解
            # - protocol: tcp
            #   port: 3389
            - protocol: tcp
              port: 5473
            - protocol: tcp
              port: 6443
            - protocol: tcp
              port: 6666
            - protocol: tcp
              port: 6667
          # 允許出站流量 (out-bound)
          FailsafeOutboundHostPorts:
            - protocol: udp
              port: 53
            - protocol: udp
              port: 67
            - protocol: tcp
              port: 179
            - protocol: tcp
              port: 2379
            - protocol: tcp
              port: 2380
            - protocol: tcp
              port: 5473
            - protocol: tcp
              port: 6443
            - protocol: tcp
              port: 6666
            - protocol: tcp
              port: 6667
        ```

    - 測試 Calico 全域網路策略

        1. 先套用以下策略測試 Calico 全域網路策略是否正常運作

            > 請務必先執行 Failsafe 設定，也就是前一大項的設定，否則策略套用後可能造成 SSH 服務中斷

            > 請將以下內容存成 yaml 檔案，方便後續清除策略

            ```yaml
            # calico-allow-internal-traffic.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: allow-cluster-internal-ingress
            spec:
              order: 10
              # 指定於 kubernetes NAT 前執行策略
              preDNAT: true
              applyOnForward: true
              # 允許內部入站流量
              ingress:
                - action: Allow
                  source:
                    # 這邊需要允許兩個內部 IP 範圍，一個是節點的 IP，另一個是 Pod 的 IP
                    # 使用 <IP_ADDR>/<CIDR> 格式撰寫
                    nets: [10.0.2.15/32, 192.168.0.0/16]
              selector: has(host-endpoint)

            ---
            # calico-drop-external-inbound-traffic.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: drop-other-ingress
            spec:
              order: 20
              preDNAT: true
              applyOnForward: true
              # 禁止所有入站流量
              ingress:
                - action: Deny
              selector: has(host-endpoint)

            ---
            # calico-outbound-external-traffic.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: deny-outbound-policy
            spec:
              order: 10
              egress:
                - action: Deny
              selector: has(host-endpoint)

            ---
            # calico-host-endpoints.yaml
            apiVersion: projectcalico.org/v3
            kind: HostEndpoint
            metadata:
              name: host-endpoints-config
              labels:
                host-endpoint: ingress
            spec:
              # 使用 ifconfig 檢視可連上網際網路的網路介面卡名稱
              interfaceName: <NETWORK_INTERFACE_NAME>
              # 使用 kubectl describe node 檢視 node 名稱
              node: <NODE_NAME>
            ```

        2. 以 curl 進行測試
            - 以 `curl https://www.google.com.tw/` 測試是否可以正常連上 Google，若無法連上，表示策略正常運作
            - 以 `curl http://<ANY_POD_IP_IN_CLUSTER>/` 測試是否可以正常連上任一的 Pod，若可以連上，表示策略正常運作
        3. 執行 `calicoctl delete -f <TEST_CALICO_FILE_NAME>` 清除策略

        > 清除策略後請務必再次進行 curl 連線 Google 確保這策略是否正常被清除

    - 設定全域網路策略

        > 以下部分可以全部組成一個 yaml 檔案進行套用

        1. 允許內部流量，並將所有外部流量拒絕掉

            ```yaml
            # calico-allow-internal-traffic.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: allow-cluster-internal-ingress
            spec:
              order: 10
              # 指定於 kubernetes NAT 前執行策略
              preDNAT: true
              applyOnForward: true
              # 允許內部入站流量
              ingress:
                - action: Allow
                  source:
                    # 這邊需要允許兩個內部 IP 範圍，一個是節點的 IP，另一個是 Pod 的 IP
                    # 使用 <IP_ADDR>/<CIDR> 格式撰寫
                    nets: [10.0.2.15/32, 192.168.0.0/16]
              selector: has(host-endpoint)
            ---
            # calico-drop-external-inbound-traffic.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: drop-other-ingress
            spec:
              order: 20
              preDNAT: true
              applyOnForward: true
              # 禁止所有入站流量
              ingress:
                - action: Deny
              selector: has(host-endpoint)
            ```

        2. 允許所有出站流量

            ```yaml
            # calico-outbound-external-traffic.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: <OUTBOUND_POLICY_NAME>
            spec:
              order: 10
              egress:
                - action: Allow
              selector: has(host-endpoint)
            ```

        3. 套用 Host Endpoints

            ```yaml
            # calico-host-endpoints.yaml
            apiVersion: projectcalico.org/v3
            kind: HostEndpoint
            metadata:
              name: <HOST_ENDPOINT_NAME>
              labels:
                host-endpoint: ingress
                # 方面後續套用 Dos 防禦規則使用
                apply-dos-mitigation: 'true'
            spec:
              # 使用 ifconfig 檢視可連上網際網路的網路介面卡名稱
              interfaceName: <NETWORK_INTERFACE_NAME>
              # 使用 kubectl describe node 檢視 node 名稱
              node: <NODE_NAME>
              # 設定此網路介面上預期的 IP 位址
              expectedIPs: [<IP>]
            ```

        4. 設定以 NodePort 暴露的服務

            > 如要指定特定的節點 (Node) 才可以存取，請在 `metadata.label` 加上 `host-endpoint: <SPECIFIC_NODE_NAME>` 宣告

            ```yaml
            # node-port-inbound-policy.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: <NODEPORT_INBOUND_POLICY_NAME>
            spec:
              preDNAT: true
              applyOnForward: true
              order: 10
              # 允許特定 Port 的 TCP 入站流量
              ingress:
                - action: Allow
                  protocol: TCP
                  destination:
                    selector: has(host-endpoint)
                    ports: [<SPECIFIC_PORT>]
              selector: has(host-endpoint)
            ```

        5. 設定防止 DoS 攻擊的策略

            ```yaml
            # dos-deny-list.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkSet
            metadata:
              name: dos-mitigation
              labels:
                dos-deny-list: 'true'
            spec:
              # 設定要拒絕連線的 IP CIDR
              nets:
                - <DISALLOWED_IP_CIDR_1>
                - <DISALLOWED_IP_CIDR_2>
                - ...

            ---
            # dos-mitigation-policy.yaml
            apiVersion: projectcalico.org/v3
            kind: GlobalNetworkPolicy
            metadata:
              name: dos-mitigation
            spec:
              selector: apply-dos-mitigation == 'true'
              doNotTrack: true
              applyOnForward: true
              types:
                - Ingress
              ingress:
                - action: Deny
                  source:
                    selector: dos-deny-list == 'true'
            ```

25. 部署 emberstack/kubernetes-reflector
    > 若不使用 cert-manager 或不需使用反射 (複製) secret 的話可以跳過此步驟

    > 由於 secret 不可跨 namespace，因故須使用此套件讓 cert-manager 可以把憑證複製到不同的 namespace 中

    > 可以參閱[官方文件](https://github.com/emberstack/kubernetes-reflector)

    ```console
    $ helm repo add emberstack https://emberstack.github.io/helm-charts
    $ helm repo update
    $ helm upgrade --install \
        reflector emberstack/reflector \
        --namespace k8s-reflector \
        --create-namespace
    ```

26. 部署 cert-manager 進行 TLS 憑證自動更新

    > 請注意，若使用 Let's Encrypt 的 HTTP01 驗證，需開啟連接埠 80

    1. 安裝 cert-manager
        - 使用 Helm
            > 使用 Helm 安裝日後更新時會比較快速且穩定

            ```console
            $ helm repo add jetstack https://charts.jetstack.io
            $ helm repo update
            $ helm install \
                cert-manager jetstack/cert-manager \
                --namespace cert-manager \
                --create-namespace \
                --version v1.11.0 \
                --set installCRDs=true
            ```

        - 使用 kubectl
            > kubectl 安裝方式在更新版本時非常麻煩，且不穩定，推薦使用 Helm 安裝

            ```console
            $ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.yaml
            ```

            > Google Kubernetes Engine 可能會發生權限錯誤的問題，如使用 GKE 無法安裝，請接著套用下面的指令

            ```console
            kubectl create clusterrolebinding cluster-admin-binding \
                --clusterrole=cluster-admin \
                --user=$(gcloud config get-value core/account)
            ```

    2. 建立 Issuer 和 Certificate 宣告，並以 `kubectl apply -f <YAML_NAME>` 套用

        > 測試請使用 staging 伺服器以避免超過申請限制

        > 完成測試後要進行正式申請前，除了要把伺服器網址換掉外，先執行 `kubectl delete` 將先前的 issuer 和 certificate 宣告移除後，**將測試申請的無效憑證從 secret 移除**，再執行一次 `kubectl apply` 後，cert-manager 才會進行正式的憑證申請

        > 頂級網域的擁有者不是自己的子網域 (例: *.freedynamicdns.net 這類 no-ip 服務申請到的域名)，其憑證宣告需逐個網域宣告

        ```yaml
        # issuer.yaml
        apiVersion: cert-manager.io/v1
        # 如有 namespace 的問題，kind 請改用 ClusterIssuer
        kind: Issuer
        metadata:
          name: <ISSUER_NAME>
          # 這邊統一宣告給 cert-manager
          namespace: cert-manager
        spec:
          acme:
            # ACME 伺服器
            # 測試請使用 staging 伺服器以避免超過申請限制
            server: https://acme-staging-v02.api.letsencrypt.org/directory
            # 正式申請請將上面這行註解掉後打開下一行的註解
            # server: https://acme-v02.api.letsencrypt.org/directory
            email: <YOUR_EMAIL_ADDRESS>
            # ACME 註冊後儲存帳號相關資訊的 secret 名稱
            privateKeySecretRef:
              name: <TLS_SECRET_NAME>
            # 啟用 HTTP-01 挑戰提供者
            solvers:
              # selector 留空表示此 solver 處理所有的域名
              - selector: {}
                http01:
                  ingress:
                    class: nginx

        ---
        # certificate.yaml
        apiVersion: cert-manager.io/v1
        kind: Certificate
        metadata:
          # 這個可以用域名當名稱，例: example-com
          name: <CERTIFICATE_NAME>
          # 這邊統一宣告給 cert-manager
          namespace: cert-manager
        spec:
          secretName: <TLS_SECRET_NAME>
          issuerRef:
            name: <ISSUER_NAME>
            # 如果這個 Issuer 種類是 ClusterIssuer，請把下面這行註解打開
            # kind: ClusterIssuer
          # 如有頂級域名請宣告在這邊，只有次級域名這邊就直接宣告次級域名即可
          commonName: <YOUR_DOMAIN>
          # 子域名才宣告在這邊
          dnsNames:
            - <SUB_DOMAIN_1>
            - <SUB_DOMAIN_2>
            - ...
          secretTemplate:
            annotations:
              # 允許反射 (複製)
              reflector.v1.k8s.emberstack.com/reflection-allowed: "true"
              # 控制要允許反射的目標 namespace，僅允許逗號分隔的字串或正規表達式
              # 這邊宣告的 namespace 若沒有先建立，則不會被反射
              reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: <NAMESPACES>
              # 啟用自動反射
              reflector.v1.k8s.emberstack.com/reflection-auto-enabled: "true"
              # 控制自動反射的 namespace，需要包含在 reflection-allowed-namespaces 設定中
              reflector.v1.k8s.emberstack.com/reflection-auto-namespaces: <NAMESPACES>
        ```

    3. Calico 新增允許 80/TCP 流量

        > 如果前面的網路策略有儲存成一份 yaml 檔，請直接修改它後進行套用，如果沒有請先取出目前的設定值並修改後再進行套用，**否則既有的策略可能會被洗掉**

        > 重申一遍，套用網路策略請一律以 `calicoctl apply -f <POLICY_FILE>` 進行套用，避免策略沒有成功套用

        > 除了 Calico 外，若有路由器或防火牆，請記得也要允許 80/TCP 的流量

        ```yaml
        # node-port-inbound-policy.yaml
        apiVersion: projectcalico.org/v3
        kind: GlobalNetworkPolicy
        metadata:
          name: <NODEPORT_INBOUND_POLICY_NAME>
        spec:
          preDNAT: true
          applyOnForward: true
          order: 10
          # 允許特定 Port 的 TCP 入站流量
          ingress:
            - action: Allow
              protocol: TCP
              destination:
                selector: has(host-endpoint)
                ports: [80]
          selector: has(host-endpoint)
        ```

    4. 使用指令 `kubectl get challenges --all-namespaces` 看目前 ACME 挑戰的狀態，直至成功為止
        > cert-manager 會起一個 solver 的 Pod、Service 和 Ingress，完成後這些資源都會被自動移除

        > 挑戰成功的話，secret 會有 `tls.crt` 和 `tls.key` 兩個鍵值對，未成功前只會有 `tls.key` 一個鍵值對

        > 挑戰成功之後，該憑證會在過期前 30 天自動進行 renew 的動作

27. 部署服務與相關設定 (Deployments、Service、Ingress、ConfigMap、SecretMap)

    > ※ 請記得先將映像 (image) 推到指定的 Registry 中，否則部署後 Pod 將無法正常運作

    > ※ 若 ingress-nginx 無法透過 Pod 的 IP 連上 Pod (未安裝 CNI 或未關閉防火牆)，請將下面這行加到該 App 的 Ingress 的 annotation 中

    ```yaml
    annotation:
      ...
      nginx.ingress.kubernetes.io/service-upstream=true
      ...
    ```

28. 設定 registry 的登入帳號密碼

    > 由於 Secrets 無法跨命名空間 (namespace) 使用，故如有多個命名空間，每個命名空間都需要部署一份 Secrets

    > 雖然 `emberstack/kubernetes-reflector` 可以反射 secret 至不同的 namespace，但後續仍要繫結 secret 與 service account，因此不建議使用反射

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

            > 請注意，Secrets 請務必與 Deployment 部署的 Pod 所屬的命名空間相同，否則會讀不到

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

## 更新軟體包倉庫位址

Kubernete 官方於 2023/08/31 公告由 Google 所維護的倉庫將於 2023/09/13 被凍結 ([參考此文章](https://kubernetes.io/zh-cn/blog/2023/08/31/legacy-package-repository-deprecation/)) 因此需要針對轉體包倉庫的位址進行更新，其更新步驟如下:

- 先透過 `kubectl version --short` 找到目前安裝的 Kubernetes 版本
- 在 `/etc/yum.repos/` 資料夾下找到 `kubernetes.repo` 檔
- 透過文字編輯軟體打開該 repo 檔案 (由於是系統檔案，需使用 sudo 或 root 權限)
- 將其內容調整為如下

> 請將 `<KUBERNETS_VERSION>` 取代為您的 Kubernetes 版本號碼，例如: `1.26`

```repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v<KUBERNETS_VERSION>/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v<KUBERNETS_VERSION>/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
```

- 完成

## 更新 kubeadm 叢集/自簽憑證

若是自行安裝的 K8s ，其 apiserver 等服務所使用的憑證皆為自簽憑證，效期為一年，因此每隔一年就需要進行自簽憑證的重新簽發，可以透過以下的方式進行：

1. 每至少一年更新一次 Control Plane，更新時就會自動重新簽發新的自簽憑證 (推薦)。
    - 先透過以下指令檢視目前 K8s 版本為何

    ```console
    $ kubectl version --short
    ```

    - 透過以下指令查詢目前最新版本的版本號碼

    ```console
    # dnf list --showduplicates kubeadm --disableexcludes=kubernetes
    ```

    - 先透過以下指令將結點騰空

        > 將 `<NODE_TO_DRAIN>` 取代為你要騰空的控制面節點名稱

    ```console
    # kubectl drain <NODE_TO_DRAIN> --ignore-daemonsets
    ```

    - 執行下面指令更新 kubeadm 版本
        > `<VERSION>` 請取代為你要更新的版本號碼，例如: `1.26.12`

    ```console
    # dnf install -y kubeadm-'<VERSION>-*' --disableexcludes=kubernetes
    ```

    - 透過下面指令驗證安裝的 kubeadm 版本

    ```console
    $ kubeadm version
    ```

    - 透過下面指令驗證升級計畫

    ```console
    # kubeadm upgrade plan
    ```

    - 執行下面指令套用升級

        > `<VERSION>` 請取代為你要更新的版本號碼，例如: `1.26.12`

    ```console
    # kubeadm upgrade apply v<VERSION>
    ```

    - 待指令成功執行後，會看到類似下面的輸出，表示升級成功

    ```console
    [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.29.x". Enjoy!

    [upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
    ```

    - 針對有一個以上的控制平面節點，第一個以後的節點請使用指令 `sudo kubeadm upgrade node` 進行升級
    - 透過下面指令升級 kubelet 與 kubectl

        > `<VERSION>` 請取代為你要更新的版本號碼，例如: `1.26.12`

        > 這邊的版本號碼請盡量與 kubeadm 的版本號碼一致

    ```console
    # dnf install -y kubelet-'<VERSION>-*' kubectl-'<VERSION>-*' --disableexcludes=kubernetes
    ```

    - 透過以下指令重啟 `kubelet`

    ```console
    $ sudo systemctl daemon-reload
    $ sudo systemctl restart kubelet
    ```

    - 執行以下兩個指令更新 `~/.kube/config` 中的憑證

    ```console
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

    - 解除節點的保護

        > 將 `<NODE_TO_DRAIN>` 取代為你的節點名稱

    ```console
    $ kubectl uncordon <NODE_TO_DRAIN>
    ```

2. 手動重新簽發憑證
    - 透過 SSH 連線到 K8s 的節點
    - 先透過指令 `kubeadm certs check-expiration` 檢查目前的自簽憑證狀態
    - 若已過期，可以透過指令 `kubeadm certs renew all` 重新簽發所有自簽憑證
    - 再透過指令 `systemctl restart kubelet` 重新啟動 kubelet 服務，始之套用這些憑證
    - 執行以下兩個指令更新 `~/.kube/config` 中的憑證

    ```console
    $ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```

    - 完成

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
- [Protect hosts tutorial](https://docs.tigera.io/calico/3.25/network-policy/hosts/protect-hosts-tutorial)
- [Calico System requirements](https://docs.tigera.io/calico/3.25/getting-started/kubernetes/requirements)
- [Configure NetworkManager](https://docs.tigera.io/calico/3.25/operations/troubleshoot/troubleshooting#configure-networkmanager)
- [Felix configuration - Calico](https://docs.tigera.io/calico/3.25/reference/resources/felixconfig)
- [Protect hosts - Calico](https://docs.tigera.io/calico/3.25/network-policy/hosts/protect-hosts)
- [Calico Network Policy介紹](https://hackmd.io/@yansheng133/BJTyrfK2Y)
- [Defend against DoS attacks](https://docs.tigera.io/calico/3.25/network-policy/extreme-traffic/defend-dos-attack)
- [cert-manager](https://cert-manager.io/docs/installation/helm/)
- [emberstack/kubernetes-reflector](https://github.com/emberstack/kubernetes-reflector)
- [HTTP Validation](https://cert-manager.io/docs/tutorials/acme/http-validation/)
- [在kubernetes上使用cert-manager自動更新Let’s Encrypt TLS憑證](https://medium.com/@kenchen_57904/%E5%9C%A8kubernetes%E4%B8%8A%E4%BD%BF%E7%94%A8cert-manager%E8%87%AA%E5%8B%95%E6%9B%B4%E6%96%B0lets-encrypt-tls%E6%86%91%E8%AD%89-834b65d43c96)
- [Is it possible to specify http01 port for acme-challenge?](https://github.com/cert-manager/cert-manager/issues/2131)
- [Let's Encrypt 速率限制](https://letsencrypt.org/zh-tw/docs/rate-limits/)
- [Nginx ingress sends private IP for X-Real-IP to services](https://stackoverflow.com/a/68347429)
- [Kubernetes 舊版軟體包倉庫將於 2023 年 9 月 13 日被凍結](https://kubernetes.io/zh-cn/blog/2023/08/31/legacy-package-repository-deprecation/)
- [pkgs.k8s.io：介紹 Kubernetes 社區自有的包倉庫](https://kubernetes.io/zh-cn/blog/2023/08/15/pkgs-k8s-io-introduction/)
- [使用 kubeadm 進行憑證管理](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- [Restart kube-apiserver when provisioned with kubeadm](https://stackoverflow.com/a/42722258)
- [升級 kubeadm 叢集](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
