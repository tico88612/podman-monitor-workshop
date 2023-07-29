# Lab 02: Grafana + Podman secret

<!-- TOC -->

- [Lab 02: Grafana + Podman secret](#lab-02-grafana--podman-secret)
    - [簡介](#%E7%B0%A1%E4%BB%8B)
        - [Grafana](#grafana)
    - [實驗步驟](#%E5%AF%A6%E9%A9%97%E6%AD%A5%E9%A9%9F)
        - [安裝 Grafana](#%E5%AE%89%E8%A3%9D-grafana)
        - [加入 DataSource](#%E5%8A%A0%E5%85%A5-datasource)
    - [原理講解](#%E5%8E%9F%E7%90%86%E8%AC%9B%E8%A7%A3)
        - [Secret](#secret)
    - [Reference](#reference)

<!-- /TOC -->

## 簡介

### Grafana

Grafana 是個開源的分析、監控視覺化工具，同時支援很多資料來源 (datasource) E.g. InfluxDB, Loki, Prometheus, MySQL 等。

常見監控做法像 Lab 01 的那張圖，Prometheus 會去蒐集資料，存放到資料庫裡面，透過 API 呼叫或 Grafana 來拉取圖表呈現。

現在就來實裝圖形化工具 Grafana 吧！

## 實驗步驟

### 安裝 Grafana

1. 進入 `attachments/kube` 資料夾

```bash
cd attachments/kube
```

2. 編輯 `grafana.yml` 把 `<REPLACE_YOUR_IP>` 改成你的 IP

```bash
sed -i 's/<REPLACE_YOUR_IP>/103.122.117.132/' grafana.yml
vim grafana.yml
```

3. 編輯 `gf-secret.yml` 把當中的內容換成你的帳號密碼（要經過 Base64 編碼過）

```bash
echo -n "admin" | base64
echo -n "mypassword" | base64
vim gf-secret.yml
```

4. 啟動 Grafana

```bash
sudo podman kube play gf-data-pvc.yml
sudo podman kube play gf-secret.yml
sudo podman kube play grafana.yml
```

### 加入 DataSource

1. 登入 Grafana
2. 左側選單選擇 Connections > Your connections
3. 按下 Add new data source 選擇 Prometheus
4. URL 打上 `http://prometheus:9090`
5. 按下 `Save & test`

## 原理講解

Lab 02 安裝了 Grafana，跟 Lab 01 相比，多了 Secret 類別，下面就來介紹使用情境。

### Secret

> 注意：Base64 是一種編碼方式 (encoding)，中文要使用的詞是「編碼 (encode)」、「解碼 (decode)」而非「加密 (encrypt)」、「解密 (decrypt)」。

顧名思義 Secret 是存放機密資料，像是密碼、Token、TLS 金鑰等，需要隱私性的東西就會使用，內容都需要轉換為 Base64 編碼 (encode)。

其實跟 ConfigMap 一樣可以用於環境變數 (environment variable) 和資料卷 (volume)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gf-secret # secret name
data:
  username: YWRtaW4=
  password: bXlwYXNzd29yZA==
```

Pod 當中使用 `valueFrom.secretKeyRef` 裡面的 `name` 和 `key` 就可以引用：

```yaml
...
  containers:
  - image: docker.io/grafana/grafana:9.5.2
    name: grafana
    ...
    env:
    - name: GF_SECURITY_ADMIN_USER
      valueFrom:
        secretKeyRef:
          name: gf-secret # secret name
          key: username
...
```

筆者去翻了 [GitHub issue](https://github.com/containers/podman/issues/18387) 和 [Podman 程式碼](https://github.com/containers/common/tree/main/pkg/secrets)，似乎是有實作其他的 driver 並使用 GPG 加密（預設 driver 是使用 file 並且沒加密），但 [Podman 文件 (v4.5)](https://docs.podman.io/en/stable/markdown/podman-secret-create.1.html) 上並沒有寫太多，估計是 undocumented，直到[最近有成員把這部分文件補上](https://github.com/containers/podman/issues/18387#issuecomment-1648029534)，會使用的人也可以去貢獻看看。

以 K8s 的實作來說，Secret 有實作加密功能，但預設功能是關閉（基本上會跟 ConfigMap 一樣），如果有 etcd 存取權限基本上都拿得到資料。需要把靜態加密 (encryption at rest) 啟用。

## Reference

- https://github.com/containers/podman/issues/18387
- https://github.com/containers/common/tree/main/pkg/secrets
- https://docs.podman.io/en/stable/markdown/podman-secret-create.1.html
