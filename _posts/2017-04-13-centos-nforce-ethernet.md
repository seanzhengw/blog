---
layout: post
title:  "CentOS 7 with nForce Ethernet"
date:   2017-04-13 21:00:00 +0800
categories: centos7
tags: [centos]
---

前幾天幫公司廢物利用，在一台很老舊的電腦上安裝 CentOS 7 打算當作臨時的網頁伺服器使用，安裝時並沒有偵測到可用的網卡，這台電腦原本是安裝 Windows 7，硬體功能都是正常的，應該是沒有安裝驅動程式造成的。

先看了核心版本

    # uname -r

版本

    3.10.0-514.el7.x86_64

一般來說主機板內建的網卡都是PCIE介面，用 lspci 列出包含 net 關鍵字的 PCI 裝置 

    # lspci -nn | grep -i net

然後就得到這個訊息，知道乙太網路設備是 NVIDIA MCP61 Ethernet

    00:07.0 Bridge [0680]: NVIDIA Corporation MCP61 Ethernet [10de:03ef] (rev a2)

接下來到 [ELRepo](http://elrepo.org/tiki/DeviceIDs) 找 `10de:03ef` 對應的驅動名稱，就在 forcedeth.ko 清單中找到

    pci	10DE:03EF	kmod-forcedeth

接著再去 [ELRepo Mirror List](http://elrepo.org/tiki/Download) 隨便找一個 Mirror 下載與核心相對的版本(e17.x86_64)， 在 [http://elrepo.reloumirrors.net/elrepo/el7/x86_64/RPMS/kmod-forcedeth-0.64-2.el7.elrepo.x86_64.rpm]
找到 RPM 包，複製到 CentOS 7 那台電腦上安裝。

**注意**如果只剩下 Windows 的電腦可以上網，那複製檔案要注意檔案系統的問題，將檔案放在一個 **FAT32** 格式的隨身碟 (因為沒有網路，所以不可能裝`exFAT`或`NTFS`，那就只能用核心已經內建的`FAT32`)，接著掛載隨身碟

先查隨身碟的磁碟代號

    # fdisk -l

通常會在最後一個叫做 `/dev/sdg1`

接著掛載

    # sudo mkdir /mnt/usb
    # sudo mount -t vfat /dev/sdg1 /mnt/usb

然後就可以在 `/mnt/usb` 目錄看到隨身碟內的東西了，安裝隨身碟內的驅動

    # sudo rpm -ivh kmod-forcedeth-0.64-2.el7.elrepo.x86_64.rpm

啟動驅動

    # modprobe forcedeth

這樣就驅動就安裝好了

卸載磁碟

    # umount /dev/sdg1

後來這台電腦用不到一個月就淘汰了。