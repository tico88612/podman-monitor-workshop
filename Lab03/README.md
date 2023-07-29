# Lab 03: Blackbox exporter

<!-- TOC -->

- [Lab 03: Blackbox exporter](#lab-03-blackbox-exporter)
    - [簡介](#%E7%B0%A1%E4%BB%8B)
        - [Blackbox exporter](#blackbox-exporter)
    - [實驗步驟](#%E5%AF%A6%E9%A9%97%E6%AD%A5%E9%A9%9F)
        - [安裝 Blackbox exporter](#%E5%AE%89%E8%A3%9D-blackbox-exporter)
        - [加入 Dashboard](#%E5%8A%A0%E5%85%A5-dashboard)
    - [原理講解](#%E5%8E%9F%E7%90%86%E8%AC%9B%E8%A7%A3)
        - [Prometheus](#prometheus)
        - [Grafana dashboard](#grafana-dashboard)
    - [Reference](#reference)

<!-- /TOC -->

## 簡介

### Blackbox exporter

Blackbox 顧名思義就是「黑盒子」，詞語緣起於「黑箱測試 (Black-box testing)」，這裡面是看不到測試過程，只需要知道服務目前是好的還是壞的。

Blackbox exporter 支援 HTTP、HTTPS、DNS、TCP、ICMP 等協議，檢測方式可以自行撰寫，或者使用[官方寫好的檢測方法](https://github.com/prometheus/blackbox_exporter/blob/master/blackbox.yml)。

這裡的實驗會教怎麼安裝 Blackbox exporter，用來檢測筆者架設測試用的[網站 (https://httptest.yjerry.tw)](https://httptest.yjerry.tw)，後面步驟會講解怎麼讓 Prometheus 知道去蒐集 metrics 資料，資料蒐集完成後，如何放到 Grafana 上呈現，最後 Lab 05 會呈現怎麼實現及時警報功能。

## 實驗步驟

### 安裝 Blackbox exporter

1. 進入 `attachments/kube` 資料夾

```bash
cd attachments/kube
```

2. 啟用 Blackbox exporter

```bash
sudo podman kube play blackbox-exporter.yml
```

3. 更新 Prometheus 和 Prometheus ConfigMap

```bash
sudo podman kube play --replace prometheus-kube.yml --configmap prom-cm.yml
```

### 加入 Dashboard

1. 進入 Grafana 點擊左側選單的 Dashboards

2. 右邊 `New` -> `Import`

3. `Import via grafana.com` 欄位輸入代號 `14928` 並按下 `Load`

4. Prometheus 點擊你的 Datasource 並按下 `Import`

## 原理講解

### Prometheus

裝完 Blackbox exporter 以後，Prometheus 並不會知道它的存在，需要設定給 `scrape_configs`。

- metrics_path：預設為 `/metrics`，Blackbox 設計為 `/probe`，要設定覆蓋過去。
- params：HTTP Get 變數
- static_configs：目標連結
- relabel_configs：重新標籤

```yaml
- job_name: 'blackbox'
  metrics_path: /probe
  params: # Get
    module: [http_2xx]
  static_configs:
    - targets:
      - https://httptest.yjerry.tw
  relabel_configs:
    - source_labels: [__address__] # 1. 使用 `__address__` 的值
      target_label: __param_target # 取代 `__param_target`
    - source_labels: [__param_target] # 2. 使用 `__param_target` 的值
      target_label: instance # 取代 `instance`
    - target_label: __address__ # 3. 修改 `__address__` 的值
      replacement: blackbox-exporter:9115 # 改為 `blackbox-exporter:9115`
```

原始連結：`https://httptest.yjerry.tw/probe?module=http_2xx`

1. 使用 `__address__` 的值取代 `__param_target`，目前結果為：`https://httptest.yjerry.tw/probe?module=http_2xx&target=https://httptest.yjerry.tw`
2. 使用 `__param_target` 的值取代 `instance`，這步驟就是把 instance 替換掉，換成 `https://httptest.yjerry.tw`
3. 修改 `__address__` 的值改為 `blackbox-exporter:9115`，最後結果為：`blackbox-exporter:9115/probe?module=http_2xx&target=https://httptest.yjerry.tw`

如果今天沒有 `relabel_configs` 就會需要寫很多 `scrape_configs`，而 relabel 後只需要在 `targets` 設定需要蒐集的監控連結即可。

### Grafana dashboard

使用 exporter 的時候可以上網找一下有沒有人寫好的 dashboard，拿下來修改會省事很多，除非真的沒有就是重新設計。

## Reference

- https://godleon.github.io/blog/Prometheus/Prometheus-Relabel/
- https://prometheus.io/docs/guides/multi-target-exporter/
