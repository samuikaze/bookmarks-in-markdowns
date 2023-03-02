# Install VirtualBox Guest Additions On Rocky Linux 9

要在 Rocky Linux 上安裝 VirtualBox Guest Additions 請依據下列步驟進行安裝

> ※ 所有的指令前綴為 `$` 表不需要 root 權限， `#` 則需要 root 權限

## 安裝 VirtualBox Guest Additions

1. 更新系統至最新

    ```console
    # dnf update --refresh -y
    ```

2. 安裝 epel-release

    ```console
    # dnf install epel-release -y
    ```

3. 安裝必要元件

    ```console
    # dnf install dkms kernel-devel kernel-headers gcc make bzip2 perl elfutils-libelf-devel -y
    ```

4. 更新核心模組

    ```console
    # dnf update kernel-*
    ```

5. 重新啟動系統

    ```console
    # reboot
    ```

6. 從 VirtualBox 選單插入 VirtualBox Guest Additions 安裝映像檔
7. 完成

## 參考資料

- [Install VirtualBox Guest Additions on Rocky Linux 9](https://kifarunix.com/install-virtualbox-guest-additions-on-rocky-linux-9/)
