# Host Private Registry On Podman

要在 Podman 上架設私有映像儲存庫 (Private Image Registry) 請依據下方步驟進行安裝

※ 所有的指令前綴為 `$` 表不需要 root 權限， `#` 則需要 root 權限

## 手動安裝

1. 建立儲存映像的資料夾，並設定其擁有者

    ```console
    $ sudo mkdir -p /var/lib/registry
    $ sudo chown -R $USER:$USER /var/lib/registry
    ```

2. 執行 registry 容器

    ```console
    $ sudo podman run --privileged -d --name registry -p 5000:5000 -v /var/lib/registry:/var/lib/registry --restart=always registry:2
    ```

## 讓 Podman 在系統啟動時會自動重啟容器

※ 使用 Podman 架設 Registry 請務必執行此指令，否則機器重啟後 Podman 不會自動重啟所有的容器服務

    ```console
    $ sudo systemctl enable podman-restart
    ```

## 推送映像檔

1. 先將要推送的映像檔加上標籤

    ※ 映像檔名稱可以利用指令 `podman images` 列出所有已存在的映像檔確認

    ```console
    $ podman tag <ORIGINAL_IMAGE_NAME_WITH_DOMAIN> localhost:5000/<IMAGE_NAME>
    ```

2. 推送映像

    ※ 若私有的 Registry 有 SSL，則不須加上 `--tls-verify=false` 參數

    ```console
    $ podman push localhost:5000/<IMAGE_NAME> --tls-verify=false
    ```

## 列出目前私有映像儲存庫中既有的映像清單

※ 通常若 K8s 的 Pod 拉不到映像檔，可以先用此指令確認 Registry 是否還活著

    ```console
    $ curl -X GET http://localhost:5000/v2/_catalog
    ```

## 參考資料

- [在 Rocky Linux 上執行私有映像儲存庫 (registry)](https://thenewstack.io/tutorial-host-a-local-podman-image-registry/)
- [列出私有映像儲存庫中的映象檔](https://stackoverflow.com/questions/31251356/how-to-get-a-list-of-images-on-docker-registry-v2)
- [Podman 開機後自動重啟容器](https://github.com/containers/podman/issues/10539#issuecomment-1279750679)
