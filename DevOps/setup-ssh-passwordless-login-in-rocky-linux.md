# 使用公鑰登入 Rocky Linux

此文件包含 `使用 Pubkey 登入 Rocky Linux` 與 `關閉密碼驗證` 兩種說明。

## 設定 SSH 使用公鑰驗證身分

> ※ Windows 系統若無 ssh-keygen，請直接安裝 git 後使用 git-bash 執行下面所有指令

1. 先在本機新建一個 SSH 的公鑰與私鑰

    ```console
    $ ssh-keygen
    ```

2. ssh-keygen 詢問鑰匙儲存位置，按 ENTER 預設即可
3. ssh-keygen 詢問鑰匙密碼，如要建立密碼給公鑰與私鑰，可以在這邊設定，否則按兩次 ENTER 跳過即可
4. 將產生的公鑰放到遠端伺服器上

    ```console
    ssh-copy-id <USERNAME>@<REMOTE_SERVER_IP_OR_DOMAIN>
    ```

    > ※ 若需要將公鑰放到如 GitHub 上，將 `id_rsa.pub` 檔案的內容全部貼到該服務供應商的設定內即可

    > ※ `id_rsa.pub` 儲存位置為 2. 中的位置，預設儲存位置為 `$HOME/.ssh` 資料夾

5. 完成，可以使用 SSH 嘗試登入遠端伺服器，驗證設定是否有效

## 關閉 SSH 除公鑰驗證外的其餘驗證方式

1. 先確定是否已經完成公鑰驗證的設定，否則完成此項設定將無法再以密碼方式遠端 SSH 登入伺服器
2. 遠端登入目標伺服器
3. 編輯 `/etc/ssh/sshd_config` 檔案，將除 `PubkeyAuthentication` 外的所有驗證全部設為 `no`
4. 儲存檔案後重啟 sshd 服務

    ```console
    # systemctl restart sshd.service
    ```

5. 完成

## 參考資料

- [[教學] 產生SSH Key並且透過KEY進行免密碼登入](https://xenby.com/b/220-%E6%95%99%E5%AD%B8-%E7%94%A2%E7%94%9Fssh-key%E4%B8%A6%E4%B8%94%E9%80%8F%E9%81%8Ekey%E9%80%B2%E8%A1%8C%E5%85%8D%E5%AF%86%E7%A2%BC%E7%99%BB%E5%85%A5)
- [How To Setup SSH Passwordless Login in Rocky Linux](https://www.linuxshelltips.com/ssh-passwordless-login-rocky-linux/)
- [How do I force SSH to only allow users with a key to log in?](https://askubuntu.com/a/346863)
- [SSH and locked users](http://arlimus.github.io/articles/usepam/)
