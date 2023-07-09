---
title: Splunkにフィードされるネットワーク機器 (Cisco ASA) のログデータをConfluentで加工する実験環境
summary: "ネットワーク機器のログをSplunkのUniversal Forwarderを利用してConfluentに転送し、ストリーム処理後にSplunkのHECに転送するサンドボックス環境。Splunk Universal Forwarderから送られるログを、Confluent内で機器ログ (CISCO ASA) とUniversal Forwarder自身のログ (SPLUNKD) にストリーム処理で分類。ストリーム処理にはksqlDBを利用。"
authors:
  - hashi
tags:
  - Confluent Platform
  - ksqlDB
  - Splunk
  - Elasticsearch
  - SIEM
  - Stream Processing
date: '2021-09-23T00:00:00Z'

links:
url_code: 'https://github.com/confluent-jp/demo-cp-splunk-elastic'
url_pdf: ''
url_slides: ''
url_video: ''
---