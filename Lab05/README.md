# Lab 05: Alertmanager

<!-- TOC -->

- [Lab 05: Alertmanager](#lab-05-alertmanager)
    - [簡介](#%E7%B0%A1%E4%BB%8B)
        - [Alertmanager](#alertmanager)
    - [實驗步驟](#%E5%AF%A6%E9%A9%97%E6%AD%A5%E9%A9%9F)
        - [安裝 Alertmanager Discord](#%E5%AE%89%E8%A3%9D-alertmanager-discord)
        - [安裝 Alertmanager](#%E5%AE%89%E8%A3%9D-alertmanager)
    - [原理講解](#%E5%8E%9F%E7%90%86%E8%AC%9B%E8%A7%A3)
        - [Prometheus](#prometheus)
        - [Alertmanager](#alertmanager)
    - [Reference](#reference)

<!-- /TOC -->

## 簡介

### Alertmanager

Prometheus 搭配的警告工具，Alertmanager 支援 Email、Pagerduty、Webhook、Telegram 等傳送，依照等級程度可以發送即時訊息或者 Email 通知。

接下來我們會使用 Lab 03 的 Blackbox exporter 做為警告通知。

## 實驗步驟

### 安裝 Alertmanager Discord

1. 進入 `attachments/kube` 資料夾

```bash
cd attachments/kube
```

2. 更換成你的 Discord webhook URL（把下方的 `https://` 換成你的 webhook 連結）

```bash
sed -i 's,<YOUR_WEBHOOK_URL>,https://,' alertmanager-discord.yml
```

3. 啟動 Alertmanager Discord

```bash
sudo podman kube play alertmanager-discord.yml
```

### 安裝 Alertmanager

1. 啟動 Alertmanager

```bash
sudo podman kube play am-data-pvc.yml
sudo podman kube play --replace alertmanager.yml --configmap am-cm.yml
```

2. 更新 Prometheus 和 Prometheus ConfigMap

```bash
sudo podman kube play --replace prometheus-kube.yml --configmap prom-cm.yml
```

## 原理講解

### Prometheus

打開 `prometheus/prometheus.yml` 的 `alerting`，這部分就是連結 Alertmanager。

```yaml
...
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - alertmanager:9093

rule_files:
  - alert.rule.yml
...
```

接下來看到 `prometheus/alert.rule.yml`，`expr` 代表是條件，`for` 代表檢查到一定時間，就會呼叫警報，`labels` 可以設定重要程度 `severity`，用 `annotations` 的 `summary` 描述發生原因。

```yaml
groups:
- name: alert.rules
  rules:
  - alert: EndpointDown
    expr: probe_success == 0
    for: 10s
    labels:
      severity: "critical"
    annotations:
      summary: "Endpoint {{ $labels.instance }} down"

```

### Alertmanager

接下來講解 Alertmanager 設定檔，

- group_by：通常設定 `alertname` 區分，但如果有多個 Prometheus 同時使用，可以再多加一個標籤區分。
- group_wait：第一次收到之後等待 10 秒後發送警告（如果有其他 inhibit rule 會一起處理）。
- group_interval：重複檢測，每 10 秒會重新檢查警告是否還在。
- repeat_interval：重複通知時間，過了一小時才可以再發送通知。
- receiver：預設接收者
- routes：根據路徑配對，選擇要給哪一位接收者，如果沒有配對成功則走到預設接收者

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'discord' # Default receiver
  routes:
  - match:
      severity: critical 
    receiver: discord
  # - match_re: # regular expression
  #     severity: ^(warning|critical)$
  #   receiver: telegram
```

下面的 `receivers` 就是在定義接收者，可以使用 Webhook、Telegram 等方式，如果官方沒有提供方式，可以使用 Webhook 傳送。

```yaml
receivers:
  - name: 'discord'
    webhook_configs:
    - url: 'http://alertmanager-discord:9094'
  # - name: 'telegram'
  #   telegram_config:
  #   - bot_token: ''
  #   - chat_id: ''

inhibit_rules: []
```

`inhibit_rules` 這裡雖然填寫空白，這次情境不算複雜。如果今天 K8s 叢集掛掉，理當說只需要收到「K8s 叢集掛掉」通知，剩下「應用程式異常」或「其他錯誤」訊息不需要。

到這裡就恭喜你完成監控、警報的流程體驗了，可以根據服務系統的性質，設定不同的警報功能（E.g. container 的 CPU 用量不超過 70% 等）。

## Reference

- https://www.digitalocean.com/community/tutorials/how-to-use-alertmanager-and-blackbox-exporter-to-monitor-your-web-server-on-ubuntu-16-04#step-5-creating-alert-rules
- https://yunlzheng.gitbook.io/prometheus-book/parti-prometheus-ji-chu/alert/alert-manager-inhibit
