# Install Kubernetes On Rocky Linux 9

要在 Rocky Linux 9 下安裝 Kubernetes 請依據下方步驟進行安裝

※ 所有的指令前綴為 `$` 表不需要 root 權限， `#` 則需要 root 權限

## 全自動化懶人安裝法

- [使用 Ansible 在 Rocky Linux 上安裝 k8s](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/)

## 手動安裝

1. 設定機器名稱

    ```console
    # hostnamectl set-hostname kubemaster-01.centlinux.com
    # echo 192.168.116.131 kubemaster-01.centlinux.com kubemaster-01 >> /etc/hosts
    ```

2. 更新系統所有套件至最新

    ```console
    # dnf update -y
    ```

3. 關閉 SELinux (可以不關)

    ```console
    # setenforce 0
    # sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
    ```

4. 載入必要的 Linux 核心模組

    ```console
    # modprobe overlay
    # modprobe br_netfilter
    ```

    ```console
    # cat > /etc/modules-load.d/k8s.conf << EOF
    > overlay
    > br_netfilter
    > EOF
    ```

    ```console
    # cat > /etc/sysctl.d/k8s.conf << EOF
    > net.ipv4.ip_forward = 1
    > net.bridge.bridge-nf-call-ip6tables = 1
    > net.bridge.bridge-nf-call-iptables = 1
    > EOF
    ```

5. 套用變更

    ```console
    # sysctl --system
    ```

6. 關閉 Swap

    ```console
    # swapoff -a
    # sed -e '/swap/s/^/#/g' -i /etc/fstab
    ```

7. 驗證 Swap 是否已經關閉

    ```console
    # free -m
    ```

8. 安裝 crio

    ```console
    $ VERSION=1.22
    $ sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable/CentOS_8/devel:kubic:libcontainers:stable.repo
    $ sudo curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:${VERSION}/CentOS_8/devel:kubic:libcontainers:stable:cri-o:${VERSION}.repo
    $ sudo dnf install cri-o cri-tools -y
    $ sudo systemctl enable --now crio
    ```
    
    或
    
    ```console
    curl https://raw.githubusercontent.com/cri-o/cri-o/main/scripts/get | bash -s -- -a arm64
    ```

9. 開啟防火牆指定的 ports

    ```console
    # firewall-cmd --permanent --add-port={6443,2379,2380,10250,10251,10252}/tcp
    # firewall-cmd --reload
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
    $ sudo kubeadm config images pull --cri-socket unix:///var/run/crio/crio.sock
    ```

14. 初始化 kubeadm

    ```console
    sudo kubeadm init \
        --pod-network-cidr=192.168.0.0/16 \
        --cri-socket unix:///var/run/crio/crio.sock
    ```

15. 讓 kubectl 指令可以不用 sudo 執行

    ```console
    $ mkdir -p $HOME/.kube
    $ sudo cp -f /etc/kubernetes/admin.conf $HOME/.kube/config
    $ sudo chown $(id -u):$(id -g) $HOME/.kube/config
    ```
16. 讓 Control Plane 機器也可以部屬 Pod

    ```console
    $ kubectl taint nodes --all node-role.kubernetes.io/master-
    ```

17. 部屬 nginx ingress controller

    ```console
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
    ```

18. LoadBalancer 設定外部 IP
    - 針對 nginx ingress controller 的部屬 yaml 中，spec 新增下面的設定

    ```console
    externalIPs:
    - <機器對外網卡上設定的固定 IP>
    ```

19. 部屬微服務 (Deployments、Service、Ingress、ConfigMap、SecretMap)

## 其它資料

- [刪除不在預設命名空間的服務](https://stackoverflow.com/a/67517905)

## 參考資料

- [使用 Ansible 在 Rocky Linux 上安裝 k8s](https://computingforgeeks.com/install-kubernetes-cluster-on-rocky-linux-with-kubeadm-crio/)
- [手動在 Rocky Linux 上安裝 k8s](https://www.centlinux.com/2022/11/install-kubernetes-master-node-rocky-linux.html)
- [在 Rocky Linux 上安裝 crio](https://computingforgeeks.com/install-cri-o-container-runtime-on-rocky-linux-almalinux/)
- [Crio GitHub](https://github.com/cri-o/cri-o#installing-cri-o)
- [nginx ingress controller](https://kubernetes.github.io/ingress-nginx/deploy/)
- [讓 Control Plane 可以部屬 Pod](https://blog.csdn.net/lisongyue123/article/details/108365127)
- [LoadBalancer 設定外部 IP](https://stackoverflow.com/questions/44110876/kubernetes-service-external-ip-pending/54168660#54168660)
