---
title: "2019 CCU 程式設計競賽"
date: 2019-06-18T11:06:09+02:00
draft: false
lastmod: 2019-06-18T11:06:09+02:00
tags: ["docker", "domjudge"]
categories: ["CCU Pineapple Contest"]
---

2019 中正大學程式設計競賽總算落幕啦！今年跟 SuperDanby 一起架設了競賽環境，過程中踩過了許多的坑，值得我們一起細數與回味 XD

請先看看 SuperDanby 大神的[競賽環境架設與回憶錄](https://superdanby.github.io/Blog/setup-a-multi-node-local-openshift-cluster.html#using-openshift)

<!--more-->

# Judge 環境

> 作者: 陳鐸元 曾俊宏

感謝 吳邦一 高孟駿老師，還有林俊男先生，以及中正大學電子計算機中心工讀生們的幫忙，我們才得已如期地完成裁判室的系統架設與部屬。

## OS

今年的 judge OS 跟 選手 OS 是不同的。

Domjudge 只有對 linux 有原生的支援，所以 judgehost 是跑在 linux 上面。

由於考慮到比賽屬於推廣性質，所以選手機台仍是使用 Windows 10。

# Domjudge 

今年是第一次使用 DomJudge 當作校內賽的比賽平台，運行的平台則是採用 Fedora 30 + official docker image。比起原生的機器，使用 docker 在效能沒有顯著差距，但是部署上，讓我們得以簡單的 scale 整個系統到多台電腦上。不然安裝 domjudge 很曠日廢時啊！

但是 Docker 版的 domjudge 有幾個坑... 我們來一一細數...

## Docker 坑坑

### Docker for windows

畢竟 docker for windows 還是跑在 hyperV 上，經過觀察，綁 core 在虛擬機內部可行，但是在 出到 windows 後就沒效了。而且只跑一個 judgehost 大概就會吃掉 windows 40% 左右的 CPU resource。

所以最後只好裝起了 Fedora 30 來當 judge 的 host OS，也就如預期了。

### Multicore issue

multiple judgehost per machine 其實也不太可行，原因不明。

在指定一個 judgehost 綁定一個 core 的情況下 （利用daemonID指定），越多 judgehost 一起同時進行 judging 的話，cputime 是保持幾乎一樣，但是 wall time 是爆炸性成長（3 個 judgehost 一起 judge 的話有機會慢到一倍之多！）

無解，最後只能一機器一個judge。

## Fork bomb

預設的 judgehost image 是會被 fork bomb 打死的，因為 container 裡面並沒有 init process。

解決方法很簡單，docker run 時指定一下 pid 1 要是 init process 即可。

### 安裝 python3 + Java

安裝新的程式語言，在 chroot 外安裝完後，要使用 `dj_run_chroot` 再裝一次，這樣這個新增的語言才能被拿來 judge。

以下是測試賽前一天火燒屁股的一段故事。

因為 `dj_run_chroot` 要 mount 東西，但是 docker build 的過程是不允許 mount 的，所以只好多階段的進行。先是把要安裝的東西寫在 shell script 中，之後再放入 dockerfile 的 CMD 裡

原本想說，在 judge 起來後，`sudo docker exec -it docker_judgehost-0_1 /bin/bash`進去 judgehost container，跑了`./opt/domjudge/judgehost/bin/dj_run_chroot "apt update && apt -y install python3 default-jre default-jdk"` 和 `apt update && apt -y install python3  default-jre default-jdk`，commit 一下目前的 judgehost 狀態，之後推上dockerhub，就可以收工了。

結果，judgehost 一換電腦，就...
![cgroup error](https://i.imgur.com/XlW4HD2.png)

慘了，奮戰一番後發現，因為 judgehost container 啟動時，`start.sh` 有 call `create_cgroup`，所以 commit 的 image 已經有著不該帶著的 cgroup 資訊！也因此換電腦執行 judging 任務時，一動到 cgroup 就死了...

所以只好分階段，build image啦QQ 

### MLE 

無解。

會直接被判成 RE。
