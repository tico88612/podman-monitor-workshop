# Prepare the experimental environment

全程只需要一般權限使用者（需要 sudoer）即可

## Podman 安裝

1. 加入 apt keyrings 和 source list

```bash
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.opensuse.org/repositories/devel:kubic:libcontainers:unstable/xUbuntu_$(lsb_release -rs)/Release.key \
  | gpg --dearmor \
  | sudo tee /etc/apt/keyrings/devel_kubic_libcontainers_unstable.gpg > /dev/null
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/devel_kubic_libcontainers_unstable.gpg]\
    https://download.opensuse.org/repositories/devel:kubic:libcontainers:unstable/xUbuntu_$(lsb_release -rs)/ /" \
  | sudo tee /etc/apt/sources.list.d/devel:kubic:libcontainers:unstable.list > /dev/null
```

2. 安裝 Podman

```bash
sudo apt update
sudo apt install podman -y
```

3. 安裝 rootless 必要套件（可選）

```bash
sudo apt install uidmap fuse-overlayfs slirp4netns -y
```

4. 測試 Podman 是否完成安裝

```bash
sudo podman run -dt --name nginx-test -p 8080:80 nginx:1.25.1
```

打完後，系統會提示這訊息

```
ubuntu@podman-workshop:~$ sudo podman run -dt --name nginx-test -p 8080:80 nginx:1.25.1
? Please select an image:
  ▸ registry.fedoraproject.org/nginx:1.25.1
    registry.access.redhat.com/nginx:1.25.1
    docker.io/library/nginx:1.25.1
    quay.io/nginx:1.25.1
```

因為沒有打上映像檔 (image) 的 registry，過去在 Docker 中會預設使用 `docker.io`，但在 Podman 來說會詢問你要使用哪邊的 registry，如果你要使用 Docker registry 就打完整吧。

```bash
sudo podman run -dt --name nginx-test -p 8080:80 docker.io/library/nginx:1.25.1
```

5. 用 `podman ps -a` 確認是否有開啟

```bash
sudo podman ps -a
```

6. 刪除 container

```bash
sudo podman stop nginx-test
sudo podman rm nginx-test
```
