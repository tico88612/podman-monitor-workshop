# Lab 04: Podman exporter + Podman socket

<!-- TOC -->

- [Lab 04: Podman exporter + Podman socket](#lab-04-podman-exporter--podman-socket)
    - [簡介](#%E7%B0%A1%E4%BB%8B)
        - [Podman exporter](#podman-exporter)
        - [Podman socket](#podman-socket)
    - [實驗步驟](#%E5%AF%A6%E9%A9%97%E6%AD%A5%E9%A9%9F)
        - [開啟 Podman socket](#%E9%96%8B%E5%95%9F-podman-socket)
        - [安裝 Podman exporter + Node exporter](#%E5%AE%89%E8%A3%9D-podman-exporter--node-exporter)
        - [匯入 Dashboard](#%E5%8C%AF%E5%85%A5-dashboard)
    - [原理講解](#%E5%8E%9F%E7%90%86%E8%AC%9B%E8%A7%A3)
        - [Podman exporter](#podman-exporter)
        - [Grafana dashboard](#grafana-dashboard)
    - [Reference](#reference)

<!-- /TOC -->

## 簡介

### Podman exporter

Podman exporter 可以把 Podman 的容器 (containers)、pods、映像檔 (images)、資料卷 (volumes) 和網路 (networks) 相關資訊擷取，由 Prometheus 蒐集這些 metrics。

Podman exporter 適用於 Podman v4.x 版本，如果裝在容器內部，則需要開啟 `podman.socket`，位置在 `/run/podman/podman.sock`，那要如何把 host 上的路徑掛載到 container 內部呢？現在就來實作吧！

## 實驗步驟

### 開啟 Podman socket

```bash
sudo systemctl start podman.socket
sudo systemctl enabled podman.socket # 開機時自動啟動
```

### 安裝 Podman exporter + Node exporter

1. 進入 `attachments/kube` 資料夾

```bash
cd attachments/kube
```

2. 安裝 Podman exporter

```bash
sudo podman kube play podman-exporter.yml
```

3. 安裝 Node exporter

```bash
sudo podman kube play node-exporter.yml
```

4. 更新 Prometheus 和 Prometheus ConfigMap

```bash
sudo podman kube play --replace prometheus-kube.yml --configmap prom-cm.yml
```

### 匯入 Dashboard

1. 點到 `attachments/grafana` 裡面有個 `.json` 檔案，把內容貼上 `Import via panel json` 或者把檔案拉到 `Upload dashboard JSON file`。

2. 選擇 Datasource

## 原理講解

### Podman exporter

根據官網說明，預設只會啟用 container 收集器，需要使用 `-a` 或 `--collector.enable-all` 來啟用全部：

```yaml
...
  containers:
  - args:
    - "-a"
...
```

Container 當中的 `securityContext` 因為 Podman 使用 `root`，這裡就要使用 `root`，`runAsUser` 和 `runAsGroup` 都要設定為 0：

```yaml
...
    securityContext:
      runAsGroup: 0
      runAsUser: 0
...
```

要如何把系統上的資料夾直接對到 container 裡面？可以使用 hostPath 做對應：

```yaml
...
    volumeMounts:
    - mountPath: /run/podman/podman.sock # In container
      name: podman-socket-host
  volumes:
  - hostPath:
      path: /run/podman/podman.sock # In host path
      type: Directory
    name: podman-socket-host
```

### Grafana dashboard 

原始 dashboard 來源是 17639，但筆者在使用時，它的資料來源選擇是壞掉的，最後自己手動修復，並且新增了 Node exporter 變數。

其中有個 Disk Percentage，筆者認為這部分原本設計不實用，因為從敘述句你可以看到，它把每個映像檔容量加總除以節點的每個 filesystem 的容量總和（理當上後者不應該是總和，而是開掛載碟的容量大小）。

`sum(podman_image_size{instance=~"$PodmanExporter"}) / sum(node_filesystem_size_bytes{instance=~"$NodeExporter"})`

這樣只有計算到映像檔容量，其他檔案並不會計算到，較實用的方式應該要是目前已使用的容量除以硬碟容量

`1 - (1 * ((node_filesystem_avail_bytes{mountpoint="/",fstype!="rootfs"} )  / (node_filesystem_size_bytes{mountpoint="/",fstype!="rootfs"}) ))`

## Reference

- https://github.com/containers/prometheus-podman-exporter
- https://hackmd.io/@saschagrunert/Sk7JG_-rF
- https://stackoverflow.com/questions/57357532/get-total-and-free-disk-space-using-prometheus
