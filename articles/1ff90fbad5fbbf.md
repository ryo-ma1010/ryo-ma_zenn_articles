---
title: "WindowsPowerShellでLinuxコマンドを使う方法"
emoji: "😸"
type: "tech"
topics:
  - "linux"
  - "powershell"
  - "wsl2"
  - "linuxコマンド"
  - "windows11"
published: true
published_at: "2023-09-06 12:08"
---

## 環境
:::message
- Windows11
- WindowsPowerShell 5.1
- WSL2 (AlmaLinux)
:::

## 愚痴

WindowsPowershellはLinuxコマンド使えません。(一部エイリアスで対応)
Linuxに慣れてしまった体にはこの環境は受け入れられないです。

## 例

```powershell
> touch hoge.txt
touch : 用語 'touch' は、コマンドレット、関数、スクリプト ファイル、または操作可能なプログラムの名前として認識されません。名前が正しく記述されていることを確認し、パスが含まれている場合はそのパスが正しいことを
確認してから、再試行してください。
発生場所 行:1 文字:1
+ touch hoge.txt
+ ~~~~~
    + CategoryInfo          : ObjectNotFound: (touch:String) [], CommandNotFoundException
    + FullyQualifiedErrorId : CommandNotFoundException
```

どうにかLinuxコマンド使えないか、めちゃくちゃググってたら
どうやらwslコマンドを経由するとLinuxコマンドが使えるらしいことを知りました。

## wslコマンドを使う

```powershell
> wsl touch hoge.txt
> wsl ls -la | wsl grep hoge
-rwxrwxrwx 1 fuga fuga       0 Sep  6 11:41 hoge.txt
```

Linuxコマンド使えました！！！
これでWindowsPowershell使い倒せます！！
😊😊😊😊😊😊

## 追記
なお、topコマンドとdfコマンド叩いたらWSL側の結果が出力される模様

```bash
> wsl top
top - 16:56:46 up  2:53,  0 users,  load average: 0.06, 0.01, 0.00
Tasks:   5 total,   1 running,   4 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.0 us,  0.0 sy,  0.0 ni,100.0 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
MiB Mem :   7843.1 total,   5100.3 free,    341.0 used,   2401.8 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.   7228.6 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
    1 root      20   0    2324   1504   1404 S   0.0   0.0   0:00.00 init(AlmaLinux9
    4 root      20   0    2324      4      0 S   0.0   0.0   0:00.00 init
   10 root      20   0    2328    108      0 S   0.0   0.0   0:00.00 SessionLeader
   11 root      20   0    2344    112      0 S   0.0   0.0   0:00.00 Relay(12)
   12 nss       20   0    7408   3628   3012 R   0.0   0.0   0:00.00 top
```

```bash
> wsl df
Filesystem      1K-blocks     Used  Available Use% Mounted on
none              4015668        4    4015664   1% /mnt/wsl
none            242329588 76307576  166022012  32% /usr/lib/wsl/drivers
none              4015668        0    4015668   0% /usr/lib/wsl/lib
/dev/sdd       1055762868  1953080 1000106316   1% /
none              4015668       76    4015592   1% /mnt/wslg
rootfs            4012424     1936    4010488   1% /init
none              4012452        0    4012452   0% /dev
none              4015668        0    4015668   0% /run
none              4015668        0    4015668   0% /run/lock
none              4015668        0    4015668   0% /run/shm
none              4015668        0    4015668   0% /run/user
tmpfs             4015668        0    4015668   0% /sys/fs/cgroup
none              4015668       72    4015596   1% /mnt/wslg/versions.txt
none              4015668       72    4015596   1% /mnt/wslg/doc
drvfs           242329588 76307576  166022012  32% /mnt/c
```