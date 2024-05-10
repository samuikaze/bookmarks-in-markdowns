# 刪除 Linux 終端機中的歷史紀錄

## 說明

有時候可能會在指令中留有敏感資訊，如果要將這些敏感資訊從歷史紀錄中移除，可以執行下列指令清除指定行數

> [!NOTE]
> 請注意，此指令僅會影響目前終端機登入的使用者

```console
$ history -d <ROW_NUM>
```

若是要完全清除，則請執行以下指令

> 請注意，此指令僅會影響目前終端機登入的使用者

```console
$ history -c
```

## 參考資料

- [How to Clear BASH Command Line History in Linux](https://www.tecmint.com/clear-command-line-history-in-linux/)
