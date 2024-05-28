# 透過 git 架設簡易儲存庫

現在有許多現成的程式碼儲存庫可以使用，但部份程式碼並不適合推送到這類服務上，因此可以透過自行架設私有的程式碼儲存庫進行儲存。

下面將會說明如何透過 git 與相關工具架設簡易的程式碼儲存庫

## Table of Contents

- [事前準備](#事前準備)
- [架設方法](#架設方法)
  - [全新的儲存庫](#全新的儲存庫)
  - [既有的儲存庫](#既有的儲存庫)
- [參考資料](#參考資料)

## 事前準備

首先須先滿足以下條件:

- 伺服器與客戶端皆須安裝 git
  > 客戶端可以使用 Docker 容器方式安裝
- 伺服器需安裝 ssh
- 一組可以連線到 Linux 的使用者帳號與密碼

## 架設方法

下面將會區分為全新的儲存庫與既有儲存庫進行說明，請選擇合適的情境，依據步驟進行實作:

### 全新的儲存庫

1. 透過 ssh 連線到 Linux
2. (選擇性步驟) 建立新的使用者與密碼，作為 push 與 pull 儲存庫使用
3. 在這支帳號可以存取的目錄中建立新的目錄，日後會將所有的儲存庫都放在這邊，這邊使用 `~/self-hosted-repositories`
4. 在這個目錄中建立儲存庫的目錄，這邊以 `example-repository` 作為其名稱
5. 切換到該目錄，執行 `git init --bare` 初始化此儲存庫
6. 透過 `pwd` 指令查詢目前目錄的完整路徑
7. 回到客戶端，透過指令 `git clone ssh://<USER>@<SERVER_DOMAIN_OR_IP>:<DIRECTORY>` 將儲存庫複製下來，其中網址中的參數說明如下:
    - `<USER>`: 登入 Linux 的使用者
    - `<SERVER_DOMAIN_OR_IP>`: Linux 伺服器位址
    - `<DIRECTORY>`: 將剛剛在 Linux 中透過 `pwd` 指令印出的完整路徑貼上來取代此值
8. 完成設定，可以透過 `git push` 與 `git pull` 指令測試伺服器是否正常運作

### 既有的儲存庫

1. 透過 ssh 連線到 Linux
2. (選擇性步驟) 建立新的使用者與密碼，作為 push 與 pull 儲存庫使用
3. 在這支帳號可以存取的目錄中建立新的目錄，日後會將所有的儲存庫都放在這邊，這邊使用 `~/self-hosted-repositories`
4. 在這個目錄中建立儲存庫的目錄，這邊以 `exists-repository` 作為其名稱
5. 切換到該目錄，執行 `git init --bare` 初始化此儲存庫
6. 透過 `pwd` 指令查詢目前目錄的完整路徑
7. 回到客戶端，透過指令 `git remote add <REMOTE_NAME> ssh://<USER>@<SERVER_DOMAIN_OR_IP>:<DIRECTORY>` 設定遠端儲存庫位址，其中網址中的參數說明如下:
    - `<REMOTE_NAME>`: 遠端名稱，通常會使用 `origin`
    - `<USER>`: 登入 Linux 的使用者
    - `<SERVER_DOMAIN_OR_IP>`: Linux 伺服器位址
    - `<DIRECTORY>`: 將剛剛在 Linux 中透過 `pwd` 指令印出的完整路徑貼上來取代此值
8. 透過 `git push -u <REMOTE_NAME> <BRANCH_NAME>` 將分支推送到 Linux 中
9. 完成

## 參考資料

- [4.2 伺服器上的 Git - 在伺服器上佈署 Git](https://git-scm.com/book/zh-tw/v2/%E4%BC%BA%E6%9C%8D%E5%99%A8%E4%B8%8A%E7%9A%84-Git-%E5%9C%A8%E4%BC%BA%E6%9C%8D%E5%99%A8%E4%B8%8A%E4%BD%88%E7%BD%B2-Git)
- [簡易 Git Server 架設](https://ithelp.ithome.com.tw/articles/10250078)
- [[Git] 建立 Git Server 在私人伺服器上](https://myoceane.fr/index.php/git-%E5%BB%BA%E7%AB%8B-git-server-%E5%9C%A8%E7%A7%81%E4%BA%BA%E4%BC%BA%E6%9C%8D%E5%99%A8%E4%B8%8A/)
