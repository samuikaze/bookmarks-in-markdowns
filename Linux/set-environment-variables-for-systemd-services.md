# 在 Service 中使用環境變數

要在 systemd 的服務中使用環境變數，全域環境變數或是 bash 的變數預設是不會被載入的，需依據以下步驟設定

## 設定環境變數

1. 設定給**所有**服務

    設定給所有服務的環境變數，編輯 `/etc/systemd/system.conf`，使用以下的格式進行設定

    ```console
    # vim /etc/systemd/system.conf
    ```

    ```plaintext
    DefaultEnvironment="VAR1=test1" "VAR2=test2"
    ```

2. 設定給指定的服務

    假定目前我們有一個服務是 `test.service`，其完整路徑為 `/etc/systemd/system/test.service`，則可依據以下步驟進行環境變數設定

    ```console
    # vim /etc/systemd/system/test.service
    ```

    在 [Service] 區塊中加入以下設定

    ```plaintext
    [Service]
    Environment="VAR1=test1" "VAR2=test2"
    Environment="VAR3=test3"
    ```

## 參考資料

- [How to set environmental variable in systemd service](https://unix.stackexchange.com/a/455283)
- [Set environment variable for all services running under systemd](https://unix.stackexchange.com/a/320570)
- [How to set environment variable in systemd service?](https://serverfault.com/a/413408)
