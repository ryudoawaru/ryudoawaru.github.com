---
layout: post
title: "在 Mac / Homebrew 下升級 PostgreSQL 從 9.3 到 9.4"
date: 2015-02-15 22:10
comments: true
categories: postgresql, database, pgsql, homebrew
---

### 前言

PostgreSQL 和 MySQL 不同，資料檔格式會隨著小數點二位數以上的版號（9.3 - 9.4）變動，換句話說每當升級版本時，資料檔就必需要升級才可以繼續使用。

升級的方式有兩種：

-	將原本的資料執行 [pg_dumpall](http://www.postgresql.org/docs/9.4/static/app-pg-dumpall.html) 後再 restore 回去
-	用 [pg_upgrade](http://www.postgresql.org/docs/9.4/static/pgupgrade.html) 命令從舊的資料檔產生出新版本的資料檔。

這邊要講的是第二種情形。

### 環境限定：
-	可執行 [Homebrew](http://brew.sh/) 的 MacOS 環境
-	原 PostgreSQL 為 Homebrew 安裝，9.3.x 版

### 步驟

<ol>
<li>安裝新的 PostgreSQL

```bash
brew update
brew install postgresql
```

</li>
<li>停止現在的 PostgreSQL Daemon
```bash
launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist
```
</li>
<li>建立 9.4 的資料庫檔案
```bash
initdb /usr/local/var/postgres9.4 -E utf8
```
</li>
<li>使用 pg_upgrade 指令升級 舊PG的資料檔案到 9.4，注意這邊的 9.3.5_1 和 9.4.0 是會依照你的舊版本和實際安裝的新板本的版號而有差異的

```bash
pg_upgrade -v \
    -d /usr/local/var/postgres \
    -D /usr/local/var/postgres9.4 \
    -b /usr/local/Cellar/postgresql/9.3.5_1/bin/ \
    -B /usr/local/Cellar/postgresql/9.4.0/bin/
```
</li>
<li>將舊資料檔備份，並且讓新資料檔的目錄更名使之變成預設的資料檔

```bash
cd /usr/local/var
mv postgres postgres9.3
mv postgres9.4 postgres
```
</li>
<li>
重啟 PostgreSQL

```bash
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.postgresql.plist
```
</li>

<li>檢查 /usr/local/var/postgres/server.log 檔是否有正常啟動 server</li>
<li>如果原本有安裝
<a href="https://rubygems.org/gems/pg">pg GEM</a>
的話要重新安裝</li>
</ol>

以上！