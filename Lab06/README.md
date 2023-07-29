# Lab 06: Rootless 實驗

<!-- TOC -->

- [Lab 06: Rootless 實驗](#lab-06-rootless-%E5%AF%A6%E9%A9%97)
    - [簡介](#%E7%B0%A1%E4%BB%8B)
        - [Rootless 容器](#rootless-%E5%AE%B9%E5%99%A8)
    - [前置作業](#%E5%89%8D%E7%BD%AE%E4%BD%9C%E6%A5%AD)
    - [實驗流程](#%E5%AF%A6%E9%A9%97%E6%B5%81%E7%A8%8B)
        - [建立普通使用者](#%E5%BB%BA%E7%AB%8B%E6%99%AE%E9%80%9A%E4%BD%BF%E7%94%A8%E8%80%85)
        - [登入 tester](#%E7%99%BB%E5%85%A5-tester)
        - [用 tester 操作 Podman](#%E7%94%A8-tester-%E6%93%8D%E4%BD%9C-podman)
    - [原理講解](#%E5%8E%9F%E7%90%86%E8%AC%9B%E8%A7%A3)
        - [Rootless 部分副作用](#rootless-%E9%83%A8%E5%88%86%E5%89%AF%E4%BD%9C%E7%94%A8)
        - [Rootless 資料卷位置](#rootless-%E8%B3%87%E6%96%99%E5%8D%B7%E4%BD%8D%E7%BD%AE)
        - [Rootless 好用嗎](#rootless-%E5%A5%BD%E7%94%A8%E5%97%8E)
    - [Reference](#reference)

<!-- /TOC -->

## 簡介

### Rootless 容器


## 前置作業

記得先把 `prepare/README.md` rootless 必要套件安裝完成。

直接輸入 `podman ps -a` 會跳出此訊息

```
ERRO[0000] cannot find UID/GID for user tester: no subuid ranges found for user "tester" in /etc/subuid - check rootless mode in man pages.
WARN[0000] Using rootless single mapping into the namespace. This might break some images. Check /etc/subuid and /etc/subgid for adding sub*ids if not using a network user
Error: unknown shorthand flag: 'a' in -a
See 'podman --help'
WARN[0000] Failed to add pause process to systemd sandbox cgroup: dial unix /run/user/1001/bus: connect: no such file or directory
```

## 實驗流程

### 建立普通使用者

1. 建立名為 `tester` 的一般使用者

```bash
sudo adduser tester
```

系統訊息：

```
Adding user `tester' ...
Adding new group `tester' (1001) ...
Adding new user `tester' (1001) with group `tester' ...
Creating home directory `/home/tester' ...
Copying files from `/etc/skel' ...
New password:
Retype new password:
passwd: password updated successfully
Changing the user information for tester
Enter the new value, or press ENTER for the default
        Full Name []: 
        Room Number []:
        Work Phone []:
        Home Phone []:
        Other []:
Is the information correct? [Y/n] Y
```

2. 切換使用者

```bash
sudo su tester -l
```

3. 加入 authorized keys

```bash
mkdir -p ~/.ssh
echo "ssh ed..." > ~/.ssh/authorized_keys
```

4. 登出本台伺服器

```bash
tester@podman-workshop:~$ exit
logout
ubuntu@podman-workshop:~$ exit
logout
Connection to 103.122.117.132 closed.
```

### 登入 `tester`

1. 用今天給的 private key 登入 `tester`

```bash
ssh tester@103.122.117.132
```

2. 確認 rootless 打開

```bash
podman info | grep rootless
```

範例訊息

```bash
tester@podman-workshop:~$ podman info | grep rootless
    rootless: true
```

3. 開始使用 Podman！

### 用 `tester` 操作 Podman

1. 把 `attachments/kube/nginx.yml` 內容以 Podman 執行。

```bash
podman kube play nginx.yml
```

2. 瀏覽 8081 port 內容，應為 Welcome Nginx 頁面。

3. 把 Lab 01 的 hostPort 改到 8090 port 執行，並且使用 kube play。

```bash
podman kube play prom-data-pvc.yml
podman kube play prometheus-kube.yml --configmap prom-cm.yml
```

4. 瀏覽 8090 port 內容，應為 Prometheus 頁面。

5. 接下來登出機器。

6. 過 10 ~ 20 秒後回去看 8081 port 是否還有 Welcome Nginx 頁面，或者 8090 port 是否還有 Prometheus 頁面。

## 原理講解

過程發生什麼事？為什麼我的 container 突然掛了？

### Rootless 部分副作用

我們可以用 `journalctl` 來看 log，重新登入 `tester` 看紀錄。

```bash
journalctl -r
```

可以用 log 看到，因為 Podman 執行 container 的 crun 開在 user space 下，使用者登出後 crun 會跟著一起被回收，就會看到 container 被 exited 情形。

### Rootless 資料卷位置

資料卷 (volume) 在 user space 下，預設會儲存在 `~/.local/share/containers/storage/volumes`。

### Rootless 好用嗎

一般情況來說，只有執行開發、大多數應用情境就可以適用，至少不需要讓普通使用者有多餘的權限去操作容器，Ubuntu 系統下設定不算複雜。

不過這部分今天主題有所違背，因為監控跟警報不一定會在使用者登入的情況下發生，因此 rootless 最後才會介紹。

詳細文件可以去看 Podman 的 GitHub repo 說明。

## Reference

- https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md
