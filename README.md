
# [ 건강검진 예약 ]

Final Project AWS 3차수 - 1팀 제출자료입니다. 

## checkpoint
1.Saga

2.CQRS

3.Correlation

4.Req/Resp

5.Gateway

6.Deploy/ Pipeline -정연식

7.Circuit Breaker --진행필요

8.Autoscale (HPA) --임문희(완료)

9.Zero-downtime deploy (Readiness Probe) --임문희(완료)

10.Config Map/ Persistence Volume --이지혜(진행중)

11.Polyglot

12.Self-healing(Liveness Probe)   -- 임문희


# Table of contents

- [건강검진예약](#---)
  - [서비스 시나리오](#시나리오)
  - [분석/설계](#분석-설계)
  - [구현:](#구현)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [Correlation](#Corrlation)                         
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)       
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출--시간적-디커플링--장애격리--최종-eventual-일관성-테스트)
    - [API Gateway](#API-게이트웨이-(gateway))                            
    - [SAGA-CQRS](#마이페이지)   
    -                      
  - [운영](#운영)
    - [CI/CD 설정](#cicd-설정) 
    - [동기식 호출 / Circuit Breaker](#동기식-호출--Circuit-Breaker) 
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포(Readiness Probe)](#무정지-배포(Readiness-Probe))
    - [Self Healing(Liveness Probe)](#Self-Healing(Liveness-Probe))
    - [ConfigMap / Persistence Volume](#Configmap) 


## 시나리오

건강검진 예약 시스템에서 요구하는 기능/비기능 요구사항은 다음과 같습니다. 
사용자가 건강검진 예약 시 결제를 진행하면
검진센터 관리자가 예약을 확정하는 시스템입니다.
사용자는 진행상황을 확인할 수 있고, SMS로도 예약상태가 전송된다. 


#### 기능적 요구사항

1. 고객이 원하는 일자를 선택하고 예약한다.
2. 고객이 결제를 진행한다.
3. 예약이 신청 되면 신청내역이 검진센터에 전달된다. 
4. 검진센터 관리자가 신청내역을 확인하여 예약을 확정한다. 
5. 고객이 예약을 취소할 수 있다.
6. 예약이 취소 되면 검진 예약이 취소 된다.
7. 예약이 취소 되면 결제도 취소된다.
8. 고객이 예약 취소를 하면 예약 정보는 삭제 상태로 갱신 된다.
9. 고객이 예약 진행상태를 원할 때마다 조회한다.
10. 예약 상태가 변경되면 SMS로 알림이 발송된다.


#### 비 기능적 요구사항

1. 트랜잭션
   - 결제가 되지 않으면 검진예약 신청이 처리되지 않아야 한다. `Sync 호출`
   - 예약이 취소되면 결제도 취소가 되어야 한다. `Sync 호출`

2. 장애격리
   - 검진센터관리자 기능이 수행 되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다. `Pub/Sub`
   - 결제 시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도 한다.
     (장애처리)

3. 성능
   - 고객이 예약 확인 상태를 마이페이지에서 확인할 수 있어야 한다. `CQRS`
   - 예약 상태가 바뀔 때 마다 SMS로 알림이 발송되어야 한다.


# 분석 설계

## Event Storming

#### ver1 - 이벤트도출
 - MSAEZ 툴에서 이벤트스토밍 작업
 - 업무별 담당자를 분배하여 각 도메인별 command,event,aggregate,policy를 도출
 - 이후 java소스로의 컨버전을 고려하여 네이밍을 영문 대문자로 시작하는 것으로 명칭변경 적용
![ver1](https://github.com/mulcung03/AWS3_healthcenter/blob/main/refer/storming_1.JPG)

#### ver2 - relation정의
![ver2](https://github.com/mulcung03/AWS3_healthcenter/blob/main/refer/storming_2.JPG)

#### ver3 - attribute생성
![final](https://github.com/mulcung03/AWS3_healthcenter/blob/main/refer/storming_new.JPG)


### 기능 요구사항을 커버하는지 검증
1. 고객이 원하는 일자를 선택하고 예약한다.  --> O
2. 고객이 결제를 진행한다.  --> O
3. 예약이 신청 되면 신청내역이 검진센터에 전달된다.   --> O
4. 검진센터 관리자가 신청내역을 확인하여 예약을 확정한다.   --> O
5. 고객이 예약을 취소할 수 있다.  --> O
6. 예약이 취소 되면 검진 예약이 취소 된다.  --> O
7. 예약이 취소 되면 결제도 취소된다.  --> O
8. 고객이 예약 취소를 하면 예약 정보는 삭제 상태로 갱신 된다.  --> O
9. 고객이 예약 진행상태를 원할 때마다 조회한다.  --> O
10. 예약 상태가 변경되면 SMS로 알림이 발송된다.  --> O


### 비기능 요구사항을 커버하는지 검증

1. 트랜잭션
   - 결제가 되지 않으면 검진예약 신청이 처리되지 않아야 한다. `Sync 호출` --> O
   - 예약이 취소되면 결제도 취소가 되어야 한다. `Sync 호출` --> O
   ==>  Request-Response 방식 처리
   
2. 장애격리
   - 검진센터관리자 기능이 수행 되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다. `Pub/Sub` --> O
   ==>  Pub/Sub 방식으로 처리(Pub/Sub)
   - 결제 시스템이 과중되면 사용자를 잠시동안 받지 않고 결제를 잠시후에 하도록 유도 한다.
     (장애처리)

3. 성능
   - 고객이 예약 확인 상태를 마이페이지에서 확인할 수 있어야 한다. `CQRS`
   - 예약 상태가 바뀔 때 마다 SMS로 알림이 발송되어야 한다.


## 구현
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현.
각 서비스 별로 포트넘버 부여 확인 ( 8081 ~ 8084 )

### 포트넘버 분리
```
spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/** 
        - id: reservation
          uri: http://localhost:8082
          predicates:
            - Path=/reservations/**,/cancellations/** 
        - id: payment
          uri: http://localhost:8083
          predicates:
            - Path=/paymentHistories/** 
        - id: customer
          uri: http://localhost:8084
          predicates:
            - Path= /mypages/**
```

### 각 서비스를 수행
```
cd /home/project/health_center/order
mvn spring-boot:run

cd /home/project/health_center/payment
mvn spring-boot:run

cd /home/project/health_center/reservation
mvn spring-boot:run

cd /home/project/health_center/notification
mvn spring-boot:run

netstat -anlp | grep :808
```

## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다: 
- (예시는 order 마이크로 서비스).
```
package healthcenter;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

import healthcenter.external.PaymentHistory;

@Entity
@Table(name="Order_table")
public class Order {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private String orderType;
    private Long cardNo;
    private String name;
    private String status;

    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();

        //Following code causes dependency to external APIs
        // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.

        PaymentHistory payment = new PaymentHistory();
        System.out.println("this.id() : " + this.id);
        payment.setOrderId(this.id);
        payment.setStatus("Reservation OK");
        // mappings goes here
        OrderApplication.applicationContext.getBean(healthcenter.external.PaymentHistoryService.class)
            .pay(payment);


    }

    @PostUpdate
    public void onPostUpdate(){
    	System.out.println("Order Cancel  !!");
        OrderCanceled orderCanceled = new OrderCanceled();
        BeanUtils.copyProperties(this, orderCanceled);
        orderCanceled.publishAfterCommit();


    }


    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public String getOrderType() {
        return orderType;
    }

    public void setOrderType(String orderType) {
        this.orderType = orderType;
    }
    public Long getCardNo() {
        return cardNo;
    }

    public void setCardNo(Long cardNo) {
        this.cardNo = cardNo;
    }



    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }

}

```

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 
데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package healthcenter;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface OrderRepository extends PagingAndSortingRepository<Order, Long>{
}
```

- 적용 후 REST API 의 테스트 <<수정필요>>
```
# app 서비스의 주문처리
http localhost:8081/orders orderType=basic

# pay 서비스의 결제처리
http localhost:8083/paymentHistories orderId=1 price=50000 payMethod=card

# hotel 서비스의 예약처리
http localhost:8082/reservations orderId=1 status="confirmed"

# 주문 상태 확인(mypage)

http localhost:8081/orders/3
root@labs--1428063258:/home/project/healthcenter# http localhost:8084/mypages/1
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Mon, 21 Jun 2021 10:43:12 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "mypage": {
            "href": "http://localhost:8084/mypages/1"
        },
        "self": {
            "href": "http://localhost:8084/mypages/1"
        }
    },
    "name": null,
    "orderId": 1,
    "reservationId": 3,
    "status": "confirmed"
}


```

## 폴리글랏 퍼시스턴스
비지니스 로직은 내부에 순수한 형태로 구현
그 이외의 것을 어댑터 형식으로 설계 하여 해당 비지니스 로직이 어느 환경에서도 잘 도작하도록 설계

![polyglot](https://user-images.githubusercontent.com/76020494/108794206-b07fb300-75c8-11eb-9f97-9a4e1695588c.png)

폴리그랏 퍼시스턴스 요건을 만족하기 위해 기존 h2를 hsqldb로 변경

```
<!--		<dependency>-->
<!--			<groupId>com.h2database</groupId>-->
<!--			<artifactId>h2</artifactId>-->
<!--			<scope>runtime</scope>-->
<!--		</dependency>-->

		<dependency>
			<groupId>org.hsqldb</groupId>
			<artifactId>hsqldb</artifactId>
			<version>2.4.0</version>
			<scope>runtime</scope>
		</dependency>

# 변경/재기동 후 예약 주문
 http localhost:8081/orders orderType=basic name=woo
 
HTTP/1.1 201 
Content-Type: application/json;charset=UTF-8
Date: Mon, 21 Jun 2021 10:53:32 GMT
Location: http://localhost:8081/orders/2
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/2"
        },
        "self": {
            "href": "http://localhost:8081/orders/2"
        }
    },
    "cardNo": null,
    "name": "woo",
    "orderType": "basic",
    "status": null
}

# 저장이 잘 되었는지 조회

http localhost:8084/mypages/2

HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Mon, 21 Jun 2021 10:55:04 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "mypage": {
            "href": "http://localhost:8084/mypages/2"
        },
        "self": {
            "href": "http://localhost:8084/mypages/2"
        }
    },
    "name": "woo",
    "orderId": 2,
    "reservationId": 5,
    "status": "Reservation Complete"
}


```



## 동기식 호출 과 Fallback 처리

분석단계에서의 조건 중 하나로 주문(app)->결제(pay) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient 를 이용하여 호출하도록 한다. 

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 
```
# (external) PaymentHistoryService.java

package healthcenter.external;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@FeignClient(name="payment", url="${api.payment.url}")
public interface PaymentHistoryService {

    @RequestMapping(method= RequestMethod.POST, path="/paymentHistories")
    public void pay(@RequestBody PaymentHistory paymentHistory);

}                      
```

- 주문을 받은 직후(@PostPersist) 결제를 요청하도록 처리
```
# Order.java (Entity)
    @PostPersist
    public void onPostPersist(){
        Ordered ordered = new Ordered();
        BeanUtils.copyProperties(this, ordered);
        ordered.publishAfterCommit();


        healthcenter.external.PaymentHistory paymentHistory = new healthcenter.external.PaymentHistory();

        PaymentHistory payment = new PaymentHistory();
        System.out.println("this.id() : " + this.id);
        payment.setOrderId(this.id);
        payment.setStatus("Reservation OK");
        // mappings goes here
        OrderApplication.applicationContext.getBean(healthcenter.external.PaymentHistoryService.class)
            .pay(payment);
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 주문도 못받는다는 것을 확인:
```
# 결제 (payment) 서비스를 잠시 내려놓음 (ctrl+c)

#주문처리
 http localhost:8081/orders orderType=prime name=jung

#Fail
HTTP/1.1 500 
Connection: close
Content-Type: application/json;charset=UTF-8
Date: Mon, 21 Jun 2021 12:45:45 GMT
Transfer-Encoding: chunked

{
    "error": "Internal Server Error",
    "message": "Could not commit JPA transaction; nested exception is javax.persistence.RollbackException: Error while committing the transaction",
    "path": "/orders",
    "status": 500,
    "timestamp": "2021-06-21T12:45:45.797+0000"
}


#결제서비스 재기동
cd /home/project/healthcenter/payment
mvn spring-boot:run

#주문처리
http localhost:8081/orders orderType=prime name=jung

#Success
HTTP/1.1 201 
Content-Type: application/json;charset=UTF-8
Date: Mon, 21 Jun 2021 12:47:48 GMT
Location: http://localhost:8081/orders/2
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/2"
        },
        "self": {
            "href": "http://localhost:8081/orders/2"
        }
    },
    "cardNo": null,
    "name": "jung",
    "orderType": "prime",
    "status": null
}
```

## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

결제가 이루어진 후에 센터예약 시스템으로 이를 알려주는 행위는 동기식이 아니라 비 동기식으로 처리하여 
예약 시스템의 처리를 위하여 결제주문이 블로킹 되지 않도록 처리
 
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)
 
```
#PaymentHistory.java

package hotel;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="PaymentHistory_table")
public class PaymentHistory {

...
    @PostPersist
    public void onPostPersist(){
        PaymentApproved paymentApproved = new PaymentApproved();
        paymentApproved.setStatus("Pay Approved!!");
        BeanUtils.copyProperties(this, paymentApproved);
        paymentApproved.publishAfterCommit();
    }
```

- 예약서비스에서는 결제승인 이벤트에 대해서 이를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다.
- 카톡/이메일 등으로 호텔은 노티를 받고, 예약 상황을 확인 하고, 최종 예약 상태를 UI에 입력할테니, 우선 예약정보를 DB에 받아놓은 후, 이후 처리는 해당 Aggregate 내에서 하면 되겠다.

```
# (reservation) PolicyHandler.java

package healthcenter;

import hotel.config.kafka.KafkaProcessor;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.handler.annotation.Payload;
import org.springframework.stereotype.Service;

@Service
public class PolicyHandler{

    @Autowired
    private ReservationRepository reservationRepository;
	
    @StreamListener(KafkaProcessor.INPUT)
    public void onStringEventListener(@Payload String eventString){

    }

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaymentApproved_(@Payload PaymentApproved paymentApproved){


        if(paymentApproved.isMe()){
            System.out.println("##### listener  : " + paymentApproved.toJson());
            Reservation reservation = new Reservation();
            reservation.setStatus("Reservation Complete");
            reservation.setOrderId(paymentApproved.getOrderId());
            reservationRepository.save(reservation);
            
        }
    }

}


```

예약시스템은 주문/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 예약시스템이 유지보수로 인해 잠시 내려간 상태라도 예약 주문을 받는데 문제가 없어야 한다.

```
# (reservation)예약 서비스를 잠시 내려놓음 (ctrl+c)

# 주문처리
http localhost:8081/orders orderType=prime name=jung   #Success

# 결제처리
http localhost:8083/paymentHistories orderId=3 price=50000 payMethod=cash   #Success

# 주문 상태 확인
http localhost:8081/orders/3   

# 주문상태 안바뀜 확인
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Mon, 21 Jun 2021 12:57:53 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/3"
        },
        "self": {
            "href": "http://localhost:8081/orders/3"
        }
    },
    "cardNo": null,
    "name": "jung",
    "orderType": "prime",
    "status": null
}

# reservation 서비스 기동
cd /home/project/healthcenter/reservation
mvn spring-boot:run

# 주문상태 확인
http localhost:8081/orders/3 

# 주문 상태가 "Reservation Complete"으로 확인
HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Mon, 21 Jun 2021 13:01:14 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "mypage": {
            "href": "http://localhost:8084/mypages/5"
        },
        "self": {
            "href": "http://localhost:8084/mypages/5"
        }
    },
    "name": "jung",
    "orderId": 3,
    "reservationId": 2,
    "status": "Reservation Complete"
}
```

## API 게이트웨이(gateway)

API gateway 를 통해 MSA 진입점을 통일 시킨다.

```
# gateway 기동(8080 포트)
cd gateway
mvn spring-boot:run

# api gateway를 통한 3001 호텔 standard룸 예약 주문
http localhost:8080/orders orderType=prime name=jung

HTTP/1.1 201 
Content-Type: application/json;charset=UTF-8
Date: Mon, 21 Jun 2021 12:47:48 GMT
Location: http://localhost:8081/orders/2
Transfer-Encoding: chunked

{
    "_links": {
        "order": {
            "href": "http://localhost:8081/orders/2"
        },
        "self": {
            "href": "http://localhost:8081/orders/2"
        }
    },
    "cardNo": null,
    "name": "jung",
    "orderType": "prime",
    "status": null
}
```

```
application.yml

server:
  port: 8080

---
spring:
  profiles: default
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://localhost:8081
          predicates:
            - Path=/orders/** 
        - id: reservation
          uri: http://localhost:8082
          predicates:
            - Path=/reservations/**,/cancellations/** 
        - id: payment
          uri: http://localhost:8083
          predicates:
            - Path=/paymentHistories/** 
        - id: notification
          uri: http://localhost:8084
          predicates:
            - Path= /mypages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true


---

spring:
  profiles: docker
  cloud:
    gateway:
      routes:
        - id: order
          uri: http://order:8080
          predicates:
            - Path=/orders/** 
        - id: reservation
          uri: http://reservation:8080
          predicates:
            - Path=/reservations/**,/cancellations/** 
        - id: payment
          uri: http://payment:8080
          predicates:
            - Path=/paymentHistories/** 
        - id: customer
          uri: http://customer:8080
          predicates:
            - Path= /mypages/**
      globalcors:
        corsConfigurations:
          '[/**]':
            allowedOrigins:
              - "*"
            allowedMethods:
              - "*"
            allowedHeaders:
              - "*"
            allowCredentials: true
            
logging:
  level:
    root: debug

server:
  port: 8080

```
## 마이페이지
# CQRS
- 고객이 예약건에 대한 Status를 조회할 수 있도록 CQRS로 구현하였음.
-  mypage 조회를 통해 모든 예약건에 대한 상태정보를 확인할 수 있음.

고객의 예약정보를 한 눈에 볼 수 있게 mypage를 구현 한다.(CQRS)

```
# mypage 호출 
http localhost:8084/mypages/2

HTTP/1.1 200 
Content-Type: application/hal+json;charset=UTF-8
Date: Mon, 21 Jun 2021 10:56:50 GMT
Transfer-Encoding: chunked

{
    "_links": {
        "mypage": {
            "href": "http://localhost:8084/mypages/2"
        },
        "self": {
            "href": "http://localhost:8084/mypages/2"
        }
    },
    "name": "woo",
    "orderId": 2,
    "reservationId": 6,
    "status": "Reservation Complete"
}
```
- 여러개의 리스트 
```{
    "_links": {
        "mypage": {
            "href": "http://localhost:8084/mypages/5"
        },
        "self": {
            "href": "http://localhost:8084/mypages/5"
        }
    },
    "name": "jung",
    "orderId": 3,
    "reservationId": 2,
    "status": "Reservation Complete"
},
{
    "_links": {
        "mypage": {
            "href": "http://localhost:8084/mypages/2"
        },
        "self": {
            "href": "http://localhost:8084/mypages/2"
        }
    },
    "name": "woo",
    "orderId": 2,
    "reservationId": 6,
    "status": "Reservation Complete"
}
```
# 운영

## CI/CD 설정
- 환경변수 준비  
<details markdown="1">
<summary>환경변수 설정 접기/펼치기</summary>
AWS_ACCOUNT_ID KUBE URL : EKS -> 클러스터 -> 구성 "세부정보"의 "API 엔드포인트 URL" CodeBuild 와 EKS 연결

```
1. eks-admin-service-account.yaml 파일 생성하여 sa 생성
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
  
2. kubectl apply -f eks-admin-service-account.yaml
혹은, 바로 적용도 가능함
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: eks-admin
  namespace: kube-system
EOF

3. eks-admin-cluster-role-binding.yaml 파일 생성하여 롤바인딩
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
  
4. kubectl apply -f eks-admin-cluster-role-binding.yaml
혹은, 바로 적용도 가능함
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: eks-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: eks-admin
  namespace: kube-system
EOF
```

만들어진 eks-admin SA 의 토큰 가져오기
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep eks-admin | awk '{print $1}')
KUBE TOKEN 가져오기
:

Code build와 ECR 연결 정책 설정 : code build -> 빌드 프로젝트 생성
<img width="1029" src=https://user-images.githubusercontent.com/17754849/108522319-0e35a600-7310-11eb-8d63-f32cf0651e0a.png>
<img width="1029" src=https://user-images.githubusercontent.com/17754849/108524004-ed6e5000-7311-11eb-831d-e6fca77ab59e.png>
<img width="400" src=https://user-images.githubusercontent.com/17754849/108524571-843b0c80-7312-11eb-968a-9d14b182afb8.png>

그리고 다시 뒷 내용은 "3. CICD-Pipeline_AWS_v2" pdf 자료 39페이지부터 (이미지가 많은 관계로, buildspec.yml 작성하기)

환경 변수  
<img width="600" src=https://user-images.githubusercontent.com/17754849/108938749-fce3f500-7693-11eb-8fca-8090f2527dfa.png>
```
{ "Action": [
      "ecr:BatchCheckLayerAvailability",
      "ecr:CompleteLayerUpload",
      "ecr:GetAuthorizationToken",
      "ecr:InitiateLayerUpload",
      "ecr:PutImage",
      "ecr:UploadLayerPart"
    ],
    "Resource": "*",
    "Effect": "Allow"
}
```

Codebuild cache 적용 : CICD PDF p.45, S3 만들고 설정해야 함
buildspec.yml에 aws eks --region $AWS_DEFAULT_REGION update-kubeconfig --name $_EKS 이거 넣어줘야 하는데 권한 에러 날 경우

https://stackoverflow.com/questions/56011492/accessdeniedexception-creating-eks-cluster-user-is-not-authorized-to-perform 상세 내용은 buildspec.yml과 코드빌드의 환경변수 확인하면 됨
</details>

- CI/CD 적용 및 빌드 성공 결과  
<img width="700" src=https://user-images.githubusercontent.com/17754849/108810818-5219fb00-75ef-11eb-9fe4-9ae4e2a4e8d7.png>
- Buildspec.yml

```
version: 0.2

env: 
  variables:
    _PROJECT_NAME: "teamtwohotel2-order"
    _DIR_NAME: "order"
    _EKS: "teamtwohotel2"
    _NAMESPACE: "teamtwohotel"

phases:
  install:
    runtime-versions:
      java: openjdk8
      docker: 18
    commands:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - echo $_PROJECT_NAME
      - echo $AWS_ACCOUNT_ID
      - echo $AWS_DEFAULT_REGION
      - echo $CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo start command
      - $(aws ecr get-login --no-include-email --region $AWS_DEFAULT_REGION)
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - cd $_DIR_NAME && mvn package -Dmaven.test.skip=true
      - ls -al
      - pwd
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$IMAGE_TAG .
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$IMAGE_TAG
      - echo connect kubectl
      - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
      - kubectl config set-credentials admin --token="$KUBE_TOKEN"
      - kubectl config set-context default --cluster=k8s --user=admin
      - kubectl config use-context default
      - |
          cat <<EOF | kubectl apply -f -
          apiVersion: v1
          kind: Service
          metadata:
            name: $_DIR_NAME
            namespace: $_NAMESPACE
            labels:
              app: $_DIR_NAME
          spec:
            ports:
              - port: 8080
                targetPort: 8080
            selector:
              app: $_DIR_NAME
          EOF
      - |
          cat  <<EOF | kubectl apply -f -
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: $_DIR_NAME
            namespace: $_NAMESPACE
            labels:
              app: $_DIR_NAME
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: $_DIR_NAME
            template:
              metadata:
                labels:
                  app: $_DIR_NAME
              spec:
                containers:
                  - name: $_DIR_NAME
                    image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$IMAGE_TAG
                    imagePullPolicy: Always
                    ports:
                      - containerPort: 8080
                    readinessProbe:
                      httpGet:
                        path: /actuator/health
                        port: 8080
                      initialDelaySeconds: 10
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 10
                    livenessProbe:
                      httpGet:
                        path: /actuator/health
                        port: 8080
                      initialDelaySeconds: 120
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 5
          EOF
cache:
  paths:
    - '/root/.m2/**/*'
```

## 동기식 호출 / Circuit Breaker

* 서킷 브레이킹 프레임워크의 선택: Spring FeignClient + Hystrix 옵션을 사용하여 구현함

시나리오는 단말앱(app)-->결제(pay) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 결제 요청이 과도할 경우 CB 를 통하여 장애격리.

- Hystrix 를 설정:  요청처리 쓰레드에서 처리시간이 610 밀리가 넘어서기 시작하여 어느정도 유지되면 CB 회로가 닫히도록 (요청을 빠르게 실패처리, 차단) 설정
```
# app 서비스, application.yml

feign:
  hystrix:
    enabled: true

# To set thread isolation to SEMAPHORE
#hystrix:
#  command:
#    default:
#      execution:
#        isolation:
#          strategy: SEMAPHORE

hystrix:
  command:
    # 전역설정
    default:
      execution.isolation.thread.timeoutInMilliseconds: 610

```

- 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
```
# (pay) Payment.java (Entity)

    @PrePersist
    public void onPrePersist(){

        if("cancle".equals(payMethod)) {
            // 예시 푸드 딜리버리처럼 행위 필드를 하나 더 추가 하려다가 payMethod에 cancle 들어오면 취소 요청인 것으로 정의
            PayCanceled payCanceled = new PayCanceled();
            BeanUtils.copyProperties(this, payCanceled);
            payCanceled.publish();
        } else {
            PayApproved payApproved = new PayApproved();
            BeanUtils.copyProperties(this, payApproved);

            // 바로 이벤트를 보내버리면 주문정보가 커밋되기도 전에 예약 상태 변경 이벤트가 발송되어 주문테이블의 상태가 바뀌지 않을 수 있다.
            // TX 리스너는 커밋이 완료된 후에 이벤트를 발생하도록 만들어준다.
            TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronizationAdapter() {
                @Override
                public void beforeCommit(boolean readOnly) {
                    payApproved.publish();
                }
            });

            try { // 피호출 서비스(결제:pay) 의 임의 부하 처리 - 400 밀리에서 증감 220 밀리 정도 왔다갔다 하게
                Thread.currentThread().sleep((long) (400 + Math.random() * 220));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

        }
    }
```

* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:
- 동시사용자 100명
- 60초 동안 실시

```
siege -c100 -t60S -r10 --content-type "application/json" 'http://localhost:8081/orders POST {"hotelId": "4001", "roomType": "standard"}'

defaulting to time-based testing: 60 seconds

{	"transactions":			        1054,
	"availability":			       80.34,
	"elapsed_time":			       59.74,
	"data_transferred":		        0.30,
	"response_time":		        5.47,
	"transaction_rate":		       17.64,
	"throughput":			        0.00,
	"concurrency":			       96.58,
	"successful_transactions":	        1054,
	"failed_transactions":		         258,
	"longest_transaction":		        8.41,
	"shortest_transaction":		        0.44
}

```
- 80.34% 성공, 19.66% 실패

## 오토스케일 아웃
#### 사전 작업
1. metric server 설치 - kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
2. Resource Request/Limit 설정
![image](https://user-images.githubusercontent.com/17021291/108804593-09f3dc00-75e1-11eb-9505-6d2140b61d00.png)
3. HPA 설정 - kubectl autoscale deployment payment --cpu-percent=50 --min=1 --max=10 cpu-percent=50 -n teamtwohotel  

Pod 들의 요청 대비 평균 CPU 사용율 (여기서는 요청이 200 milli-cores이므로, 모든 Pod의 평균 CPU 사용율이 100 milli-cores(50%)를 넘게되면 HPA 발생)"

#### Siege 도구 활용한 부하(Stress) 주기
1. siege 설치 - kubectl create -f siege.yaml
2. siege 접속 - kubectl exec -it siege -- /bin/bash
![image](https://user-images.githubusercontent.com/17021291/108792500-c1c6c080-75c4-11eb-8d9b-718f7c030de3.png)

#### 부하에 따른 오토스케일 아웃 모니터링
![image](https://user-images.githubusercontent.com/17021291/108803415-f4c97e00-75dd-11eb-9fa0-7c01135c551d.png)


## 무정지 배포(Readiness Probe)
#### 무정지 배포 전 replica 3 scale up
![image](https://user-images.githubusercontent.com/17021291/108797620-f0e22f80-75ce-11eb-81db-de7a27574d03.png)

#### Readiness 설정
![image](https://user-images.githubusercontent.com/17021291/108806467-18dc8d80-75e5-11eb-822a-3c187cb7ffcc.png)

#### Rolling Update
kubectl set image deploy order order=새로운 이미지 버전
![image](https://user-images.githubusercontent.com/17021291/108797739-461e4100-75cf-11eb-96fc-959f48dc17c0.png)

#### siege로 무중단 확인
![image](https://user-images.githubusercontent.com/17021291/108806577-6f49cc00-75e5-11eb-99c8-8904c9995186.png)

## Self Healing(Liveness Probe)
- room deployment.yml 파일 수정 
```
콘테이너 실행 후 /tmp/healthy 파일을 만들고 
90초 후 삭제
livenessProbe에 'cat /tmp/healthy'으로 검증하도록 함
```
![deployment yml tmp healthy](https://user-images.githubusercontent.com/38099203/119318677-8ff0f300-bcb4-11eb-950a-e3c15feed325.PNG)

- kubectl describe pod room -n airbnb 실행으로 확인
```
컨테이너 실행 후 90초 동인은 정상이나 이후 /tmp/healthy 파일이 삭제되어 livenessProbe에서 실패를 리턴하게 됨
pod 정상 상태 일때 pod 진입하여 /tmp/healthy 파일 생성해주면 정상 상태 유지됨
```

![get pod tmp healthy](https://user-images.githubusercontent.com/38099203/119318781-a9923a80-bcb4-11eb-9783-65051ec0d6e8.PNG)
![touch tmp healthy](https://user-images.githubusercontent.com/38099203/119319050-f118c680-bcb4-11eb-8bca-aa135c1e067e.PNG)



## Configmap
- configmap 생성  
  > kubectl create configmap my-config --from-literal=key1=value1 --from-literal=key2=value2
- configmap 정보 가져오기  
  > kubectl get configmaps my-config -o yaml  

- 파일로부터 configmap 생성 (configmap.yml 생성)
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: customer1
data:
  TEXT1: Customer1_Company
  TEXT2: Welcomes You
  COMPANY: Customer1 Company Technology Pct. Ltd.
```
  > kubectl create -f configmap.yml
- ![configmap](https://user-images.githubusercontent.com/17754849/108792601-fd618a80-75c4-11eb-9386-3d8785979309.png)
- 출력하는 소스는 아래의 secret에서 함께 

## Secret
- 시크릿 생성
```
kubectl create secret generic my-password --from-literal=password=mysqlpassword --namespace teamtwohotel
```
  > ![secret](https://user-images.githubusercontent.com/17754849/108868200-4f43f800-7639-11eb-8915-1999a695b85b.png)
- 시크릿 확인
```
kubectl get secret my-password -o yaml
```
  > ![확인](https://user-images.githubusercontent.com/17754849/108868606-b8c40680-7639-11eb-8296-dfc2ad9cb4e0.png)
- 시크릿 buildspec.yml
  > ![소스](https://user-images.githubusercontent.com/17754849/108870840-dd20e280-763b-11eb-8e55-bfc9dc70e9e0.png)
- 시크릿 자바 출력
  > ![결과출력](https://user-images.githubusercontent.com/17754849/108871144-30933080-763c-11eb-8e76-453348bb7ec0.png)


# 참고

## 개발 환경 구성

1. 도커 설치
https://whitepaek.tistory.com/38
위에 가면 도커 관련 명령어들도 있음

2. 카프카 설치
```
https://dev-jj.tistory.com/entry/MAC-Kafka-%EB%A7%A5%EC%97%90-Kafka-%EC%84%A4%EC%B9%98-%ED%95%98%EA%B8%B0-Docker-homebrew-Apache
https://jdm.kr/blog/208
경로 이동 /Users/jinhyeonbak/intensive/kafka_2.12-2.3.0/bin
주키퍼 실행
./zookeeper-server-start.sh ../config/zookeeper.properties &
카프카 broker 실행
./kafka-server-start.sh ../config/server.properties ( 4 ~ 5 는 건너뛰어도 됨 )
카프카 topic 만들기
./kafka-topic.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic teamtwohotel
카프카 producer 실행
./kafka-console-poducer.sh --broker-list localhost:9092 --topic teamtwohotel
카프카 consumer 실행
./kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic teamtwohotel --from-beginning
카프카 토픽 삭제 ./kafka-topics.sh --zookeeper localhost:2181 --delete --topic DummyTopic
카프카 토픽 리스트 ./kafka-topics.sh --list --zookeeper localhost:2181
카프카가 비정상일 때 sudo lsof -i :2181 한뒤
kill -9 pid 하고 다시 띄워준다
```

3. httpie 설치

4. aws cli 설치
https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/install-cliv2-mac.html
aws configure 로 액세스 ID 등 입력

5. eksctl 설치
https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/getting-started-eksctl.html

6. IAM 생성
https://www.44bits.io/ko/post/publishing_and_managing_aws_user_access_key

7. eksctl 생성 ( 시간이 좀 걸림 )
클러스터 생성
eksctl create cluster --name admin-eks --version 1.17 --nodegroup-name standard-workers --node-type t3.medium --nodes 4 --nodes-min 1 --nodes-max 4

8. Local EKS 클러스터 토큰가져오기 ( CI/CD 할때 필요한건데, 앞에 설정해줘야 할 게 더 있으니 아래 쪽 CI/CD 다시 참고 )
aws eks --region ap-northeast-2 update-kubeconfig --name admin-eks

9. 아마존 컨테이너 레지스트리
아마존 > ecr (elastic container registry) > ecr 레파지터리 : ECR은 각 배포될 이미지 대상과 이름을 맞춰준다
aws ecr create-repository --repository-name admin-eks --region ap-northeast-2
aws ecr put-image-scanning-configuration --repository-name admin-eks --image-scanning-configuration scanOnPush=true --region ap-northeast-2

10. AWS 컨테이너 레지스트리 로그인
aws ecr get-login-password --region (Region-Code) | docker login --username AWS --password-stdin (Account-Id).dkr.ecr.(Region-Code).amazonaws.com

11. AWS 레지스트리에 도커 이미지 푸시하기 (이건 위에서 한 거랑 좀 겹치는듯)
aws ecr create-repository --repository-name (IMAGE_NAME) --region ap-northeast-2
docker push (Account-Id).dkr.ecr.ap-northeast-2.amazonaws.com/(IMAGE_NAME):latest
