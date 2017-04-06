title: DRBD教學文章整理
author: Terry Chen
tags:
  - drdb
  - ha
date: 2016-06-22
---

DRBD (distributed replicated block device) 藉由network的方式將一台的block device (通常指硬碟)，備份到另一台的block device上。這種solution非常依賴網路的速度，如果網卡是10/100Mbps且你要備份的空間太大，第一次同步就得花上不少時間。DRBD常用在HA的架構上，透過Health check (ex: Heartbeat、keepalive等) 來偵測主機狀況，當Primary Server發生異常，自動切換到Secondary Server上，來確保服務不中斷。


## 教學文章

* [MySQL HA 高可用性，DRBD 與Heartbeat - SSORC.tw](http://ssorc.tw/5928)

    對於resource.res有基本的說明，可以清楚地知道設定了什麼；而對於/proc/drbd的各個欄位，也有解釋其代表的意義。

* [Linux HA - DRBD & Heartbeat - James LAB](http://www.james-tw.com/jnote/ha--drbd-heartbeat)

    條列式說明，清楚而方便查閱，並且附上了鏡像寫入的測試方法。


以上教學文章都是使用另一顆硬碟且全新的partition來儲放資料。如果沒有多餘空間或是只想試玩看看的話，則可以利用`dd`和`losetup`指令建出虛擬的硬碟（請參考[使用losetup 建立虛擬磁碟– 煎炸熊の記事本](http://note.artchiu.org/2008/09/29/%E4%BD%BF%E7%94%A8-losetup-%E5%BB%BA%E7%AB%8B%E8%99%9B%E6%93%AC%E7%A3%81%E7%A2%9F/))。如果使用`Vagrant`來玩DRBD的話，則需要額外安裝`linux-headers-$(uname -r)`和`linux-image-extra-$(uname -r)`兩個套件，如果沒有安裝套件，執行`service drbd start`時則會發生`Can not load the drbd module`。


## 常見錯誤

* no resources defined ([linux - no resources defined drbd - Stack Overflow](http://stackoverflow.com/questions/27732757/no-resources-defined-drbd))

    確定`hostname`指令所得到的名字跟resource.res裡頭是一樣的。

* Command 'drbdmeta 1 v08 /dev/sdb1 internal create-md' terminated with exit code 40

    代表此sdb1此partition有資料，DRBD建議了三種方法：

        You need to either
        * use external meta data (recommended)
        * shrink that filesystem first
        * zero out the device (destroy the filesystem)

    如果sdb1的資料你不在乎的話，則可以用 `dd if=/dev/zero of=/dev/sdb1 bs=1M count=128` destroy the filesystem。
