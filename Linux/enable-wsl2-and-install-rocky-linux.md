<!-- markdownlint-disable MD028 -->

# Windows 啟用 WSL 2 並安裝 Rocky Linux 發佈版

此文章將會說明如何在 Windows 中啟用 WSL 2，並安裝 Rocky Linux 發佈版

## Table of Contents

- [系統需求](#系統需求)
- [啟用 WSL 2](#啟用-wsl-2)
  - [簡易啟用方式](#簡易啟用方式)
  - [手動啟用方式](#手動啟用方式)
- [安裝 Rocky Linux 到 WSL 2 中](#安裝-rocky-linux-到-wsl-2-中)
  - [安裝 Rocky Linux](#安裝-rocky-linux)
  - [啟動後的基礎設定](#啟動後的基礎設定)
- [WSL 存取 Windows 檔案系統](#wsl-存取-windows-檔案系統)
- [設定 WSL 預設啟動的發佈版](#設定-wsl-預設啟動的發佈版)
- [備份與還原](#備份與還原)
- [VirtualBox 與 WSL 相容](#virtualbox-與-wsl-相容)
- [WSL 常用指令](#wsl-常用指令)
- [參考資料](#參考資料)

## 系統需求

若要使用 WSL 2，須符合以下系統需求

- Windows 10
  - x64 系統需使用 1903 (含)以上版本
  - ARM64 系統需使用 2004 (含)以上版本
- Windows 11 全版本皆支援

更詳細的系統需求資訊，請從[官方文件](https://learn.microsoft.com/zh-tw/windows/wsl/install-manual#step-2---check-requirements-for-running-wsl-2)中檢視

## 啟用 WSL 2

Windows 目前啟用 WSL 2 有兩種方式，下面將會逐一進行說明

### 簡易啟用方式

> 此方式僅較新版的 Windows 支援，若無法使用 `wsl --install` 指令，請使用手動啟用方式啟用 WSL 2

1. 透過 Microsoft Store 安裝[新版的 WSL](https://apps.microsoft.com/store/detail/windows-subsystem-for-linux/9P9TQF7MRM4R)
2. 打開 Windows 設定 -> 系統 -> 選用功能 -> 更多 Windows 功能 -> 將 `Windows 子系統 Linux 版` 打勾
3. 執行指令 `wsl --set-default-version 2` 將 WSL2 設定為預設版本
4. 執行指令 `wsl --install` 指令啟用需要的 Windows 功能與安裝 Ubuntu 發佈版
5. 若有提示需要重新開機則進行重新開機
6. 待完成後，透過指令 `wsl` 就可以開啟 Ubuntu 的子系統

### 手動啟用方式

1. 打開 Powershell 執行以下指令啟用虛擬機器平台

    ```Powershell
    dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
    ```

2. 重新開機
3. 透過 Microsoft Store 安裝[新版的 WSL](https://apps.microsoft.com/store/detail/windows-subsystem-for-linux/9P9TQF7MRM4R)
4. 打開 Windows 設定 -> 系統 -> 選用功能 -> 更多 Windows 功能 -> 將 `Windows 子系統 Linux 版` 打勾
5. 下載並安裝 [WSL2 Linux 核心更新套件](https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi)
    > 若網址失效，請透過[官方文件](https://learn.microsoft.com/zh-tw/windows/wsl/install-manual#step-4---download-the-linux-kernel-update-package)下載
6. 執行指令 `wsl --set-default-version 2` 將 WSL2 設定為預設版本
7. 透過 `Microsoft Store` 下載 Linux 發行版或執行 `wsl --install` 指令安裝 Linux 發行版
    > `wsl --install` 指令預設會安裝 Ubuntu 子系統
8. 若有提示需要重新開機則進行重新開機
9. 待完成後，透過指令 `wsl` 就可以開啟 Ubuntu 的子系統

## 安裝 Rocky Linux 到 WSL 2 中

### 安裝 Rocky Linux

透過以下方式安裝 Rocky Linux 到 WSL 2 中

> 若要使用 `systemd` 啟動 WSL，請見[啟動後的基礎設定](#啟動後的基礎設定)的安裝 `systemd` 說明，在完成該說明的設定前請勿隨意啟用 WSL 設定中的 systemd 設定，否則會造成 WSL 無法啟動

1. 到[官方文件](https://docs.rockylinux.org/guides/interoperability/import_rocky_to_wsl/#steps)中下載所需的版本
    > 若是執行在 WSL 中，建議下載 Base x86_64 版本，Minimal 版本包含的工具較少，較適合使用於提供服務的容器執行

    > 若 WSL 版本非最新版，需先將 `.tar.xz` 檔解壓縮成 `.tar` 檔

    > 此步驟也可以透過 docker 或 podman 指令將執行中的 Rocky Linux 容器匯出成 `tar` 檔
    >
    > ```command
    > # 透過 docker 匯出
    > docker export rockylinux:9 > rocky-9-image.tar
    > # 透過 podman 匯出
    > podman export rockylinux:9 > rocky-9-image.tar
    > ```

2. 建立存放資料的資料夾
3. 透過以下指令將 Rocky Linux 安裝到 WSL 中
    > `<MACHINE_NAME>` 請取代為機器名稱，這個名稱後續會顯示於 `wsl -l -v` 清單中，進入此機器也需要輸入此名稱

    > `<PATH_TO_CREATED_DIRECTORY>` 請取代為 2. 中建立的資料夾完整路徑

    > `<PATH_TO_TAR_FILE>` 請取代為 1. 中下載的 `.tar.xz` 或 `.tar` 檔位置

    ```command
    wsl --import <MACHINE_NAME> <PATH_TO_CREATED_DIRECTORY> <PATH_TO_TAR_FILE> --version 2
    ```

4. 待完成後，透過指令 `wsl -d <MACHINE_NAME>` 就可以開啟 Rocky Linux 的子系統
    > `<MACHINE_NAME>` 請取代為 3. 中指定的機器名稱

### 啟動後的基礎設定

進入 Rocky Linux 會發現其預設使用 root 使用者登入，可以依據以下步驟執行基本的設定

> 下面的步驟沒有一定的順序，其中有些步驟非必要，可以視需求選擇要不要進行設定

- 設定 DNS

  Windows 有安裝防毒軟體或防火牆的情況下，使用預設的 DNS 設定將無法正常連線到網際網路，需透過以下步驟調整 DNS 設定

  1. 執行指令 `vi /etc/resolve.conf` 編輯 DNS 設定
      > vi 可以取代為任意的文字編輯工具
  2. 將原本的 nameserver 註解掉，並新增新的 DNS 設定
  3. 退出 WSL，透過指令 `wsl -t <MACHINE_NAME>` 停止 WSL，並再透過 `wsl -d <MACHINE_NAME>` 重新啟動 WSL
      > `<MACHINE_NAME>` 請取代為 3. 中指定的機器名稱
  4. 完成

- 安裝 `systemd`

  此發佈版預設不包含 `systemd` 套件，因此需要自行透過 dnf 進行安裝，下面是其安裝與起用步驟

  1. 透過以下指令安裝 `systemd`

      ```command
      dnf install systemd -y
      ```

  2. 在 /etc/wsl.conf 中加入以下設定

      ```conf
      [boot]
      systemd=true
      ```

  3. 退出 WSL，透過以下指令重新啟動 WSL

      ```Powershell
      wsl -t <MACHINE_NAME>
      wsl -d <MACHINE_NAME>
      ```

  4. 完成

- 變更語言

  Rocky Linux 預設安裝完後為英文版，若要切換成中文，請依據以下步驟進行設定

  > 請注意，由於 `localectl` 涉及 `systemd` 服務，請先完成 `systemd` 的設定

  1. 透過指令 `locale -a` 查詢目前已安裝的語系
      > 若需求語系已經安裝，可以跳到 5.
  2. 執行指令 `dnf list langpacks-*` 列出所有可安裝的語系
  3. 執行指令 `dnf install langpacks-<LANG_NAME>` 安裝需求語系
      > `<LANG_NAME>` 請取代為 2. 中查詢到的語系名稱
  4. 再透過指令 `locale -a` 查詢目前已安裝的語系
  5. 執行指令 `localectl set-locale LANG=<LANG_NAME>` 切換到需求語系
      > `<LANG_NAME>` 請取代為 1. 或 4. 中查詢到的語系名稱
  6. 若設定為生效，請先退出 WSL，並透過以下指令重新啟動 WSL

      ```Powershell
      wsl -t <MACHINE_NAME>
      wsl -d <MACHINE_NAME>
      ```

- 安裝 `sudo`，並新增預設的登入使用者

  安裝完後預設登入的使用者為 root，可以透過以下方式調整預設登入的使用者，增加系統安全性

  1. 執行指令 `passwd root` 設定 root 的密碼
      > 請務必設定 root 密碼，否則日後將無法再透過指令 `su -` 登入 root 帳號
  2. 執行指令 `dnf install sudo -y` 安裝 `sudo` 工具
  3. 透過以下指令新增使用者，並設定此使用者為 sudoer
      > `<USERNAME>` 請取代為使用者名稱

      > 建議設定完成後先透過指令 `su - <USERNAME>` 登入新的使用者後，測試 sudo 是否正常運作

      ```command
      adduser <USERNAME>
      passwd <USERNAME>
      usermod -aG wheel <USERNAME>
      ```

  4. 執行指令 `vim /etc/wsl.conf` 新增 WSL 設定檔，並寫入以下的設定
      > vim 可以取代為任意的文字編輯工具

      > `<USERNAME>` 請取代為使用者名稱

      ```conf
      [user]
      default=<USERNAME>
      ```

  5. 退出 WSL，透過指令 `wsl -t <MACHINE_NAME>` 停止 WSL，並再透過 `wsl -d <MACHINE_NAME>` 重新啟動 WSL
      > `<MACHINE_NAME>` 請取代為正確的機器名稱，可透過指令 `wsl -l -v` 查閱

  6. 若登入的使用者為前面所設定的使用者，則設定成功

- 安裝 `clear` 指令

  若是透過安裝光碟或 DVD 映像檔進行安裝，`clear` 指令預設會裝在系統中，由於目前此發佈版未包含此指令相關的套件，因此需透過以下指令安裝

  ```command
  dnf install ncurses -y
  ```

- 安裝 `podman` 指令

  透過以下方式安裝 podman 指令

  1. 執行指令 `dnf install podman -y` 安裝 podman
  2. 執行指令 `mount --make-rshared /` 讓 `/` 成為 shared mount
      > 建議將此行加入 `/etc/wsl.conf` 檔中，使其啟動時可以自動執行，如下所示:
      >
      > ```conf
      > [boot]
      > ...
      > command="mount --make-rshared /"
      > ...
      > ```
      >
  3. 執行指令 `dnf reinstall shadow-utils` 重新安裝 `shadow-utils`
  4. 完成

- 安裝 `net-tools`

  若有使用 `ifconfig` 指令的需求，則需安裝此套件，可以執行指令 `dnf install net-tools -y` 進行安裝

## WSL 存取 Windows 檔案系統

WSL 可以透過 `/mnt` 目錄存取 Windows 檔案系統，以 `C:\Users\demouser` 為例，在 WSL 中的路徑為 `/mnt/c/Users/demouser`，其中 `/mnt/c` 表示 C 槽，若要換成其它槽區，請修改其值為該槽區代號的小寫英文字母

> Windows 與 Linux 不同，Linux 的檔案系統是大小寫敏感的，因此路徑除了槽區代號外，其它都必須與原本的名稱相同，例: Windows 中資料夾為 `C:\Users\DemoUser`，WSL 中存取 `/mnt/c/Users/demouser` 是找不到該資料夾的

另外透過 Powershell 執行 `wsl` 指令啟動 WSL 者，進入 WSL 的終端機後，其預設的路徑會是 Powershell 執行 `wsl` 指令時的所在目錄

建議可以在 WSL 中建立類似於下方的腳本快速切換到 Windows 的資料夾

```bash
#!/bin/bash
# changedir.sh

default() {
    cd /mnt/c/Users/demouser/Documents
}

case "$1" in
default)
    default
;;
*)
    echo ""
    echo "To use this script, issue the command below"
    echo ""
    echo "    source changedir.sh [ARGS]"
    echo ""
    echo "    Allowed arguments:"
    echo "        default: Change current directory to C:\\Users\\demouser\\Documents"
    echo ""
;;
esac
```

## 設定 WSL 預設啟動的發佈版

WSL 預設啟動的發佈版會是第一個安裝的發佈版，若要切換預設啟動的發佈版，可以透過以下方式進行設定

1. 執行指令 `wsl -l -v` 查閱目前已經安裝的發佈版名稱
2. 執行指令 `wsl --set-default <DISTRO_NAME>` 將預設啟動的發佈版切換到 `<DISTRO_NAME>`
    > `<DISTRO_NAME>` 請取代為正確的發佈版名稱
3. 可以透過指令 `wsl -l -v` 確認 `*` 字號是否標示在 2. 中設定的發佈版名稱前方
4. 完成，下次可以直接執行 `wsl` 指令啟動指定的發佈版了

## 備份與還原

備份與還原 WSL 可以透過以下兩個指令完成

> `<MACHINE_NAME>` 請取代為正確的機器名稱，可透過指令 `wsl -l -v` 查閱

> `<EXPORT_PATH>` 請取代為 tar 檔路徑

- 備份 WSL 為 tar 壓縮檔

  ```command
  wsl –-export <MACHINE_NAME> <EXPORT_PATH>
  ```

- 從 tar 壓縮檔還原 WSL

  ```command
  wsl -–import <MACHINE_NAME> <EXPORT_PATH>
  ```

## VirtualBox 與 WSL 相容

若系統中有安裝 VirtualBox，且啟用 WSL 的方式是透過 Hyper-V 的方式，則可透過以下方式讓 VirtualBox 與 WSL 相容
> VirtualBox 主要是與 Hyper-V 不相容，因此若啟動 WSL 非使用 Hyper-V 者，不須執行此設定

1. 打開 VirtualBox 安裝資料夾，右鍵打開終端機或複製其路徑，並在終端機中切換到此路徑
2. 執行以下兩個指令其中一個
    - 指定特定的虛擬機器啟用此功能

      > `<VM_NAME>` 請取代為虛擬機器的名稱

      ```Powershell
      ./VBoxManage.exe setextradata "<VM_NAME>" "VBoxInternal/NEM/UseRing0Runloop" 0
      ```

    - 指令所有虛擬機器都啟用此功能

      ```Powershell
      ./VBoxManage.exe setextradata global "VBoxInternal/NEM/UseRing0Runloop" 0
      ```

3. 完成

## WSL 常用指令

- `wsl -l -v`: 查閱目前已安裝的發佈版、執行狀態與 WSL 版本資訊
- `wsl -d <MACHINE_NAME>`: 啟動指定的發佈版
  > `<MACHINE_NAME>` 請取代為正確的機器名稱，可透過指令 `wsl -l -v` 查閱
- `wsl -t <MACHINE_NAME>`: 停止指定的發佈版
- `wsl --unregiser <MACHINE_NAME>`: 移除指定的發佈版
  > 請注意，這會將發佈版中所有資料都移除
- `wsl –-export <MACHINE_NAME> <EXPORT_PATH>`: 備份指定的發佈版
  > `<MACHINE_NAME>` 請取代為正確的機器名稱，可透過指令 `wsl -l -v` 查閱
  > `<EXPORT_PATH>` 請取代為 tar 檔輸出路徑
- `wsl -–import <MACHINE_NAME> <EXPORT_PATH>`: 還原指定的發佈版
  > `<MACHINE_NAME>` 請取代為正確的機器名稱，可透過指令 `wsl -l -v` 查閱
  > `<EXPORT_PATH>` 請取代為 tar 檔路徑

## 參考資料

- [Import Rocky Linux to WSL](https://docs.rockylinux.org/guides/interoperability/import_rocky_to_wsl/)
- [Windows 11 安裝 WSL2](https://hackmd.io/@Kailyn/H1N5OPKlF)
- [Windows 10 安裝 WSL1、WSL2(手動安裝)](https://hackmd.io/@Kailyn/BkMi80IeF)
- [Import any Linux distribution to use with WSL](https://learn.microsoft.com/en-us/windows/wsl/use-custom-distro#import-the-tar-file-into-wsl)
- [rockylinux - Official Image | Docker Hub](https://hub.docker.com/_/rockylinux)
- [How To Create a New Sudo-enabled User on Rocky Linux 8 [Quickstart]](https://www.digitalocean.com/community/tutorials/how-to-create-a-new-sudo-enabled-user-on-rocky-linux-8-quickstart)
- [WSL 的基本命令](https://learn.microsoft.com/zh-tw/windows/wsl/basic-commands)
- [WSL 中的進階設定組態](https://learn.microsoft.com/zh-tw/windows/wsl/wsl-config)
- [Windows 10 (2004) 啟用 wsl2, 並與 VirtualBox 6.0+ 共存](https://entr0pia.github.io/arts/2020-07-22-hyper-V%20and%20VirtualBox.html)
- [備份 WSL](https://hackmd.io/@LHB-0222/WSL2)
- [Installing and Using 'Clear' Command | Linux Guide](https://ioflood.com/blog/install-clear-command-linux/)
- [Podman - Guide 2 WSL](https://www.guide2wsl.com/podman/)
- [Running podman rootless gives ERRO[0000] cannot setup namespace using newuidmap: exit status 1 #2788](https://github.com/containers/podman/issues/2788)
- [Dockerfile - eniocarboni/docker-rockylinux-systemd](https://github.com/eniocarboni/docker-rockylinux-systemd/blob/main/Dockerfile)
- [Systemd support is now available in WSL!](https://devblogs.microsoft.com/commandline/systemd-support-is-now-available-in-wsl/#set-the-systemd-flag-set-in-your-wsl-distro-settings)
- [RHEL 安裝 locale](https://hackmd.io/@yzai/S1vMGJiqq)
- [How to run command when starting a machine in WSL2](https://superuser.com/a/1717840)
- [如何使用 WSL 在 Windows 上安裝 Linux](https://learn.microsoft.com/zh-tw/windows/wsl/install)
- [舊版 WSL 的手動安裝步驟](https://learn.microsoft.com/zh-tw/windows/wsl/install-manual)
