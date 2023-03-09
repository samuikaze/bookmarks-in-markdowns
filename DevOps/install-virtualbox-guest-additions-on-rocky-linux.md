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

5. 從 VirtualBox 選單插入 VirtualBox Guest Additions 安裝映像檔
6. 重新啟動系統

    ```console
    # reboot
    ```
7. 若有共享資料夾的需求，須執行以下指令，以利存取時不用每次都輸入密碼

    > 請注意，此指令僅適用於目前登入的使用者，如需設定到其他使用者，請將 `$USER` 更改為該使用者的名稱

    ```console
    $ sudo usermod -aG vboxsf $USER
    ```

8. 完成

## 參考資料

- [Install VirtualBox Guest Additions on Rocky Linux 9](https://kifarunix.com/install-virtualbox-guest-additions-on-rocky-linux-9/)
