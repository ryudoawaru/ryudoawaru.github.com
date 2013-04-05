---
layout: post
title: "Ruby,Sinatra and Websocket"
date: 2013-04-05 21:48
comments: true
categories: ruby sinatra websocket
---

###近半年來花了很多時間在研究Ruby上實作Websocket的方式, 現介紹簡單心得如下

Base Solution:

*   [EventMachine-Websocket](https://github.com/igrigorik/em-websocket)
*   [Cramp](http://cramp.in/)

Ruby系的幾乎不出這兩種, 其中又以前者為大宗, 後者顯然在Ruby界不流行而且基本上沒在更新了, 然後想當然我也只會用EM

##使用過的EM類Solution：

###[Sinatra-Websocket](https://github.com/simulacre/sinatra-websocket)
最初的嘗試, 好處是整合性極高, 可以直接登記在Sinatra的route上以及共享一切Sinatra的設定, Application Server原則上只能用thin, 不過有致命性的memory leak問題, 簡單來說就是Thin會以為已經斷線的WS連線未斷線因此長久下去會在Thin的process上留著無數的連線進而吃掉大量記憶體, 而且作者沒有配合EM-Websocket改版, 因此被判定為不可用

###直接用EM-Websocket當server
現在的解法, 優點是完全自訂, 同時也是缺點; 等於從0開始設定一個socket server, DB/model…等等的設定都要自己寫; 另外我還比較搞缸的寫了init script變成系統服務, 前端使用haproxy

###負載能力
非常強, 只要處理掉系統的nofile[設定](https://github.com/shokai/sinatra-websocketio/wiki/C10K)的話, 單一行程可以吃到破萬連線不是問題, 不過以現在的EM架構要開多行程處理數萬連線的話, 如果沒有再上一層的pub/sub機制就得要用fork出多行程的方式, 這時就得考慮類似mutex lock的問題了

###IE相容問題
[web-socket-js](https://github.com/gimite/web-socket-js)可以解決你的問題, 不過如果你要用haproxy開前端的話就必需要自己設定flash socket policy file, 使用[flash_policy_server ](http://rubygems.org/gems/flash_policy_server)可手動開這個服務

實際的代碼與架構會在以後說明

