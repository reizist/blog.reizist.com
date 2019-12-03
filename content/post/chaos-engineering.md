---
title: "Chaos Engineering"
date: 2018-11-26T09:36:24-08:00
draft: true
---

## 導入

問題が発生したときに何をすればよいか

* ドメインの作り忘れ
* SSL証明書の更新忘れと期限切れ

"You can not legislate against failure, focus on fast detection and response."


## 歴史とともに本紹介
* Drift Into Failure
* Release it!
* [principales of Chaost Engineering](https://principlesofchaos.org/)


## Failure layers
* Operations
* Application
* Software stack
* Infrastructure

* Mitigation Layers
  * Application level replication
  * Structured database replication
  * Storage block level replication

## 歴史
### Past: Disaster recovery
RPO:
Recovery Point objective
time interval between recovery point snapshots; e.g. daily backups

RTO:
Recovery Time Objective
Time taken to recovery after a failure; e.g. time to locate and restore from backup

### Present: Chaos engineering
chaos engineeringの歴史年表
Netflixのchaos engineeringの実装-OSS化-Gremlin Incの創設-本の出版に至るまで

#### アーキテクチャ
* チーム構成
  * People
  * Application
  * Switching
  * Infrastructure

* Failures are a  system problem - lack of safety margin
* Hypothesis(仮説) testing


### Gremlin Inc
Failure as service
Resource, network, state attacks on hosts or containers
Application-level fault injection for serverless
UIやAPI経由で

グレムリンのデモ - パケロスの設定をしてインターネットに問題がある環境再現



### Future: Reslient(弾力) critical systems


