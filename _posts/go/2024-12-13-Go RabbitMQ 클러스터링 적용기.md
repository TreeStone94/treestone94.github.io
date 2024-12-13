---
title: "Go RabbitMQ 클러스터링 적용기"
description: "rabbitmq를 단일서버에서 클러스터링 서버로 전환하면서 spring만 사용해왔던 저는 생소했던 Go 언어에 rabbitmq 클러스터링 어떻게 연결하고 적용했는지에 대해 공유하고자 합니다."
date: 2024-12-13 +21:00:00
mermaid: true
categories: [Blogging,go]
---


spring을 주로 할줄 알았던 저는 부서가 변경되면서 core업무도 같이 맡게 되었습니다. 그러면서 기존에 맡고 있던 알림 시스템 개발에 성능 개선을 위해 rabbitMQ 클러스러링을 전환하면서 알림 core쪽도 개선하기로 했습니다. 개선해야 할 부분은 두 가지로 판단되었습니다.

왜 성능개선을 하게되었는지는 다른 글로 알려드리도록 하겠습니다.

## 1. 공식 rabbitMQ 라이브러리 전환

`github.com/streadway/amqp` → [`github.com/rabbitmq/amqp091-go`](http://github.com/rabbitmq/amqp091-go) 

기존에 사용하던 라이브러리는 공식 rabbitmq 라이브러리가 아니고 버전관리를 하고 있지 않기 때문에 클러스터링으로 연결하면서 버전 및 이슈 관리를 위해 공식 golang rabbitmq 연결 라이브러리로 변경하였습니다.

## 2. 단일 서버에서 클러스터링 서버로 적용

단일 서버에서 클러스터링 서버로 전환하면서 중점적으로 고려한 포인트는 아래와 같습니다.

### rabbitMQ 서버 다운 시 golang 장애복구 메커니즘 적용

연결하고 있는 RabbitMQ 서버가 문제가 생겼을 때 클러스터링된 다른 서버로 연결 할 수 있도록 모니터링 로직 추가

```go
func (c *Consumer) monitorConnection() {
	for {
		select {
		case <-c.done:
			return
		default:
			// 연결 상태 확인
			if c.conn == nil || c.conn.IsClosed() {
				log.Error("Connection lost, attempting to reconnect...")

				// 재연결 시도
				if err := c.Connect(); err != nil {
					log.Errorf("Reconnection failed: %v", err)
					time.Sleep(5 * time.Second)
					continue
				}
				return
			}

			// 5초마다 연결 상태 확인
			time.Sleep(5 * time.Second)
		}
	}
}
```

**장애복구 시나리오**

1. golang 서버에서 연결하고 있는 임의로 서버 다운
    
    ![스크린샷 2024-12-13 오후 4.34.26.png](/assets/img/go/2024-12-13-Go%20RabbitMQ%20클러스터링%20적용기-01.png)
    
2. 장애복구
    
    ![스크린샷 2024-12-13 오후 4.36.44.png](/assets/img/go/2024-12-13-Go%20RabbitMQ%20클러스터링%20적용기-02.png)
    

시나리오대로 장애 복구 메커니즘을 통해 클러스터링된 RabbitMQ 서버들의 고가용성을 보장할 수 있는지 확인 할 수 있었습니다.

### Go언어 동시성, 병렬성 특징을 이용한 다중 채널 연결

고루틴을 통해 connection을 여러개를 두면 rabbitmq 서버에 리소스를 많이 차지하기에 가상연결 채널을 서버 cpu에 맞춰 golang에 적용

다중 채널을 위한 struct 생성

```go
type Consumer struct {
	urls          []string
	prefetchCount int
	conn          *amqp091.Connection
	channels      []*amqp091.Channel // 여러 채널로 변경
	handlers      map[string]func(amqp091.Delivery)
	mu            sync.RWMutex
	done          chan struct{}
	channelCount  int // 생성할 채널 수
}
```

클러스터링 서버 연결

```go
for _, url := range c.urls {
		conn, err := amqp091.DialConfig(url, amqp091.Config{
			Heartbeat: 10 * time.Second,
			Locale:    "en_US",
			Dial: func(network, addr string) (net.Conn, error) {
				return net.DialTimeout(network, addr, 5*time.Second)
			},
		})
		if err != nil {
			lastErr = fmt.Errorf("failed to connect to %s: %v", url, err)
			log.Errorf("Connection error: %v", lastErr)
			continue
		}
}
```

연결이후 CPU 수에 맞춰 다중 채널 생성

```go
// 다중 채널 생성
for i := 0; i < c.channelCount; i++ {
	ch, err := conn.Channel()
	if err != nil {
		conn.Close()
		lastErr = fmt.Errorf("failed to create channel %d on %s: %v", i, url, err)
		log.Errorf("Channel error: %v", lastErr)
		break
	}

	if err := ch.Qos(c.prefetchCount, 0, false); err != nil {
		ch.Close()
		conn.Close()
		lastErr = fmt.Errorf("failed to set QoS on channel %d of %s: %v", i, url, err)
		log.Errorf("QoS error: %v", lastErr)
		break
	}

	c.channels[i] = ch
}
```

**RabbitMQ 매니저 페이지에서 연결 수 전후 비교**

![스크린샷 2024-12-13 오후 5.27.32.png](/assets/img/go/2024-12-13-Go%20RabbitMQ%20클러스터링%20적용기-03.png)

![스크린샷 2024-12-13 오후 5.29.13.png](/assets/img/go/2024-12-13-Go%20RabbitMQ%20클러스터링%20적용기-04.png)

이전 서버는 connection을 여러번 연결을 하여 사용하였다면 개선하면서 연결은 한번 다중 채널로 연결하면서 리소스 부하를 줄일 수 있었습니다

## 느낀점

예전에는 기능 개발만 하고 끝나는 경우가 많았지만, 이번에 Go 언어 서버에 RabbitMQ 클러스터링 서버를 적용하면서 단순히 연결하는 것뿐 아니라 여러 중요한 포인트를 고려해야 한다는 것을 깨닫게 되었습니다.

특히, 시스템의 가용성을 높이기 위해서는 사전 테스트와 장애 복구 시뮬레이션이 필수적이라는 점도 배웠습니다.

생소하게 느껴졌던 Go 언어에 대한 이해도 한층 높아졌고, 앞으로 더 친숙해지기 위해 지속적으로 노력할 계획입니다. 😂
