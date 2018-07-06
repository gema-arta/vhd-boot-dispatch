# msys2-windows-utils
Windows utilities under MSYS2

## 說明
使用 GNU/Linux 環境同時派送 Windows VHD 至多台個人電腦，以 Native VHD Boot 方式開機。
- Server 端以 WOL (Wake-on-LAN) 方式啟動多台個人電腦
- dhcpd 派送網路設定與 next-server 資訊
- in.tfptd 派送 pxelinux.0、pxelinux.cfg/default 與 Tiny Core Linux 作業系統
- 修改 Tiny Core Linux 套件，增加 ntfs-3g、ntfsprogs、openssh、rsync、udpcast 與 uftp 等程式
- 修改 Tiny Core Linux 帳號密碼，如 /etc/passwd、/etc/shadow，並啟動 sshd server
- 於 server 端使用 PSSH 控制遠端的個人電腦
  * 掛載 NTFS 分割區
  * 以 UDP Multicast 或 BitTorrent 方式接收 VHD 檔案、grub4dos、BOOTMGR
  * 安裝 grub4dos 至 MBR 或使用既有的開機程式
  * 重新開機
- 於 Windows 中安裝 MSYS2，以 cygrunsrv 方式啟動 sshd server
- Windows 開機後自動依 IP 設定電腦名稱

## Disk layouts
|檔案名稱|用途|類型|狀態|大小|
|---|---|---|---|---|
|disk_p.vhd|母碟|基礎磁碟|靜態|約 30 GB|
|disk.vhd|子碟|差異化磁碟|動態|隨差異增加|
|disk_chd.vhd|還原檔|差異化磁碟|靜態|約 173 KB|


## 如何準備 VHD 檔案
- 透過 VirtualBox 安裝新系統
- [取消 `VirtualDiskExpandOnMount`](https://superuser.com/questions/1149941/how-to-by-pass-vhd-boot-host-volume-not-enough-space-when-native-booting)
  ```
  HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\services\FsDepends\Parameters
  VirtualDiskExpandOnMount = 4
  ```
- [SDelete 清理磁碟](https://docs.microsoft.com/en-us/sysinternals/downloads/sdelete)
```
> sdelete64 -c -z C:
```
- 緊湊 VHD 磁碟空間
  * [Windows](https://social.technet.microsoft.com/wiki/contents/articles/8043.how-to-compact-a-dynamic-vhd-with-diskpart.aspx)
  ```
  diskpart
  > select vdisk file="f:\disk_p.vhd"
  > compact vdisk
  ```
  * [Linux](https://serverfault.com/questions/888986/compact-a-vhd-on-a-linux-host)
  ```
  $ vboxmanage clonemedium INPUT.VHD OUTPUT.VHD --format VHD --variant Standard
  ```
- [建立差異化磁碟 (Differencing Disks)](https://blogs.msdn.microsoft.com/7/2009/10/07/diskpart-exe-and-managing-virtual-hard-disks-vhds-in-windows-7/)
```
diskpart
> create vdisk file="f:\disk.vhd" parent="f:\disk_p.vhd"
```
- [合併差異化磁碟](https://blogs.msdn.microsoft.com/7/2009/10/07/diskpart-exe-and-managing-virtual-hard-disks-vhds-in-windows-7/)
```
diskpart
> select vdisk file="f:\disk.vhd"
> merge depth=1
```

## 開機流程
- grub2 (無 grub2 可略過)
```
menuentry 'GRUB4DOS (NTFS)' --class windows {
    savedefault
    insmod part_msdos
    insmod ntfs
    insmod ntldr
    search --set=root --no-floppy --file /sig_ntfs    # 特徵檔案，識別磁碟區位置
    ntldr /grldr                                      # 控制權交給 grub4dos
}
```
- grub4dos
```
timeout 5
default 0

title Windows 10 (Native VHD Boot)

find --set-root --ignore-floppies --ignore-cd /sig_ntfs
dd if=()/Boot/BCD.disk.vhd of=()/Boot/BCD                  # 設定 disk.vhd 為開機裝置 (可切換不同用途的 VHD 檔案)

find --set-root --ignore-floppies --ignore-cd /sig_ntfs    # 還原 disk_chd.vhd 至 disk.vhd
dd if=()/disk_chd.vhd of=()/disk.vhd

find --set-root --ignore-floppies --ignore-cd /sig_ntfs
chainloader /bootmgr                                       # 控制權交給 bootmgr (Boot\BCD)
```

## Boot
- bootmgr
- BOOTNXT
- Boot
- Boot\BCD (紀錄作業系統開機的裝置)
- grub4dos\
- sig_* (特徵檔案，識別磁碟區位置)

## VHD
```
$ modprobe loop
$ losetup /dev/loop0 disk.vhd
$ partprobe /dev/loop0
$ mount /dev/loop0p1 /mnt/images

$ losetup -d /dev/loop0
```

```
$ VBoxManage clonehd dynamicd_disk.vhd fixed_disk.vhd --format vhd --variant dynamic
```

## PSSH
```
$ rm /root/.ssh/known_hosts; sshpass -p [密碼] pssh -i -A -h [主機清單] -O "StrictHostKeyChecking no" -- "[指令]"
$ rm /root/.ssh/known_hosts; sshpass -p [密碼] pssh -i -A -H user@host_or_ip -O "StrictHostKeyChecking no" -- "[指令]"
```

```
$ rm /root/.ssh/known_hosts; sshpass -p [密碼] pssh -i -A -h [主機清單] -O "StrictHostKeyChecking no" -- "sudo ntfs-3g /dev/sda2 /mnt/sda2"
$ rm /root/.ssh/known_hosts; sshpass -p [密碼] pssh -i -A -h [主機清單] -O "StrictHostKeyChecking no" -- "tar xvfz /mnt/sda2/vhd.tgz -C /mnt/sda2"
$ rm /root/.ssh/known_hosts; sshpass -p [密碼] pssh -i -A -h [主機清單] -O "StrictHostKeyChecking no" -- "sudo reboot"
```

```
$ rm /root/.ssh/known_hosts; sshpass -p [密碼] pssh -i -A -h [主機清單] -O "StrictHostKeyChecking no" -- "shutdown -s -t 30"
$ rm /root/.ssh/known_hosts; sshpass -p [密碼] pssh -i -A -h [主機清單] -O "StrictHostKeyChecking no" -- "shutdown -r -t 30"
```

## grub4dos
```
$ /usr/local/share/grub4dos/bootlace.com /dev/sda
$ cp /usr/local/share/grub4dos/{chinese/}grldr /mnt/sda2
```

## External Reference
- [Grub4dos Guide](http://diddy.boot-land.net/grub4dos/Grub4dos.htm)
- [Tiny Core Linux](http://distro.ibiblio.org/tinycorelinux/)
  * ntfs-3g
  * ntfsprogs
  * rsync
  * grub4dos
  
  ```
  $ tce-load -wi ntfs-3g
  ```
  
- [Tiny Core Linux Customizations for Remastering](http://www.canbike.org/off-topic/aggregate/tiny-core-linux-customizations-for-remastering.html)

  ```
  $ zcat core.gz | sudo cpio -i -H newc -d
  $ sudo find | sudo cpio -o -H newc | gzip -2 > core.gz
  ```
- [Transmission](https://transmissionbt.com/)

  ```
  $ vi settings.json
  "dht-enabled": true
  "lpd-enabled": true
  ```

  ```
  $ transmission-create -o disk.torrent -c "Win7 VHD Boot" disk.vhd
  $ tftp -r pcroom/disk.torrent -l /mnt/sda2/p2p/disk.torrent -g [伺服器位址]
  $ transmission-cli -v -w /mnt/sda2/p2p/ /mnt/sda2/p2p/disk.torrent
  ```

- [Udpcast](https://www.udpcast.linux.lu/) / [uftp](http://uftp-multicast.sourceforge.net/) / [mrsync](https://sourceforge.net/projects/mrsync/)

  ```
  $ udp-sender --full-duplex -f source.vhd
  $ udp-receiver -f saved.vhd
  ```
  
  ```
  $ mkdir /mnt/sda2/t                       # Temp
  $ uftpd -D /mnt/sda2/ -T /mnt/sda2/t/     # Dest, Temp (Client)
  $ uftp -R -1 /srv/pcroom/pieces/          # Source directory (Server)
  
  $ split -b 1G /src/pcroom/disk.vhd /src/pcroom/pieces  # Split, unknown file size limit (2147483647)
  $ cat * > disk.vhd                                     # Join
  ```
  * [How can I get gcc to write a file larger than 2.0 GB?](https://askubuntu.com/questions/21474/how-can-i-get-gcc-to-write-a-file-larger-than-2-0-gb)
  
  ```
  $ tce-load -wi compiletc
  $ cd uftp-4.9.3/
  $ vim makefile                            # Add -D_GNU_SOURCE (-D_FILE_OFFSET_BITS=64)
  ifeq ("Linux", "$(UNAME_S)")
  ...
  ...
  bad-function-cast -DHAS_GETIFADDRS -D_GNU_SOURCE $(ENC_OPTS) 
  
  $ make NO_ENCRYPTION=1
  ```

- [Parallel SSH](https://pypi.python.org/pypi/pssh)
- [MSYS2](http://www.msys2.org/)
- [ConEmu](https://conemu.github.io/)
- [Booting Windows to a Differencing Virtual Hard Disk](https://blogs.msdn.microsoft.com/heaths/2009/10/13/booting-windows-to-a-differencing-virtual-hard-disk/)
- [Native VHD Boot on unsupported versions of Windows 7](http://agnipulse.com/2016/12/native-vhd-boot-unsupported-versions-windows-7/)
- [WakeMeOnLan](http://www.nirsoft.net/utils/wake_on_lan.html)
  * [HowTo: Wake Up Computers Using Linux Command [ Wake-on-LAN ( WOL ) ]](https://www.cyberciti.biz/tips/linux-send-wake-on-lan-wol-magic-packets.html)
  
  ```
  $ ethtool -s net0 wol g
  $ etherwake -i [介面] -b [MAC Address / 11:22:33:44:55:66]
  ```
  
- [UEFI support for PXE booting](http://projects.theforeman.org/projects/foreman/wiki/PXE_Booting_UEFI/8)
- [PXELINUX](http://www.syslinux.org/wiki/index.php?title=PXELINUX)

  ```
  LABEL tiny-core-linux
  TITLE Tiny Core Linux
  LINUX /corelinux/vmlinuz
  INITRD /corelinux/core.gz,/corelinux/tce.gz,/corelinux/cde.gz
  APPEND loglevel=3,next-server=163.26.68.15
  ```

- [chntpw](https://en.wikipedia.org/wiki/Chntpw)
  * [Reset Windows 7 Admin Password with Ubuntu Live CD/USB](http://www.chntpw.com/reset-windows-7-admin-password-with-ubuntu/)
  * [How to Hack a Windows 7/8/2008/10 Admin Account Password with Windows Magnifier](https://cx2h.wordpress.com/2015/04/02/how-to-hack-a-windows-782008-admin-account-password-with-windows-magnifier/)
- [憶傑科技 - VHD 無硬碟系統](https://sites.google.com/a/vhdsoft.com/web/)
- [[網路工具] 如何使用透過網路(LAN)喚醒功能(Wake on LAN)](https://www.asus.com/tw/support/FAQ/1009775)
- [ATA over Ethernet](https://en.wikipedia.org/wiki/ATA_over_Ethernet)
  * [WinAoE - AoE Windows Driver](https://winaoe.org/)
- [Visual BCD Editor](https://www.boyans.net/)
- [ProductPolicy viewer](http://reboot.pro/topic/20585-productpolicy-viewer/?p=196418)
- [PsTools](https://docs.microsoft.com/en-us/sysinternals/downloads/pstools) / 遠端執行桌面程式

  ```
  $ ./PsExec -accepteula
  $ ./PsExec -i 1 -s notepad.exe
  ```

## About
HSIEH, Li-Yi @進學國小資訊組
