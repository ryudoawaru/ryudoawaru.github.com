---
layout: post
title: "解決用 Mac 登入 Linux 主機時出現的 locale 警告訊息"
date: 2015-02-20 16:07
comments: true
categories: [bash, locale, ssh, mac, iterm]
---

前陣子用 Mac 登入遠端的 Linux 主機時常常遇到一個有點討厭的訊息：

```bash
-bash: warning: setlocale: LC_CTYPE: cannot change locale (UTF-8)
```

這個狀況是從幾個月前突然發生的，Google 之後發現兩種解法：

1.	修改主機 locale 設定
	
	編輯：/etc/environment 並輸入：
	```
	LANG=en_US.utf-8
	LC_ALL=en_US.utf-8
	```

2.	修改 iterm 設定

![](images/iterm-local-setting.jpg)

將裡面的 "Set locale variables automatically" 取消勾選即可。

