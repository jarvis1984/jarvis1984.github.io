---
layout: post
title: "使用gitpage與jekyll建立部落格"
date: 2017-04-16 23:30:00
tags: forntend
description: 使用gitpage搭配jekyll建立部落格之過程
---

最近考慮開個部落格紀錄平常看資料的一些心得
選擇過程首先考量支援markdown
不過光是這一點就少掉很多選擇了
後來大概考慮logdown與gitpage兩個
最後選擇gitpage主要是因為高度支援markdown
以及高度客製化能力
另外整合原本github版本管理也是優點
只是使用過程發現自由真的是有代價的
所以打算把一些過程紀錄下來
<br>

### 相關參考：

**[Jekyll (1) ](http://pigggggggggggggy.logdown.com/posts/2016/02/24/546001)**
- 初始流程及環境建置可參考此篇<br>

**[Creating and Hosting a Personal Site on GitHub](http://jmcglone.com/guides/github-pages/)**
- 詳細流程可參考此篇<br>

**[Themes](https://jekyllrb.com/docs/themes/)**
- theme深入介紹<br>

**[如何把Google Blogger搬到Github pages ](http://blog.kenyang.net/2015/11/26/move-blogger-to-github)**
- 主要可參考其引用現有theme的方法，但須注意第二步有個指令打錯，應改為<br>
git remote set-url origin git@github.com:USERNAME/USERNAME.github.io.git<br>

### 附註：
- github預設以master branch之內容作為網頁來源，需注意專案是否有建立master branch
- 套用現有theme其實可以直接fork過來再改名為github page的名稱即可
