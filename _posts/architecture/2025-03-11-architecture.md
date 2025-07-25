---
title: "알림 속도 개선"
description: "이 글에서는 기존 API 방식으로 알림을 발송할 때 발생한 성능 문제를 해결하기 위해, 발송 기능을 RabbitMQ로 전환한 이유에 대해 설명합니다."
date: 2025-03-11 +00:00:00
permalink: /posts/2025-03-11-architecture/
mermaid: true
categories: [Blogging,architecture]
---

### 문제점: 제한된 API 처리 속도

알림 시스템으로 발송 방식은 API로 요청을 받아 큐에 넣는 방식을 사용했습니다. 그러나 이 방식은 1000건의 알림 발송 시 초당 **50 TPS**로 처리 속도가 제한되는 한계가 있었습니다. 특히 메신저 서비스에서 대량의 알림 발송 요청이 들어올 경우, API 처리 병목 현상으로 인해 시스템 성능이 크게 저하되었습니다.

- **발송 TPS 확인 (키바나)**: 초당 요청수 지표

  ![스크린샷 2024-09-05 오후 2.37.48.png](/assets/img/architecture/2025-03-11-architecture-01.png)

  ![스크린샷 2024-09-05 오후 2.37.31.png](/assets/img/architecture/2025-03-11-architecture-02.png)


- 기존 발송 아키텍쳐

![notification-Page-1.drawio (5).png](/assets/img/architecture/2025-03-11-architecture-03.png)
### 해결책: 발송 기능의 RabbitMQ 전환

성능 문제를 해결하기 위해 **발송 기능**을 RabbitMQ로 전환했습니다. RabbitMQ는 **메시지 큐 시스템**을 활용해 발송 요청을 **비동기 방식**으로 처리하며, 대량의 요청도 안정적으로 분산 처리할 수 있습니다. 또한, **RabbitMQ 모니터링 시스템**을 구축하여 발송 상태를 지속적으로 관리하고 장애를 신속히 대응할 수 있도록 했습니다.

기존 아키텍처에서는 **알림 API 서버**가 발송과 조회 기능을 함께 제공했으나, 이번 전환을 통해 **조회 기능**은 API 서버에 유지하고, **발송 기능**은 RabbitMQ를 활용한 비동기 처리 방식으로 분리했습니다. 이로 인해 시스템 부하를 줄이고, 더 많은 발송 요청을 안정적으로 처리할 수 있는 구조로 개선되었습니다.

- 변경 발송 아키텍쳐

![notification-Page-1의 복사본.drawio (11).png](/assets/img/architecture/2025-03-11-architecture-04.png)

### 필요한 추가 설정

- **RabbitMQ 서버 확장 및 클러스터링**: RabbitMQ 서버 한 대로는 부하 집중 문제가 발생할 수 있으므로, 서버를 **스케일 아웃**하고 **클러스터링**을 통해 부하를 분산했습니다.
- **모니터링 시스템 구축**: RabbitMQ와 시스템의 성능을 실시간으로 모니터링하기 위해 **Grafana**와 **Prometheus**를 사용해 모니터링 서버를 구축했습니다. 이를 통해 시스템의 상태를 지속적으로 관리하고, 이상 징후가 발생할 경우 빠르게 대응할 수 있도록 했습니다.

  ![SCR-20250311-sngr.png](/assets/img/architecture/2025-03-11-architecture-05.png)

- **rabbitMQ 발송 인터페이스 개발:** 서비스단에서 알림 시스템에 있는 rabbitMQ에 큐에 메세징을 넣을 수 있도록 spring framework에서 적용가능한 인터페이스를 개발하여 템플릿과 함께 제공했습니다.

### Kafka에서 RabbitMQ로 통합이유

현재 시스템은 Kafka 1.1.0 레거시 버전을 사용 중이며, **Broker와 ZooKeeper가 동일한 서버에서 실행되고 있습니다.** 이로 인해 장애 복구 및 버전 업그레이드 과정에서 어려움이 발생할 가능성이 있습니다. **클러스터링 및 모니터링 서버까지 구성된 RabbitMQ로 전환하는 것이 유지보수와 관리 측면에서 더 유리하다고 판단하였습니다.**

더불어, 알림 시스템의 발송량이 **수백만 건 규모에 미치지 않으며**, **실시간 처리가 중요한 요구사항**임을 고려해 Kafka 대신 RabbitMQ로 전환하기로 결정하였습니다. 이를 통해 Kafka를 사용하는 기존 Golang 서버를 RabbitMQ로 통합하여 운영 효율성을 높이고자 합니다.

### 성능 비교: 개선 전 vs 개선 후

jmeter를 이용해서 개발기를 TPS가 얼마나 차이 나는지 측정해보았습니다.

1000건에 대한 TPS

- 기존 API 발송 TPS

![스크린샷 2024-12-04 오후 5.40.33.png](/assets/img/architecture/2025-03-11-architecture-06.png)

- RabbitMQ 발송 TPS

![스크린샷 2024-12-04 오후 5.41.06.png](/assets/img/architecture/2025-03-11-architecture-07.png)

### 개선결과

API 처리량(TPS)이 기존 **36**에서 **RabbitMQ를 활용한 비동기 처리 방식으로** **9259 TPS**까지 크게 향상되었습니다. 이는 약 **25,688% 증가**한 수치입니다.

데몬 서버가 증가한 RabbitMQ 메시지 처리 속도를 안정적으로 유지할 수 있도록 **Grafana를 활용한 지속적인 모니터링**을 진행할 계획입니다. 이를 통해 **메시지 소비 속도, 서버 리소스 사용량** 등을 실시간으로 분석하고 지속적인 개선을 통해 **최대 처리량을 유지하면서도 안정적인 운영이 가능하도록 최적화해 나갈 계획입니다.**
