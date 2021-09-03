---
title: "編譯 Raspberry Pi 4 的 Kernel Image"
date: 2021-09-03T09:38:04+02:00
draft: false
lastmod: 2021-09-03T09:38:04+02:00
tags: ["Raspberry Pi", "Kernel", "Cross compile"]
categories: ["Tech"]
---

比較常見可以跑在 Raspberry Pi 4 的 Linux kernel 大致上分成 [Mainline Kernel](https://www.kernel.org/)（本文撰寫時 [5.14](https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.14.tar.xz) 剛剛推出）, [Raspbian](https://github.com/raspberrypi/linux) (5.10, 32-bit), 跟 [Ubuntu](https://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux-raspi/+git/focal) (5.4, 32/64-bit) 這幾種。其中，Raspbian 目前還是以 32-bit 的 kernel 為主（所有 user space 的應用程式都是 32-bit，雖然你可以把 kernel 換成 64-bit 的）。

由於我自己的實驗所需(對 kernel 進行 patching），下面在 Raspberry Pi 上面進行編譯的段落，是基於 [Ubuntu Server for Raspberry Pi 64-bit 版本的 kernel](https://ubuntu.com/download/raspberry-pi) 所撰寫，畢竟我需要的是 64-bit 的環境。

Raspbian 的編譯方式[官方文件](https://www.raspberrypi.org/documentation/computers/linux_kernel.html)就已經寫得很清楚了，所以本文著墨的主要是在 Ubuntu Server for Raspberry Pi (64-bit) 版本的 kernel 跟 Mainline Kernel 的部分。

> 這篇文是從我研究過程中[雜亂的筆記](https://hackmd.io/ryWMkvbhRDOoqYEooWZ6BQ)所整理出來，過程中需要安裝的套件或是步驟可能會有錯誤或是缺漏（畢竟前前後後找問題裝了不少東西）。
> 
> 如有疏漏，請大家不吝指教，謝謝大家！

<!--more-->

# 編譯前準備

Kernel 可以直接在 host 環境上編譯，也可以使用 cross compile 的技巧來進行。通常嵌入式系統的 CPU 運算能力都比較差，所以使用 cross compile 的技巧可以省很多時間。

## 在 Raspberry Pi 4 (host) 上面編譯 Kernel

這一段都是基於假設你使用的是 Ubuntu for Raspberry Pi 64-bit 版本的 kernel 所撰寫。

### 準備 native compiler 跟工具

```bash
sudo apt update
sudo apt build-dep linux linux-image-$(uname -r)   
sudo apt install build-essential libncurses-dev linux-tools-common fakeroot
```

### 耗時

經驗上來說，完整的編譯一次大概要 2~3 小時（如果有裝風扇且開 performance mode 的情況下）。

## 使用 x86 的機器 (guest) 進行 Cross Compilation

### 準備環境

```bash
sudo apt update
sudo apt build-dep linux 
sudo apt install build-essential libncurses-dev linux-tools-common fakeroot
```

### 準備 cross compiler

兩種方法都可以，就只是要注意一下 compiler 的名稱有一點點差異。 

#### 使用 `apt` 安裝

* `sudo apt install crossbuild-essential-arm64`
    * 如果你只想要 compiler 而不是整套工具，可以改成執行 `sudo apt install gcc-aarch64-linux-gnu`

#### 使用預先編譯好的 compiler toolchain

* 因為 toolchain 的檔案 $\geq$ 100Mb, 所以必須許用 [git lfs](https://www.atlassian.com/git/tutorials/git-lfs) 完整下載整個 repo 裡面的 toolchain
    * 執行 `sudo apt install git-lfs` 安裝 `git lfs`
* `git clone https://github.com/DLTcollab/toolchain-arm`
* 挑選所要的 toolchain 進行解壓縮 `tar -xvJf [filename]`
    * 64-bit 的 kernel 請使用 `aarch64-none-linux-gnu` 系列的 toolchain
* 在你的 `~/.bashrc` 裡（或是你對應的 shell config file 中），加入剛剛解壓縮的 compiler 的 `bin` 路徑
    * `export PATH=$PATH:/home/ubuntu/toolchain-arm/gcc-arm-10.2-2020.11-x86_64-aarch64-none-linux-gnu/bin` 

### 耗時

我自己是使用 `i5-1135g7` (4C8T) ，跑 virtual box 配給 4 個 virtual CPU，半小時內可以完成編譯。

### 筆記

網路上有好心人把 cross compile 的環境[打包成了 docker image](https://github.com/carlonluca/docker-rpi-ubuntu-kernel)。我自己沒實際使用過，但是看 dockerfile 跟 run script 都滿清晰，也跟我自己的編譯步驟是相同的。

# 編譯

分成 Ubuntu 版本 跟 mainline kernel 兩個進行討論。

## 編譯 [Ubuntu Server for Raspberry Pi 64-bit](https://ubuntu.com/download/raspberry-pi) 的 kernel (5.4) - 使用 `debian/rules`

[官方文件](https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel) 雖然有點舊，但是仍然是正確的（也可以參考[這篇文](https://askubuntu.com/questions/1238261/customizing-the-kernel-arm64-using-ubuntu-20-04-lts-on-a-raspberry-pi-4)的 all-in-one 整理）。

Ubuntu 是使用 Debian 的 `fakeroot debian/rules` toolchain 系列，滿好使用的。

* 下載原始碼 （建議原始碼外面有多一層目錄，因為編譯出來的 `.deb` 會被放在執行編譯指令的上一層目錄中）
    ```bash
    mkdir ~/kbuild
    cd ~/kbuild
    git clone --depth=1 git://git.launchpad.net/~ubuntu-kernel/ubuntu/+source/linux-raspi/+git/focal
    ```
* 如果打算進行 cross compile
    ```bash
    export CC=aarch64-linux-gnu-gcc
    export $(dpkg-architecture -aarm64)
    export CROSS_COMPILE=aarch64-linux-gnu-
    ```
* 修改 `uname` (可以略過這一步驟)
    * 使用 `fakeroot debian/rules editconfigs` 來進行修改是無效的（因為 append LOCALVERSION 的選項已經被強制關閉 - [module 編譯的問題](https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel#Modifying_the_configuration)）
    * 要去 `debian.raspi/changelog` 在第一行的版本號那裡加字串
        * 請使用加號開頭，大小寫無所謂 e.g. 新增字串`+CFS`
* `fakeroot debian/rules clean`
* `fakeroot debian/rules binary-headers binary binary-perarch` 
    
## 編譯 Mainline Kernel (5.14)

* 下載原始碼（建議原始碼外面有多一層目錄，因為編譯出來的 `.deb` 會被放在執行編譯指令的上一層目錄中）
    ```bash
    mkdir ~/kbuild
    cd ~/kbuild
    wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.14.tar.xz
    tar -xvJf linux-5.14.tar.xz
    ```
* 編譯方式
    ```
    make distclean
    make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- defconfig
    sed -i 's/CONFIG_LOCALVERSION_AUTO=y/CONFIG_LOCALVERSION_AUTO=n/' .config
    make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- LOCALVERSION="-cfs-raspi" -j$(nproc) deb-pkg
    ```
    * 一些筆記
        * `make distclean` 
            * 注意，這命令並不會清理 patch 過的程式碼（可以使用 `git reset` 或是 `git checkout` 等命令）
        * `make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- defconfig`
            * 請注意 `ARCH` 跟 `CROSS_COMPILE` 的設定是不是是不是正確的
                * 如果沒有要 cross compile 可以省略這個參數
            * *`defconfig` 會造成編譯出來的 `deb` 裡頭的 `dts` 包含很多用不到的 driver -> 也許可以從 raspbian 的 `arch/arm64/config/bcm2711_defconfig` 進行調整（這只是個發想）*
        * `sed -i 's/CONFIG_LOCALVERSION_AUTO=y/CONFIG_LOCALVERSION_AUTO=n/' .config`
            * *花了我最多時間所得到的一行指令* -> 如果想要讓 mainline kernel 得到跟使用 Ubuntu toolchain 編譯 kernel 一樣效果（可以使用 `dpkg -i` 來自動進行 kernel 置換），這個就很重要了。
                * 這一行的功能是讓 `deb` 的 post installation script (在 `/etc/kernel`) 的 `flash-kernel` 可以正常的被執行。
                    *  `flash-kernel` 要可以正常地執行的[先決條件](https://askubuntu.com/questions/1207467/raspberry-pi-4-custom-kernel-will-not-install-in-ubuntu-19-10)就是 `uname -r` 要是 `-raspi` 結尾！由於 `CONFIG_LOCALVERSION_AUTO` 會自動在 `uname -r` 結尾加上 `-gxxxxxxx` 的字串，所以我們要[抑制掉這個效果](https://unix.stackexchange.com/questions/194129/linux-kernel-version-suffix-config-localversion)。
        * `make ARCH=arm64 CROSS_COMPILE=aarch64-none-linux-gnu- LOCALVERSION="-cfs-raspi" -j$(nproc) deb-pkg`
            * 請注意 `ARCH` 跟 `CROSS_COMPILE` 的設定
                * 如果沒有要 cross compile 可以省略這兩個參數
            * `LOCALVERSION` 
                * 可以略過這個修改 `uname` 的步驟
                * 建議使用 `-` 開頭，才不會把版本號直接跟你的字串串接了
                * 一定要用 `-raspi` 當結尾，理由請看上一段
                * 一定要小寫
            * 如果只要編譯 binary 的話，請使用 [`bindeb-pkg`](https://patchwork.kernel.org/project/linux-kbuild/patch/1432804275-13187-3-git-send-email-riku.voipio@linaro.org/)

# 安裝 kernel

* `sudo dpkg -i *.deb`
    * 如果 `uname` 正確的話，u-boot 會自動地被置換。
* `sync; sudo reboot` 重新開機

# 移除 kernel

* 使用 `apt list --installed | grep "linux-image*raspi"` 看一下目前有安裝的 kernel 
* 使用 `sudo apt autoremove --purge [kernel image name]` 來進行移除
    * 如果 `uname` 正確的話，u-boot 會自動地被置換。