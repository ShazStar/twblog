---
layout: post
title:  架設UPS NUT伺服器連接3台電腦
date:   2025-09-10 18:45:00 +0800
category: Tutorial
tags: [tutorial]
---
暑假期間，我很幸運的從老家一小段距離的回收場搞了台UPS, 要處理的東西就只有電池跟一顆電容，在我換完電池並憑藉朋友之力幫我處理完電容後，機子就完全正常了，但後續接上電腦才是真正要處理的活，只接一台其實很簡單好搞，但多台電腦連接就有點頭痛了，所以在花了半個下午找資料並把所有東西都設定起來後，我決定寫這篇Blog給我自己還有需要的人。

UPS的型號是APC Back-UPS Pro 700. 在線互動式，700VA/420W, 6個插座（3個電池保護，3個突波保護），接上Mac Pro, TrueNAS跟Proxmox主機後在正常使用下通常座落在230~300W之間，Mac Pro睡眠狀態則座落在80W之間，目前使用到現在還沒讓電腦跑重度的東西去看UPS吃不吃得消，但我覺得這台UPS目前應該是足以應付這三台的使用量了。

UPS 擺設 + 近距離:

<img src="{{ site.baseurl }}/images/20250910_UPS/UPS_Placement.jpeg" width="300"/>
<img src="{{ site.baseurl }}/images/20250910_UPS/Close_Up.jpeg" width="300"/>

# 硬體配置

這台UPS有3個電池保護插孔，我拿來接了Mac Pro, Proxmox跟TrueNAS主機，UPS USB則與Proxmox主機連接主從，其餘兩台則是隨從。

我原本其實有考慮過用我最近拿到的Raspberry Pi 3B來當主從系統但最後沒有實作，~~因為我還沒玩過Raspberry Pi~~ 因為我沒有把Pi接上UPS所以沒什麼用處...除非我Mac Pro不接UPS不然就是把Pi接上去其中一台電腦吃他們的電。

# 軟體配置

基本上跟上面說的一樣，我Proxmox主機會拿來當主從機器，所以需要在這台電腦上架設NUT伺服器，開搞！

## NUT主從電腦設定

首先，我們得先確認UPS有接上電腦並有辨識出，在終端機打上 `lsusb` 列出所有連接的USB設備。
<img src="{{ site.baseurl }}/images/20250910_UPS/lsusb.png" width="800"/>
可以看到在Bus 003 Device 002是UPS連接的地方，記下來。

重要筆記: 因為Proxmox可以使用root權限，所以我不需要使用sudo, 請依照你配置電腦來判斷是否需要使用sudo

用`apt update`更新軟體源後用這個指令安裝nut, nut-client 跟 nut-server: `apt install nut nut-client nut-server`

使用 `nut-scanner -U` 去搜尋UPS裝置並取得其資訊，筆記下來。

我們需要編輯以下檔案: ups.conf, upsmon.conf, upsd.conf, nut.conf and upsd.users.

因範例檔案有許多資料所以我最後把這些原本的檔案備份起來並重現創建新的使用。
可以用 `cp /etc/nut/ups.conf /etc/nut/ups.example.conf` 來複製原檔案後再把原檔案的內容做清除，或用 `mv /etc/nut/ups.conf /etc/nut/ups.example.conf` 來重新命名並重新創建新檔案， 更改ups.conf為你要編輯的檔案， 現在我們用nano來編輯這些檔案: `nano /etc/nut/ups.conf`. 以下是我的配置。

ups.conf:
```
pollinterval = 1
maxretry = 3

[ups]
driver = usbhid-ups
port = auto
desc = "APC Back-UPS Pro 700"
vendorid = 051D
productid = 0002
serial = 3xxxxxxxxxxx
```
upsmon.conf:
```
MONITOR ups@localhost 1 upsmon secret master
```
upsd.conf:
```
LISTEN 0.0.0.0 3493
```
nut.conf:
```
MODE=netserver
```
upsd.users:
```
[monuser]
password = secret
upsmon master
```

編輯完這些檔案後，重新開機。

重新開機後，用這個指令確認UPS連接狀態: `upsc ups@localhost`

你會看到這個: <img src="{{ site.baseurl }}/images/20250910_UPS/upsc_master.png" width="800"/>

這就代表設定是成功的，NUT可以抓到UPS的資料。

## NUT CGI 設置
CLI資料很詳細，但如果能靠網頁顯示圖表資訊的話會更好對吧？nut-cgi就是用來提供這功能的。

需要先安裝apache2跟nut-cgi: `apt install apache2 nut-cgi`

安裝後使用nano編輯host.conf檔案： `nano /etc/nut/hosts.conf`. 一樣備份原本檔案後再來做編輯動作。

hosts.conf:
`MONITOR ups@localhost "APC Back-UPS Pro 700"`

編輯完後打上 `a2enmod cgi` 啟動模組後用 `systemctl restart apache2`來重啟apache2.

你還需要編輯 upsset.conf檔案 -> `nano /etc/nut/upsset.conf`

upsset.conf:
`I_HAVE_SECURED_MY_CGI_DIRECTORY`

全部完成後就可以使用這個網頁去看UPS資訊了: `http://192.168.x.x/cgi-bin/nut/upsstats.cgi`, 把x.x改成你主從電腦的ip位置。

<img src="{{ site.baseurl }}/images/20250910_UPS/cgi.png" width="800"/>

## 隨從電腦設定

上面都處理完後就可以開始搞隨從電腦了, TrueNAS Scale, macOS, Linux跟Windows都會列出。

### TrueNAS Scale
設置其實非常簡單明瞭，你只需要去系統設定-> 服務並在那邊設定UPS服務就行了，直接看圖片設置就懂了： <img src="{{ site.baseurl }}/images/20250910_UPS/TrueNAS_Setup.png" width="800"/>

設定完後，記得啟用服務並勾選自動啟動。

驗證連線狀態，去系統設定-> Shell打上這串指令，但這次把localhost改成主從PC的ip位置: `upsc ups@192.168.x.x`

如果設定成功的話你就能看到UPS資訊回報了: <img src="{{ site.baseurl }}/images/20250910_UPS/upsc_slave_truenas.png" width="800"/>

### Linux
其實我並沒有Linux裝置接上UPS (TrueNAS跟Proxmox不算), 但我還是決定把Linux的部分寫出來供有需要的人使用，但可能會因你使用的Distro有所差異，我會寫Arch Linux的因為我個人是用Arch Linux.

pacman安裝nut -> `sudo pacman -Syu nut`

安裝後你需要編輯3個檔案: nut.conf, upsmon.conf and upssched.conf.

nut.conf:
`MODE=netclient`

upsmon.conf:
```
RUN_AS_USER root

MONITOR ups@192.168.22.117 1 upsmon secret slave

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h"
NOTIFYCMD /usr/sbin/upssched
POLLFREQ 2
POLLFREQALERT 1
HOSTSYNC 15
DEADTIME 15
POWERDOWNFLAG /etc/killpower

NOTIFYMSG ONLINE    "UPS %s on line power"
NOTIFYMSG ONBATT    "UPS %s on battery"
NOTIFYMSG LOWBATT   "UPS %s battery is low"
NOTIFYMSG FSD       "UPS %s: forced shutdown in progress"
NOTIFYMSG COMMOK    "Communications with UPS %s established"
NOTIFYMSG COMMBAD   "Communications with UPS %s lost"
NOTIFYMSG SHUTDOWN  "Auto logout and shutdown proceeding"
NOTIFYMSG REPLBATT  "UPS %s battery needs to be replaced"
NOTIFYMSG NOCOMM    "UPS %s is unavailable"
NOTIFYMSG NOPARENT  "upsmon parent process died - shutdown impossible"

NOTIFYFLAG ONLINE   SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT   SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT  SYSLOG+WALL
NOTIFYFLAG FSD      SYSLOG+WALL+EXEC
NOTIFYFLAG COMMOK   SYSLOG+WALL+EXEC
NOTIFYFLAG COMMBAD  SYSLOG+WALL+EXEC
NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC
NOTIFYFLAG REPLBATT SYSLOG+WALL
NOTIFYFLAG NOCOMM   SYSLOG+WALL+EXEC
NOTIFYFLAG NOPARENT SYSLOG+WALL

RBWARNTIME 43200

NOCOMMWARNTIME 600

FINALDELAY 5
```
upssched.conf:
```
CMDSCRIPT /etc/nut/upssched-cmd
PIPEFN /etc/nut/upssched/upssched.pipe
LOCKFN /etc/nut/upssched/upssched.lock

AT ONBATT * START-TIMER onbatt 30
AT ONLINE * CANCEL-TIMER onbatt online
AT ONBATT * START-TIMER earlyshutdown 30
AT LOWBATT * EXECUTE onbatt
AT COMMBAD * START-TIMER commbad 30
AT COMMOK * CANCEL-TIMER commbad commok
AT NOCOMM * EXECUTE commbad
AT SHUTDOWN * EXECUTE powerdown
AT SHUTDOWN * EXECUTE powerdown
```
設定完成後用 `sudo systemctl restart nut`來重啟nut服務.

驗證: `upsc ups@192.168.x.x`


### macOS
這其實是我搞最久的，有點頭痛但其實不會到很麻煩，要記住的點是因為NUT是靠homebrew安裝的，所以路徑會與Linux稍有不同，通常NUT的路徑會在`/etc/nut/`, 但macOS的在`/usr/local/Cellar`, 他的設定檔存在`/usr/local/etc/nut`.

用homebrew安裝nut -> `brew install nut`

安裝後你需要編輯3個檔案: nut.conf, upsmon.conf and upssched.conf.

nut.conf:
`MODE=netclient`

upsmon.conf:
```
RUN_AS_USER root

MONITOR ups@192.168.22.117 1 upsmon secret slave

MINSUPPLIES 1
SHUTDOWNCMD "/sbin/shutdown -h"
NOTIFYCMD /usr/sbin/upssched
POLLFREQ 2
POLLFREQALERT 1
HOSTSYNC 15
DEADTIME 15
POWERDOWNFLAG /etc/killpower

NOTIFYMSG ONLINE    "UPS %s on line power"
NOTIFYMSG ONBATT    "UPS %s on battery"
NOTIFYMSG LOWBATT   "UPS %s battery is low"
NOTIFYMSG FSD       "UPS %s: forced shutdown in progress"
NOTIFYMSG COMMOK    "Communications with UPS %s established"
NOTIFYMSG COMMBAD   "Communications with UPS %s lost"
NOTIFYMSG SHUTDOWN  "Auto logout and shutdown proceeding"
NOTIFYMSG REPLBATT  "UPS %s battery needs to be replaced"
NOTIFYMSG NOCOMM    "UPS %s is unavailable"
NOTIFYMSG NOPARENT  "upsmon parent process died - shutdown impossible"

NOTIFYFLAG ONLINE   SYSLOG+WALL+EXEC
NOTIFYFLAG ONBATT   SYSLOG+WALL+EXEC
NOTIFYFLAG LOWBATT  SYSLOG+WALL
NOTIFYFLAG FSD      SYSLOG+WALL+EXEC
NOTIFYFLAG COMMOK   SYSLOG+WALL+EXEC
NOTIFYFLAG COMMBAD  SYSLOG+WALL+EXEC
NOTIFYFLAG SHUTDOWN SYSLOG+WALL+EXEC
NOTIFYFLAG REPLBATT SYSLOG+WALL
NOTIFYFLAG NOCOMM   SYSLOG+WALL+EXEC
NOTIFYFLAG NOPARENT SYSLOG+WALL

RBWARNTIME 43200

NOCOMMWARNTIME 600

FINALDELAY 5
```
upssched.conf:
```
CMDSCRIPT /usr/local/Cellar/nut/2.8.4/bin/upssched-cmd
PIPEFN usr/local/etc/nut/upssched/upssched.pipe
LOCKFN usr/local/etc/nut/upssched/upssched.lock

AT ONBATT * START-TIMER onbatt 30
AT ONLINE * CANCEL-TIMER onbatt online
AT ONBATT * START-TIMER earlyshutdown 30
AT LOWBATT * EXECUTE onbatt
AT COMMBAD * START-TIMER commbad 30
AT COMMOK * CANCEL-TIMER commbad commok
AT NOCOMM * EXECUTE commbad
AT SHUTDOWN * EXECUTE powerdown
AT SHUTDOWN * EXECUTE powerdown
```

設定完成後用 `brew services start nut`來啟動nut服務.

可以用同個指令驗證: `upsc ups@192.168.x.x`
<img src="{{ site.baseurl }}/images/20250910_UPS/upsc_slave_mac.png" width="800"/>


### Windows

Windows的設定也非常簡單，你只需要安裝 [WinNUT Client](https://github.com/nutdotnet/WinNUT-Client) 後設定就好了，這邊不需要詳細解說因為他有GUI介面。

WinNUT 設定:
<img src="{{ site.baseurl }}/images/20250910_UPS/WinNUT_Setup.png" width="800"/>

WinNUT Client:
<img src="{{ site.baseurl }}/images/20250910_UPS/WinNUT.png" width="800"/>

**有用連接**

我所使用的教學:

[Techno Tim's Ultimate NUT Server Guide Video](https://www.youtube.com/watch?v=vyBP7wpN72c) 跟他的 [blog post](https://technotim.live/posts/NUT-server-guide/)
 
[Kreaweb's guide](https://www.kreaweb.be/diy-home-server-2021-software-proxmox-ups/)

[Hardware Haven's video](https://www.youtube.com/watch?v=dXSbURqdPfI)