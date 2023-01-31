# Install Terraform On Rocky Linux 9

要在 Rocky Linux 9 下安裝 Terraform 請依據下方步驟進行安裝

※ 所有的指令前綴為 `$` 表不需要 root 權限， `#` 則需要 root 權限

## 安裝步驟

1. 安裝 yum-utils

    ```console
    $ sudo yum install yum-utils -y
    ```

2. 加入軟體庫

    ```console
    $ sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
    ```

3. 安裝 Terraform

    ```console
    $ sudo dnf install terraform -y
    ```

## 參考資料

- [Install Terraform on CentOS 8 / Rocky Linux 8](https://computingforgeeks.com/how-to-install-terraform-on-centos-linux/)
