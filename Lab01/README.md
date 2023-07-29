# Lab 01: Prometheus + Podman Kube

<!-- TOC -->

- [Lab 01: Prometheus + Podman Kube](#lab-01-prometheus--podman-kube)
    - [簡介套件 & 部署方式](#%E7%B0%A1%E4%BB%8B%E5%A5%97%E4%BB%B6--%E9%83%A8%E7%BD%B2%E6%96%B9%E5%BC%8F)
        - [Podman Compose](#podman-compose)
        - [Podman Kube](#podman-kube)
        - [Prometheus](#prometheus)
    - [設計架構](#%E8%A8%AD%E8%A8%88%E6%9E%B6%E6%A7%8B)
    - [安裝 Prometheus 在 Podman](#%E5%AE%89%E8%A3%9D-prometheus-%E5%9C%A8-podman)
    - [原理講解](#%E5%8E%9F%E7%90%86%E8%AC%9B%E8%A7%A3)
        - [ConfigMap](#configmap)
            - [環境變數 Environment variable](#%E7%92%B0%E5%A2%83%E8%AE%8A%E6%95%B8-environment-variable)
            - [資料卷 Volume](#%E8%B3%87%E6%96%99%E5%8D%B7-volume)
        - [PersistentVolumeClaim](#persistentvolumeclaim)
        - [Pod](#pod)
    - [Reference](#reference)

<!-- /TOC -->

## 簡介套件 & 部署方式

Podman 筆者所知道的部署方式有三種：

1. Podman
2. Podman Compose
3. Podman Kube

### Podman Compose

Podman Compose 跟 Docker Compose 會很接近，但這部分屬於社群維護並非 Podman 團隊管理。雖然是基於 Compose 的 spec 實作，但內容支援性還沒有到齊全（E.g. top level 的 configs 目前沒有寫，secrets 會受到 spec 限制，跟後面要說的 kube 支援度差異很多）。

### Podman Kube

Kube 全名就是 Kubernetes，也就是 K8s，用 K8s 的 YAML 來做部署，這部分由 Podman 團隊維護，這也是 Red Hat 目前推薦的方式。

目前官網上有這些種類可以支援：

- Pod
- Deployment（複製出數個相同的 Pod）
- PersistentVolumeClaim（儲存）
- ConfigMap（應用程式設定檔）
- Secret（機密）

如果對於 K8s 不熟悉也沒關係，筆者這次都會幫你準備，讓你可以動手實作。

### Prometheus

Prometheus 是個 Time-series 的 database，最早是由 SoundCloud 研發，後續許多開發者青睞使用，2016 年加入 CNCF 專案中，2018 年從 CNCF 專案畢業 (graduated)。

以時間作為資料儲存方式，可以是記錄連續或不連續資料。

- 連續資料：CPU、Memory、Network 等
- 非連續資料：Application log、Syslog 等

![](images/architecture.png)

Prometheus 主要會用在**連續資料**上，用於非連續資料也是可以，但筆者會推薦另一個 Log system - Loki，不過篇幅有限這次不會出現（需要的話問卷當中填寫回饋，說不定有機會在 CNTUG 辦 workshop 專場）。

## 設計架構

Lab 01 中我們將會用 Pod 來部署 Prometheus，為了把資料保存而非佔用 tmp 空間，會使用 PersistentVolumeClaim 放在系統內並掛載到 container 的 `/prometheus`，ConfigMap 用於 Promtheus 應用程式設定檔。

## 安裝 Prometheus 在 Podman

1. 進入 `attachments/kube` 資料夾

```bash
cd attachments/kube
```

2. 把設定檔給 Podman 部署

```bash
sudo podman kube play prom-data-pvc.yml
sudo podman kube play prometheus-kube.yml --configmap prom-cm.yml
```

## 原理講解

最基礎的 Kube YAML 會有這幾項：

- apiVersion：承接 K8s 的設定，通常為 v1
- kind：說明這個 YAML 是哪種類別
- metadata：給予該項目名字 (name)、標籤 (label)、註解 (annotation)，其中註解某些情況會看特定字串。

底下就會根據不同的 kind，後方會有不同的寫法。

### ConfigMap

通常會用於「應用程式設定檔」，跟 Pod 有很多不同搭配方式，E.g. 環境變數 (environment variable)、資料卷 (volume) 等，除了純文字以外可以用二進位檔案 (binary data)。

純文字型態後面會接的是 `data`，如果要放入的是二進位檔案就會是 `binaryData`，把二進位資料使用 base64 做編碼 (encode) 放入 YAML。

#### 環境變數 (Environment variable)

`data` 中就會接 key-value，環境變數如果想要都從 ConfigMap 取得，就可以這樣寫：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: foo # ConfigMap name
data:
  FOO: bar # key: value
---
apiVersion: v1
kind: Pod
metadata:
  name: foobar
spec:
  containers:
  - name: container-1
    image: foobar
    envFrom:
    - configMapRef:
        name: foo # ConfigMap name
        optional: false
```

或者要一個一個設計對應（這樣可以混合 Secret 一起搭配）：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: foo # ConfigMap name
data:
  THIS_FOO: bar # key: value
---
apiVersion: v1
kind: Pod
metadata:
  name: foobar
spec:
  containers:
  - name: container-1
    image: foobar
    env:
    - name: FOO
      valueFrom:
        configMapKeyRef:
          name: foo # ConfigMap name
          key: THIS_FOO # which key
          optional: false
```

#### 資料卷 (Volume)

如果變成資料卷 (volume) 去掛載，key 就會變成檔案名稱，value 就會變成檔案內容。

以 Lab 01 來說，`prometheus.yml` 會變成檔案名稱，後方的 `global...` 則會變成檔案內容。 

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config # configmap name
data:
  prometheus.yml: |-
    global:
      scrape_interval: 15s
      scrape_timeout: 10s
...
```

要在 Pod 當中以資料卷 (volume) 去掛載 ConfigMap：

```yaml
...
  volumes:
  - name: config-volume # pod volume name
    configMap:
      name: prometheus-config # configmap name
...
```

最後在 container 中使用 `volumeMounts` 即可成功掛載：

```yaml
  containers:
  - image: docker.io/prom/prometheus:v2.44.0
    ...
    volumeMounts:
    - mountPath: /etc/prometheus # container path
      name: config-volume # pod volume name
```

要注意的是，Podman 目前沒有特別為 ConfigMap 做特定型態，不能像 K8s 可以是獨立的型態，必須要有 Pod 搭配才能使用。

### PersistentVolumeClaim

PersistentVolumeClaim 簡稱為 PVC，以 K8s 來說，PVC 需要搭配 PersistVolume (PV) 才能使用，不過 Podman 這段簡化成可以只用 PVC 做資料保存。

PVC 會看註解 (annotation)，`volume.podman.io/driver` 設定為 `local` 就代表以本機磁碟作為作為儲存，會存放在 `/var/lib/containers/storage/volumes`。

`accessModes` 是 K8s 設定，Podman 通常只要寫 `ReadWriteOnce`，其他選項 E.g. `ReadWriteMany` 基本上是沒有效果。

`resources.requests.storage` 是 K8s 設定，需要的磁碟空間，K8s 就會需要設定 PV 大小。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  annotations:
    volume.podman.io/driver: local
  name: prom-data # pvc name
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5G
```

掛載方式就要在 Pod 中說明有 volume，對應的 PVC：

```yaml
...
  volumes:
  - name: prom-data-pvc # pod volume name
    persistentVolumeClaim:
      claimName: prom-data # pvc name
...
```

接下來在 container 裡面寫上要掛載的路徑即可使用：

```yaml
...
  containers:
  - image: docker.io/prom/prometheus:v2.44.0
    name: prometheus
    ...
    volumeMounts:
    - mountPath: /prometheus # container path
      name: prom-data-pvc # pod volume name
...
```

### Pod

1 個 Pod 通常搭配 1 個 container，有些會為了做 Service Mesh 就會放多個 container。

- image：映像來源。
- name：容器名稱。
- command：相當於 Docker 中的 Entrypoint。
- args：相當於 Docker 中的 Cmd。
- securityContext：安全性配置，這部分筆者通常不會設定，可以對容器做細部的權限調整。
- volumeMounts：掛載容器卷 (volume) 到容器內。

```yaml
spec:
  containers:
  - image: docker.io/prom/prometheus:v2.44.0
    name: prometheus
    # command: [] # Docker Entrypoint
    # args: [] # Docker Cmd
    ports:
    - containerPort: 9090
      hostPort: 9090
    securityContext: {}
    volumeMounts:
    - mountPath: /etc/prometheus
      name: config-volume
  volumes:
  - name: config-volume
    configMap:
      name: prometheus-config
```

## Reference

- https://www.influxdata.com/time-series-database/
- https://medium.com/@henry-chou/%E4%BE%86%E5%BE%9Etimescaledb%E8%AA%8D%E8%AD%98time-series-database%E5%90%A7-603603506d09
- https://prometheus.io/docs/introduction/overview/
