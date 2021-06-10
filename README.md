# 숙소예약(AirBnB) + 신규 MSA 서비스(고객불만등록) 추가

본 예제는 MSA/DDD/Event Storming/EDA 를 포괄하는 분석/설계/구현/운영 전단계를 커버하도록 구성한 예제입니다.
이는 클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트들을 통과하기 위한 예시 답안을 포함합니다.
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW


# Table of contents

- [예제 - 숙소예약](#---)
  - [서비스 시나리오](#서비스-시나리오)
  - [체크포인트](#체크포인트)
  - [분석/설계](#분석설계)
  - [구현:](#구현-)
    - [DDD 의 적용](#ddd-의-적용)
    - [폴리글랏 퍼시스턴스](#폴리글랏-퍼시스턴스)
    - [폴리글랏 프로그래밍](#폴리글랏-프로그래밍)
    - [동기식 호출 과 Fallback 처리](#동기식-호출-과-Fallback-처리)
    - [비동기식 호출 과 Eventual Consistency](#비동기식-호출-과-Eventual-Consistency)
  - [운영](#운영)
    - [CI/CD 설정](#cicd설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
  - [신규 개발 조직의 추가](#신규-개발-조직의-추가)

# 서비스 시나리오

AirBnB 커버하기 + 신규 MSA 추가

[Team] 기능적 요구사항
1. 호스트가 임대할 숙소를 등록/수정/삭제한다.
2. 고객이 숙소를 선택하여 예약한다.
3. 예약과 동시에 결제가 진행된다.
4. 예약이 되면 예약 내역(Message)이 전달된다.
5. 고객이 예약을 취소할 수 있다.
6. 예약 사항이 취소될 경우 취소 내역(Message)이 전달된다.
7. 숙소에 후기(review)를 남길 수 있다.
8. 전체적인 숙소에 대한 정보 및 예약 상태 등을 한 화면에서 확인 할 수 있다.(viewpage)

**[개인] 기능적 요구사항
1. 고객이 불만을 등록한다.
2. 고객이 불만 내용을 수정/삭제 할 수 있다.
3. 불만등록 시, Host 에게 알림 메시지가 발송된다 (Pub/Sub)
4. 등록한 Complain 을 조회할 수 있는 페이지가 제공된다(CQRS)**

[Team] 비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 예약 건은 성립되지 않아야 한다.  (Sync 호출)
1. 장애격리
    1. 숙소 등록 및 메시지 전송 기능이 수행되지 않더라도 예약은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
    1. 예약 시스템이 과중되면 사용자를 잠시동안 받지 않고 잠시 후에 하도록 유도한다  Circuit breaker, fallback
1. 성능
    1. 모든 방에 대한 정보 및 예약 상태 등을 한번에 확인할 수 있어야 한다  (CQRS)
    1. 예약의 상태가 바뀔 때마다 메시지로 알림을 줄 수 있어야 한다  (Event driven)

**[개인] 비기능적 요구사항
1. 트랜잭션
  - 결재가 되지 않은 Complain 은 등록할 수 없다. (Sync)
2. 장애격리
  - Host 알림 등록기능이 수행되지 않더라도 Complain 은 364일 24시간 받을 수 있어야 한다. (Async-Even Driven, Eventual Consistency)
  - Complain 등록 부하 발생 시, 잠시동안 받지 않고 안정된 다음 받도록 한다. (Circuit Breadker, fallback)
3. 성능
  - Complain 등록 내용을 방/예약정보와 함께 한번에 확인 할 수 있다. (CQRS)
  - Complain 등록 시, Host 에게 메시지 알림이 간다. (Event Driven)**


# 체크포인트

- 분석 설계

  - 이벤트스토밍: 
    - 스티커 색상별 객체의 의미를 제대로 이해하여 헥사고날 아키텍처와의 연계 설계에 적절히 반영하고 있는가?
    - 각 도메인 이벤트가 의미있는 수준으로 정의되었는가?
    - 어그리게잇: Command와 Event 들을 ACID 트랜잭션 단위의 Aggregate 로 제대로 묶었는가?
    - 기능적 요구사항과 비기능적 요구사항을 누락 없이 반영하였는가?    

  - 서브 도메인, 바운디드 컨텍스트 분리
    - 팀별 KPI 와 관심사, 상이한 배포주기 등에 따른  Sub-domain 이나 Bounded Context 를 적절히 분리하였고 그 분리 기준의 합리성이 충분히 설명되는가?
      - 적어도 3개 이상 서비스 분리
    - 폴리글랏 설계: 각 마이크로 서비스들의 구현 목표와 기능 특성에 따른 각자의 기술 Stack 과 저장소 구조를 다양하게 채택하여 설계하였는가?
    - 서비스 시나리오 중 ACID 트랜잭션이 크리티컬한 Use 케이스에 대하여 무리하게 서비스가 과다하게 조밀히 분리되지 않았는가?
  - 컨텍스트 매핑 / 이벤트 드리븐 아키텍처 
    - 업무 중요성과  도메인간 서열을 구분할 수 있는가? (Core, Supporting, General Domain)
    - Request-Response 방식과 이벤트 드리븐 방식을 구분하여 설계할 수 있는가?
    - 장애격리: 서포팅 서비스를 제거 하여도 기존 서비스에 영향이 없도록 설계하였는가?
    - 신규 서비스를 추가 하였을때 기존 서비스의 데이터베이스에 영향이 없도록 설계(열려있는 아키택처)할 수 있는가?
    - 이벤트와 폴리시를 연결하기 위한 Correlation-key 연결을 제대로 설계하였는가?

  - 헥사고날 아키텍처
    - 설계 결과에 따른 헥사고날 아키텍처 다이어그램을 제대로 그렸는가?
    
- 구현
  - [DDD] 분석단계에서의 스티커별 색상과 헥사고날 아키텍처에 따라 구현체가 매핑되게 개발되었는가?
    - Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 데이터 접근 어댑터를 개발하였는가
    - [헥사고날 아키텍처] REST Inbound adaptor 이외에 gRPC 등의 Inbound Adaptor 를 추가함에 있어서 도메인 모델의 손상을 주지 않고 새로운 프로토콜에 기존 구현체를 적응시킬 수 있는가?
    - 분석단계에서의 유비쿼터스 랭귀지 (업무현장에서 쓰는 용어) 를 사용하여 소스코드가 서술되었는가?
  - Request-Response 방식의 서비스 중심 아키텍처 구현
    - 마이크로 서비스간 Request-Response 호출에 있어 대상 서비스를 어떠한 방식으로 찾아서 호출 하였는가? (Service Discovery, REST, FeignClient)
    - 서킷브레이커를 통하여  장애를 격리시킬 수 있는가?
  - 이벤트 드리븐 아키텍처의 구현
    - 카프카를 이용하여 PubSub 으로 하나 이상의 서비스가 연동되었는가?
    - Correlation-key:  각 이벤트 건 (메시지)가 어떠한 폴리시를 처리할때 어떤 건에 연결된 처리건인지를 구별하기 위한 Correlation-key 연결을 제대로 구현 하였는가?
    - Message Consumer 마이크로서비스가 장애상황에서 수신받지 못했던 기존 이벤트들을 다시 수신받아 처리하는가?
    - Scaling-out: Message Consumer 마이크로서비스의 Replica 를 추가했을때 중복없이 이벤트를 수신할 수 있는가
    - CQRS: Materialized View 를 구현하여, 타 마이크로서비스의 데이터 원본에 접근없이(Composite 서비스나 조인SQL 등 없이) 도 내 서비스의 화면 구성과 잦은 조회가 가능한가?

  - 폴리글랏 플로그래밍
    - 각 마이크로 서비스들이 하나이상의 각자의 기술 Stack 으로 구성되었는가?
    - 각 마이크로 서비스들이 각자의 저장소 구조를 자율적으로 채택하고 각자의 저장소 유형 (RDB, NoSQL, File System 등)을 선택하여 구현하였는가?
  - API 게이트웨이
    - API GW를 통하여 마이크로 서비스들의 집입점을 통일할 수 있는가?
    - 게이트웨이와 인증서버(OAuth), JWT 토큰 인증을 통하여 마이크로서비스들을 보호할 수 있는가?
- 운영
  - SLA 준수
    - 셀프힐링: Liveness Probe 를 통하여 어떠한 서비스의 health 상태가 지속적으로 저하됨에 따라 어떠한 임계치에서 pod 가 재생되는 것을 증명할 수 있는가?
    - 서킷브레이커, 레이트리밋 등을 통한 장애격리와 성능효율을 높힐 수 있는가?
    - 오토스케일러 (HPA) 를 설정하여 확장적 운영이 가능한가?
    - 모니터링, 앨럿팅: 
  - 무정지 운영 CI/CD (10)
    - Readiness Probe 의 설정과 Rolling update을 통하여 신규 버전이 완전히 서비스를 받을 수 있는 상태일때 신규버전의 서비스로 전환됨을 siege 등으로 증명 
    - Contract Test :  자동화된 경계 테스트를 통하여 구현 오류나 API 계약위반를 미리 차단 가능한가?


# 분석/설계

## AS-IS 조직 (Horizontally-Aligned)
  ![image](https://user-images.githubusercontent.com/77129832/119316165-96ca3680-bcb1-11eb-9a91-f2b627890bab.png)

## TO-BE 조직 (Vertically-Aligned)  
  ![image](https://user-images.githubusercontent.com/77129832/121279571-bbf4b100-c90f-11eb-90a2-15879dc3b5a5.png)


## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과:  http://www.msaez.io/#/storming/QtpQtDiH1Je3wad2QxZUJVvnLzO2/share/6f36e16efdf8c872da3855fedf7f3ea9


### 이벤트 도출
![image](https://user-images.githubusercontent.com/77129832/121282586-be0d3e80-c914-11eb-8f82-707c486332e7.png)


### 액터, 커맨드 부착 및 어그리게잇, 바운디드 컨텍스트 묶기
![image](https://user-images.githubusercontent.com/77129832/121282666-df6e2a80-c914-11eb-9f14-d6ad281d4d58.png)


   - 도메인 서열 분리 
     - Core Domain
       : reservation, room : 없어서는 안될 핵심 서비스이며, 연간 Up-time SLA 수준을 99.999% 목표, 배포주기는 reservation 의 경우 1주일 1회 미만, room 의 경우 1개월 1회 미만
     - Supporting Domain
       : message, viewpage, complain(*) : 경쟁력을 내기위한 서비스이며, SLA 수준은 연간 60% 이상 uptime 목표, 배포주기는 각 팀의 자율이나 표준 스프린트 주기가 1주일 이므로 
                                        1주일 1회 이상을 기준으로 함. (*) 금번 신규 추가한 Domain 영역 --> 고객불만 Complain(*)
      - General Domain
        : payment : 결제서비스로 3rd Party 외부 서비스를 사용하는 것이 경쟁력이 높음 

### 폴리시 부착 (괄호는 수행주체, 폴리시 부착을 둘째단계에서 해놔도 상관 없음. 전체 연계가 초기에 드러남)

![image](https://user-images.githubusercontent.com/77129832/121282907-396ef000-c915-11eb-96c8-00c90588c13a.png)


### 폴리시의 이동과 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp), 완성된 모델

![image](https://user-images.githubusercontent.com/77129832/121282943-4be92980-c915-11eb-9343-c9d881960c1e.png)
  : Complain 신규 MSA 추가, View 페이지 속성/기능 구현, Message 연동, Payment 동기식 체크 등


### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증

![image](https://user-images.githubusercontent.com/77129832/121283680-80111a00-c916-11eb-9b24-932ca52de617.png)


    - Team 기존 서비스와의 연계, 정상 서비스 여부 체크
      . 호스트가 임대할 숙소를 등록/수정/삭제한다.(ok)
      . 고객이 숙소를 선택하여 예약한다.(ok)
      . 예약과 동시에 결제가 진행된다.(ok)
      . 예약이 되면 예약 내역(Message)이 전달된다.(ok)
      . 고객이 예약을 취소할 수 있다.(ok)
      . 예약 사항이 취소될 경우 취소 내역(Message)이 전달된다.(ok)
      . 숙소에 후기(review)를 남길 수 있다.(ok)
      . 전체적인 숙소에 대한 정보 및 예약 상태 등을 한 화면에서 확인 할 수 있다.(View-green Sticker 추가로 ok)
    - 신규 추가 서비스 체크
      . 고객이 Complain 을 등록한다. (ok)
      . 고객이 Complain 내용을 수정/삭제 한다. (ok)
      . Complain 등록 시, Host 에게 알림 메시지가 발송된다 (ok)
      . 등록한 Complain 을 기존 Room 기준으로 통합 조회할 수 있다. (ok)
      . 결재된 건(paid) 만 Complain 등록할 수 있다. (ok)
    

### 비기능 요구사항에 대한 검증

![image](https://user-images.githubusercontent.com/77129832/121283966-04fc3380-c917-11eb-988c-d5ac5bbe2231.png)


- 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
- [Team] 고객 예약시 결제처리:  결제가 완료되지 않은 예약은 절대 받지 않는다고 결정하여, ACID 트랜잭션 적용. 예약 완료시 사전에 방 상태를 확인하는 것과 결제처리에 대해서는 Request-Response 방식 처리
- [개인] 결재 미 완료 고객에 대한 Complain 은 받지 않는 걸로 가정함. ACID 트랜잭션 저용. Complain 등록 시, 사전 결재완료 여부 확인하는 부분은 Request-Response 방식으로 처리함
- [Team] 결제 완료시 Host 연결 및 예약처리:  reservation 에서 room 마이크로서비스로 예약요청이 전달되는 과정에 있어서 room 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
- [개인] 결재 완료 시, Complain 등록 및 Message 발송 : Complain 에서 Message 서비스로 메시지 알림이 전달되는 과정에서 Message 서비스는 별도 배포 정책/주기를 가져가므로 Eventual Consistency 방식으로 트랜잭션 처리함
- 나머지 모든 inter-microservice 트랜잭션: 예약상태, 후기처리 등 모든 이벤트에 대해 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.


## 헥사고날 아키텍처 다이어그램 도출

![image](https://user-images.githubusercontent.com/77129832/121284506-ed717a80-c917-11eb-8942-232354f63c3f.png)


    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


# 구현:

분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)

```
   mvn spring-boot:run
```

## CQRS

숙소(Room) 의 사용가능 여부, 리뷰 및 예약/결재 등 총 Status 에 대하여 고객(Customer)이 조회 할 수 있도록 CQRS 로 구현하였다.
- room, review, reservation, payment, complain, message 개별 Aggregate Status 를 통합 조회하여 성능 Issue 를 사전에 예방할 수 있다.
- 비동기식으로 처리되어 발행된 이벤트 기반 Kafka 를 통해 수신/처리 되어 별도 Table 에 관리한다
- Table 모델링 (ROOMVIEW) (*) 기존 항목에 신규 MSA(Complain) 관리항목 추가함

  ![image](https://user-images.githubusercontent.com/77129832/121284667-33c6d980-c918-11eb-8e45-a53235ccc49e.png)  
  
- viewpage MSA ViewHandler 를 통해 구현 ("ComplainRegistered" 이벤트 발생 시, Pub/Sub 기반으로 별도 Roomview 테이블에 저장)

  ![image](https://user-images.githubusercontent.com/77129832/121284997-9d46e800-c918-11eb-9c7b-4260af04699b.png)

  ![image](https://user-images.githubusercontent.com/77129832/121284838-6b358600-c918-11eb-8793-f6949a418e8e.png)  
  
- ViewPage 조회 시, Room 기반 예약/결재 및 Complain 등 상태정보를 종합적으로 확인 할 수 있다.

  ![image](https://user-images.githubusercontent.com/77129832/121287480-94f0ac00-c91c-11eb-8323-e2250b57628e.png)



## 게이트웨이(Gateway)

      1. gateway 스프링부트 App을 추가 후 application.yaml내에 각 마이크로 서비스의 routes 를 추가하고 gateway 서버의 포트를 8080 으로 설정함
       
          - application.yaml 예시
            ```
            spring:
		  profiles: docker
		  cloud:
		    gateway:
		      routes:
			- id: payment
			  uri: http://payment:8080
			  predicates:
			    - Path=/payments/**, /paycheck/** 
			- id: room
			  uri: http://room:8080
			  predicates:
			    - Path=/rooms/**, /reviews/**, /check/**
			- id: reservation
			  uri: http://reservation:8080
			  predicates:
			    - Path=/reservations/**
			- id: message
			  uri: http://message:8080
			  predicates:
			    - Path=/messages/** 
			- id: viewpage
			  uri: http://viewpage:8080
			  predicates:
			    - Path= /roomviews/**
			- id: complain
			  uri: http://complain:8080
			  predicates:
			    - Path=/complains/**
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

		server:
		  port: 8080          
            ```
         
      2. Kubernetes용 Deployment.yaml 을 작성하고 Kubernetes에 Deploy를 생성함
          - Deployment.yaml 예시
          

            ```
            apiVersion: apps/v1
		kind: Deployment
		metadata:
		  name: gateway
		  namespace: airbnb
		  labels:
		    app: gateway
		spec:
		  replicas: 1
		  selector:
		    matchLabels:
		      app: gateway
		  template:
		    metadata:
		      labels:
			app: gateway
		    spec:
		      containers:
			- name: gateway
			  image: 654789606772.dkr.ecr.ap-northeast-2.amazonaws.com/user03-gateway:v1
			  ports:
			    - containerPort: 8080
            ```              
            ```
            Deploy 생성
            kubectl apply -f deployment.yaml
            ```     
          - Kubernetes에 생성된 Deploy. 확인
            
   ![image](https://user-images.githubusercontent.com/77129832/121293132-16990780-c926-11eb-9310-2f482959a196.png)

            
      3. Kubernetes용 Service.yaml을 작성하고 Kubernetes에 Service/LoadBalancer을 생성하여 Gateway 엔드포인트를 확인함. 
          - Service.yaml 예시
          
            ```
            apiVersion: v1
              kind: Service
              metadata:
                name: gateway
                namespace: airbnb
                labels:
                  app: gateway
              spec:
                ports:
                  - port: 8080
                    targetPort: 8080
                selector:
                  app: gateway
                type:
                  LoadBalancer           
            ```             

           
            ```
            Deploy 생성
            kubectl apply -f service.yaml            
            ```             
            
            
          - API Gateay 엔드포인트 확인
           
            ```
            Service 
            kubectl get svc -n airbnb           
            ```                 
![image](https://user-images.githubusercontent.com/77129832/121293180-29134100-c926-11eb-93ff-2bfae3acff1e.png)


# Correlation

Airbnb 프로젝트에서는 PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현을 
이벤트 클래스 안의 변수로 전달받아 서비스간 연관된 처리를 정확하게 구현하고 있습니다. 

아래의 구현 예제를 보면

Room 등록, Reservation 수행 및 결재(Paymen) 까지 완료되면 Complain 을 등록할 수 있으며,
Complain 등록 시, 각 서비스(기존 서비스 및 Complain, Message, ViewPage) 상태 값이 Update 되는 것을 확인할 수 있습니다.

Room 등록/확인
![image](https://user-images.githubusercontent.com/77129832/121293562-cb332900-c926-11eb-9b15-e799d9a43bfb.png)

Reservation
![image](https://user-images.githubusercontent.com/77129832/121293731-12211e80-c927-11eb-9eb2-6ca999a7edf3.png)

Payment
![image](https://user-images.githubusercontent.com/77129832/121293777-282edf00-c927-11eb-8bb7-ce00620c07fb.png)

Complain 등록/확인
![image](https://user-images.githubusercontent.com/77129832/121305144-3507fe80-c938-11eb-8d98-18347b6ff067.png)

Complain 등록 결과(CQRS 확인)
![image](https://user-images.githubusercontent.com/77129832/121305247-579a1780-c938-11eb-9422-7d5667efaa20.png)

등록된 Complain 메시지 발송
![image](https://user-images.githubusercontent.com/77129832/121305340-73052280-c938-11eb-9f51-d89c51d99bae.png)


## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. (Complain MSA). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 현실에서 발생가는한 이벤트에 의하여 마이크로 서비스들이 상호 작용하기 좋은 모델링으로 구현을 하였다.

```
package airbnb;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Complain_table")
public class Complain {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long cmpId;
    private Long roomId;
    private Long rsvId;
    private Long payId;
    private String contents;

    ... 
    
    public Long getCmpId() {
        return cmpId;
    }

    public void setCmpId(Long cmpId) {
        this.cmpId = cmpId;
    }
    public Long getRoomId() {
        return roomId;
    }

    public void setRoomId(Long roomId) {
        this.roomId = roomId;
    }
    public Long getRsvId() {
        return rsvId;
    }

    public void setRsvId(Long rsvId) {
        this.rsvId = rsvId;
    }

    public Long getPayId() {
        return payId;
    }

    public void setPayId(Long payId) {
        this.payId = payId;
    }

    public String getContents() {
        return contents;
    }

    public void setContents(String contents) {
        this.contents = contents;
    }

}

```
- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다
```
package airbnb;

import org.springframework.data.repository.PagingAndSortingRepository;
import org.springframework.data.rest.core.annotation.RepositoryRestResource;

@RepositoryRestResource(collectionResourceRel="complains", path="complains")
public interface ComplainRepository extends PagingAndSortingRepository<Complain, Long>{

}

```
- 적용 후 REST API 의 테스트
```
# Complain 서비스의 고객불만 등록
http POST localhost:8088/complains payId=1 roomId=1 contents="불만입니다"

```

## 폴리글랏 퍼시스턴스

기존 서비스 DB(H2)와는 다르게 Complain 서비스 DB는 MSSQL 기반으로 구현함

- pom.xml 설정

  ![image](https://user-images.githubusercontent.com/77129832/121313463-5f11ee80-c941-11eb-8a90-a92fa237f131.png)


- Application.yml 설정

  ![image](https://user-images.githubusercontent.com/77129832/121313407-515c6900-c941-11eb-85db-70c45925d15e.png)
 


- Complain 등록

  http POST ab11cd3a40ac94d4b9bc845d27bd6ae0-685141981.ap-northeast-2.elb.amazonaws.com:8080/complains payId=1 roomId=1 contents="불만입니다"

- DB Insert 결과

  ![image](https://user-images.githubusercontent.com/77129832/121313742-a6987a80-c941-11eb-966c-30e61ade7a4b.png)



## 동기식 호출(Sync) 과 Fallback 처리

Complain 등록 시, Payment 여부 확인 절차는 동기호출 구조로 처리함. 
FeignClient 를 이용하여 PaymentService 호출 후, 결과 여부에 따라 Complain Publish 구현함

- Complain, Payment 서비스 호출을 위해 FeignClient 기반 Service 대행 인터페이스 구현


```
# external/PaymentService.java

package airbnb.external;

import org.springframework.cloud.openfeign.FeignClient;
//import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(name="Payment", url="${prop.payment.url}")
public interface PaymentService {

    @RequestMapping(method= RequestMethod.GET, path="/paycheck/chkPayment")
    public boolean chkPayment(@RequestParam("payId") long payId);

}
```

- Complain 등록 요청 바은 지 후(@PostPersist) Paid 여부 확인 및 등록 절차를 Sync 방식으로 처리

```
# Complain.java (Entity)

   @PostPersist
    public void onPostPersist(){

        /////////////////////////////////////////////////////////////////
        // complain Insert (고객이 불만을 등록함)
        /////////////////////////////////////////////////////////////////
        System.out.println("### Param getPayId Paid 여부 확인 : " + this.getPayId());

        // Paid 된 건인지 체크        
        boolean result = ComplainApplication.applicationContext.getBean(airbnb.external.PaymentService.class)
                        .chkPayment(this.getPayId());
        System.out.println("####### Check Result : " + result);

        if(result) {

            // 이벤트 발행 --> ComplainRegistered (불만이 등록됨)
            ComplainRegistered complainRegistered = new ComplainRegistered();
            BeanUtils.copyProperties(this, complainRegistered);
            complainRegistered.publishAfterCommit();

        }
    }
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, Payment 장애 시 고객 Complain 을 받을수 없음


# Payment 서비스 Down
![image](https://user-images.githubusercontent.com/77129832/121306390-be6c0080-c939-11eb-8e97-ccd3e0feb4d9.png)


# Complain 등록
http POST localhost:8088/complains payId=1 roomId=1 contents="Too Dirty" --> 오류 발생

![image](https://user-images.githubusercontent.com/77129832/121306539-efe4cc00-c939-11eb-9767-c2fbffd13d01.png)


# Payment 서비스 Start
mvn spring-boot:run

# Complain 등록
http POST localhost:8088/complains payId=1 roomId=1 contents="Too Dirty" --> 성공
![image](https://user-images.githubusercontent.com/77129832/121307572-16efcd80-c93b-11eb-8f04-a9f6212b33c7.png)




- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. (서킷브레이커, 폴백 처리는 운영단계에서 설명한다.)



## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트


결재가 완료되어야 Complain 등록이 가능하고, Complain 등록 시 Message 알림 서비스 발송은 ASync 로 처리한다. 
- Complain 이 등록되면 complainRegistered 이벤트를 카프카로 송출한다. (Publish)

```
# Complain.java

package airbnb;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;

@Entity
@Table(name="Complain_table")
public class Complain {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long cmpId;
    private Long roomId;
    private Long rsvId;
    private Long payId;
    private String contents;

    @PostPersist
    public void onPostPersist(){

        /////////////////////////////////////////////////////////////////
        // complain Insert (고객이 불만을 등록함)
        /////////////////////////////////////////////////////////////////

        System.out.println("### Param getPayId Paid 여부 확인 : " + this.getPayId());

        // Paid 된 건인지 체크
        
        boolean result = ComplainApplication.applicationContext.getBean(airbnb.external.PaymentService.class)
                        .chkPayment(this.getPayId());
        System.out.println("####### Check Result : " + result);

        if(result) {

            // 이벤트 발행 --> ComplainRegistered (불만이 등록됨)
            ComplainRegistered complainRegistered = new ComplainRegistered();
            BeanUtils.copyProperties(this, complainRegistered);
            complainRegistered.publishAfterCommit();
        }       
    }
    ...
}

```

- Message 서비스에서는 complainRegistered 이벤트를 수신하여 발송 처리하도록 PolicyHandler 를 구현한다.

```
# Message.java

package airbnb;

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverComplainRegistered_SendComplainMsg(@Payload ComplainRegistered complainRegistered){

        if(complainRegistered.isMe()){

            ///////////////////////////
            // Complain 등록 시
            ///////////////////////////
            System.out.println("\n\n##### listener SendComplainMsg : " + complainRegistered.toJson() + "\n\n");

            // Complain ID 추출
            long cmpId = complainRegistered.getCmpId(); // 등록된 Complain ID
            long roomId = complainRegistered.getRoomId(); // Complain 제기된 Room ID
            String msgString = "Complain 이 등록 되었습니다. Room ID : [" + roomId + ", Complain ID : [" + cmpId +"]";

            // Message 전송
            sendMsg(roomId, msgString);
        }            
    } 

```

# 운영


## CI/CD 설정

서비스 단위 개별 Repository 구성 기반으로 AWS CodeBuild 기반 CI/CD Pipeline 을 구현함. (buildspec.yml 기반)

- buildspec.yml 파일 생성
  (*) runtime 시 설치되는 java 를 'corretto8' 로 설정해야 함 (AWS CodeBuild 환경 이미지 유형 3.0 버전이 6/8 기준 없어짐)
```
version: 0.2

env:
  variables:
    _PROJECT_NAME: "user03-complain"
    _SERVICE_NAME: "complain"

phases:
  install:
    runtime-versions:
      java: corretto8
      docker: 18
    commands:
      - echo install kubectl
      - curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin/kubectl
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
      - mvn package -Dmaven.test.skip=true
      - docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:v2 .
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:v2
      - echo connect kubectl
      - kubectl config set-cluster k8s --server="$KUBE_URL" --insecure-skip-tls-verify=true
      - kubectl config set-credentials admin --token="$KUBE_TOKEN"
      - kubectl config set-context default --cluster=k8s --user=admin
      - kubectl config use-context default
      - kubectl apply -f kubernetes/deployment.yml
      - kubectl apply -f kubernetes/service.yml      
cache:
  paths:
    - '/root/.m2/**/*'
```

- AWS CodeBuild 빌드 프로젝트 생성 후, 빌드 세부정보를 세팅한다.

프로젝트 구성 (소스를 github 로 설정함)
(*) 각 MSA 별로 개별 Repository 생성

![image](https://user-images.githubusercontent.com/77129832/121316161-ff691280-c943-11eb-97de-01247290411b.png)


환경변수 세팅 (AWS_ACCOUNT_ID, KUBE_URL, KUBE_TOKEN)

![image](https://user-images.githubusercontent.com/77129832/121316294-21fb2b80-c944-11eb-8567-9a33bc467cc2.png)




- CodeBuild 프로젝트를 생성하고 AWS_ACCOUNT_ID, KUBE_URL, KUBE_TOKEN 환경 변수 세팅을 한다
```
SA 생성
kubectl apply -f eks-admin-service-account.yml
```
```
Role 생성
kubectl apply -f eks-admin-cluster-role-binding.yml
```
```
Token 확인
kubectl -n kube-system get secret
kubectl -n kube-system describe secret eks-admin
```
![image](https://user-images.githubusercontent.com/77129832/121316795-a51c8180-c944-11eb-9e38-a21ddd524071.png)

```
- codebuild 실행
```
전체 프로젝트 빌드내역

![image](https://user-images.githubusercontent.com/77129832/121316992-d301c600-c944-11eb-8f27-6549630b6c07.png)

- CodeBuild 내역 (Complain 서비스)

![image](https://user-images.githubusercontent.com/77129832/121317093-ee6cd100-c944-11eb-9216-ad753e30eedb.png)


## 오토스케일 아웃

사용자 요청을 100% 처리하기 위하여 자동화된 확장 기능을 적용하고자 함.

- Metric 서버 설치
  ```
  kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.3.7/components.yaml
  ```
- complain 서비스 deployment.yml 내 resource 설정
  ```
      spec:
      containers:
        - name: complain
          image: 654789606772.dkr.ecr.ap-northeast-2.amazonaws.com/user03-complain:v3
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "1000m"
              memory: "256Mi"
            limits:
              cpu: "2500m"  
              memory: "512Mi"	
  ```
- Complain 서비스 replica 를 동적으로 늘리기 위해 HPA(horizontalpodautoscaler) 를 설정한다
  (CPU 사용량 1% 넘으면 10개까지 늘려줌)
  ```
  kubectl autoscale deployment complain -n airbnb --cpu-percent=1 --min=1 --max=10
  ```
 
- siege 를 활용하여 부하 발생(100명, 1분)
  ```
  siege -c100 -t60S -v --content-type "application/json" 'http://ab11cd3a40ac94d4b9bc845d27bd6ae0-685141981.ap-northeast-2.elb.amazonaws.com:8080/complains POST {"payId":"1", "roomId":"1", "contents":"불만입니다"}'
  ```

- complain 서비스 모니터링
  ```
  kubectl get deploy complain -w -n airbnb
  ```

- 스케일아웃 확인
  ![image](https://user-images.githubusercontent.com/77129832/121443374-5b757a80-c9c8-11eb-8e6c-221fe863d366.png)  
  ![image](https://user-images.githubusercontent.com/77129832/121443756-1271f600-c9c9-11eb-91fa-dc53fa5afb97.png)


## 무정지 재배포

- 무정지 배포 확인을 위해 기존에 설정한 오토스케일러 설정을 제거함
  ```
  kubectl delete hpa complain -n airbnb
  ```
  ![image](https://user-images.githubusercontent.com/77129832/121445763-11db5e80-c9cd-11eb-9392-3398e9a87000.png)

- 수정된 내용 배포하고, CodeBuild 활용하여 New 버전(v4)으로 빌드한다
  (complain 서비스 deployment.yml 파일 내 readiness probe설정 없는 상태)
  
  deployment.yml
  ```
  apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: complain
	  namespace: airbnb
	  labels:
	    app: complain
	spec:
	  replicas: 1
	  selector:
	    matchLabels:
	      app: complain
	  template:
	    metadata:
	      labels:
		app: complain
	    spec:
	      containers:
		- name: complain
		  image: 654789606772.dkr.ecr.ap-northeast-2.amazonaws.com/user03-complain:v4
		  ports:
		    - containerPort: 8080
		  resources:
		    requests:
		      cpu: "1000m"
		      memory: "256Mi"
		    limits:
		      cpu: "2500m"  
		      memory: "512Mi"
  ```
  
  CodeBuild 배포 수행
  
  ![image](https://user-images.githubusercontent.com/77129832/121446955-9dee8580-c9cf-11eb-8428-ef2228f5436f.png)

- Siege 부하 발생, 500 Error 확인
 
  ![image](https://user-images.githubusercontent.com/77129832/121447003-bbbbea80-c9cf-11eb-80ff-6d22a569c399.png)

  Availability 떨어짐 확인
  
  ![image](https://user-images.githubusercontent.com/77129832/121447036-ce362400-c9cf-11eb-8e76-9a43d63d4f4c.png)

- 이에 대한 조치를 위해 Complain 서비스 Deployment.yml 파일 내 readiness probe 설정
  ```
      ...
      spec:
      containers:
        - name: complain
          image: 654789606772.dkr.ecr.ap-northeast-2.amazonaws.com/user03-complain:v4
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "1000m"
              memory: "256Mi"
            limits:
              cpu: "2500m"  
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: '/actuator/health'
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 2
            periodSeconds: 5
            failureThreshold: 10
  ```

- CodeBuild 활용 재 배포 후, siege 부하 결과 확인 --> Availability 100% 확인

  ![image](https://user-images.githubusercontent.com/77129832/121447505-bc08b580-c9d0-11eb-9fad-a5174af72a78.png)


# Self-healing (Liveness Probe)

- complain 서비스 deployment.yml 파일 수정 후, 빌드실행(CodeBuild)
   ```
   Container 실행 후, /tmp/healthy 파일 생성 90초 후, 삭제
   livenessProbe에 'cat /tmp/healthy' 검증함
   ```
   ![image](https://user-images.githubusercontent.com/77129832/121449360-9d0c2280-c9d4-11eb-83d7-a9998e987125.png)

- Container 실행 이 후, /tmp/healthy 파일 삭제되고, LivenessProbe 실패 리턴함.
  Pod 진입하여 /tmp/healthy 파일 생성하고, 정상 상태로 변경됨을 확인함
  
  kubectl describe pod complain -n airbnb 확인
  
  ![image](https://user-images.githubusercontent.com/77129832/121451697-fece8b80-c9d8-11eb-9c19-1c14432e1aaf.png)

  complain 해당 pod 상에 tmp/healthy 파일 생성
  ```
  kubectl exec -it pod/complain-77867b4c46-586tv -n airbnb -- touch /tmp/healthy
  ```
  
- 파일 생성 후, Running 상태 확인
  
  ![image](https://user-images.githubusercontent.com/77129832/121451873-540a9d00-c9d9-11eb-96c4-34f71288a9c4.png)



## 동기식 호출 / 서킷 브레이킹 / 장애격리

* 서킷 브레이킹 프레임워크의 선택: istio 사용

 Complain 등록 시, 사전 Paid 여부를 Pament 서비 Req/Res Sync 연동으로 구현되어 있으며, 
Complain 요청 부하가 과도하게 발생 시 서킷 브레이커를 통해 격리조치 함

- istio 및 kiali 설치

  ![image](https://user-images.githubusercontent.com/77129832/121453802-b44f0e00-c9dc-11eb-99d6-da30d5888da0.png)

- destination-rule 생성
  ```
  cat <<EOF | kubectl apply -f -
	apiVersion: networking.istio.io/v1alpha3
	kind: DestinationRule
	metadata:
	  name: dr-complain
	  namespace: airbnb
	spec:
	  host: complain
	  trafficPolicy:
	    connectionPool:
	      http:
		http1MaxPendingRequests: 1
		maxRequestsPerConnection: 1
	    outlierDetection:
	      consecutiveErrors: 5          # 5xx 에러가 5번 발생하면
	      interval: 1s                  # 1초마다 스캔 하여
	      baseEjectionTime: 30s         # 30 초 동안 circuit breaking 처리   
	      maxEjectionPercent: 100       # 100% 로 차단
	EOF
  ```





시나리오는 예약(reservation)--> 룸(room) 시의 연결을 RESTful Request/Response 로 연동하여 구현이 되어있고, 예약 요청이 과도할 경우 CB 를 통하여 장애격리.

- DestinationRule 를 생성하여 circuit break 가 발생할 수 있도록 설정
최소 connection pool 설정
```
# destination-rule.yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dr-room
  namespace: airbnb
spec:
  host: room
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 1
        maxRequestsPerConnection: 1
#    outlierDetection:
#      interval: 1s
#      consecutiveErrors: 1
#      baseEjectionTime: 10s
#      maxEjectionPercent: 100
```

* istio-injection 활성화 및 room pod container 확인

```
kubectl get ns -L istio-injection
kubectl label namespace airbnb istio-injection=enabled 
```

![Circuit Breaker(istio-enjection)](https://user-images.githubusercontent.com/38099203/119295450-d6812600-bc91-11eb-8aad-46eeac968a41.PNG)

![Circuit Breaker(pod)](https://user-images.githubusercontent.com/38099203/119295568-0cbea580-bc92-11eb-9d2b-8580f3576b47.PNG)


* 부하테스터 siege 툴을 통한 서킷 브레이커 동작 확인:

siege 실행

```
kubectl run siege --image=apexacme/siege-nginx -n airbnb
kubectl exec -it siege -c siege -n airbnb -- /bin/bash
```


- 동시사용자 1로 부하 생성 시 모두 정상
```
siege -c1 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

** SIEGE 4.0.4
** Preparing 1 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.49 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.05 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     254 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     256 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     256 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     256 bytes ==> POST http://room:8080/rooms
```

- 동시사용자 2로 부하 생성 시 503 에러 168개 발생
```
siege -c2 -t10S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'

** SIEGE 4.0.4
** Preparing 2 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.10 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.04 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.05 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.22 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.08 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.07 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.01 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.01 secs:      81 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     258 bytes ==> POST http://room:8080/rooms
HTTP/1.1 503     0.00 secs:      81 bytes ==> POST http://room:8080/rooms

Lifting the server siege...
Transactions:                   1904 hits
Availability:                  91.89 %
Elapsed time:                   9.89 secs
Data transferred:               0.48 MB
Response time:                  0.01 secs
Transaction rate:             192.52 trans/sec
Throughput:                     0.05 MB/sec
Concurrency:                    1.98
Successful transactions:        1904
Failed transactions:             168
Longest transaction:            0.03
Shortest transaction:           0.00
```

- kiali 화면에 서킷 브레이크 확인

![Circuit Breaker(kiali)](https://user-images.githubusercontent.com/38099203/119298194-7f7e4f80-bc97-11eb-8447-678eece29e5c.PNG)


- 다시 최소 Connection pool로 부하 다시 정상 확인

```
** SIEGE 4.0.4
** Preparing 1 concurrent users for battle.
The server is now under siege...
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.03 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.02 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.00 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms
HTTP/1.1 201     0.01 secs:     260 bytes ==> POST http://room:8080/rooms

:
:

Lifting the server siege...
Transactions:                   1139 hits
Availability:                 100.00 %
Elapsed time:                   9.19 secs
Data transferred:               0.28 MB
Response time:                  0.01 secs
Transaction rate:             123.94 trans/sec
Throughput:                     0.03 MB/sec
Concurrency:                    0.98
Successful transactions:        1139
Failed transactions:               0
Longest transaction:            0.04
Shortest transaction:           0.00

```

- 운영시스템은 죽지 않고 지속적으로 CB 에 의하여 적절히 회로가 열림과 닫힘이 벌어지면서 자원을 보호하고 있음을 보여줌.
  virtualhost 설정과 동적 Scale out (replica의 자동적 추가,HPA) 을 통하여 시스템을 확장 해주는 후속처리가 필요.


### 오토스케일 아웃
앞서 CB 는 시스템을 안정되게 운영할 수 있게 해줬지만 사용자의 요청을 100% 받아들여주지 못했기 때문에 이에 대한 보완책으로 자동화된 확장 기능을 적용하고자 한다. 

- room deployment.yml 파일에 resources 설정을 추가한다
![Autoscale (HPA)](https://user-images.githubusercontent.com/38099203/119283787-0a038680-bc79-11eb-8d9b-d8aed8847fef.PNG)

- room 서비스에 대한 replica 를 동적으로 늘려주도록 HPA 를 설정한다. 설정은 CPU 사용량이 50프로를 넘어서면 replica 를 10개까지 늘려준다:
```
kubectl autoscale deployment room -n airbnb --cpu-percent=50 --min=1 --max=10
```
![Autoscale (HPA)(kubectl autoscale 명령어)](https://user-images.githubusercontent.com/38099203/119299474-ec92e480-bc99-11eb-9bc3-8c5246b02783.PNG)

- 부하를 동시사용자 100명, 1분 동안 걸어준다.
```
siege -c100 -t60S -v --content-type "application/json" 'http://room:8080/rooms POST {"desc": "Beautiful House3"}'
```
- 오토스케일이 어떻게 되고 있는지 모니터링을 걸어둔다
```
kubectl get deploy room -w -n airbnb 
```
- 어느정도 시간이 흐른 후 (약 30초) 스케일 아웃이 벌어지는 것을 확인할 수 있다:
![Autoscale (HPA)(모니터링)](https://user-images.githubusercontent.com/38099203/119299704-6a56f000-bc9a-11eb-9ba8-55e5978f3739.PNG)

- siege 의 로그를 보아도 전체적인 성공률이 높아진 것을 확인 할 수 있다. 
```
Lifting the server siege...
Transactions:                  15615 hits
Availability:                 100.00 %
Elapsed time:                  59.44 secs
Data transferred:               3.90 MB
Response time:                  0.32 secs
Transaction rate:             262.70 trans/sec
Throughput:                     0.07 MB/sec
Concurrency:                   85.04
Successful transactions:       15675
Failed transactions:               0
Longest transaction:            2.55
Shortest transaction:           0.01
```


# Config Map/ Persistence Volume
- Persistence Volume

1: EFS 생성
```
EFS 생성 시 클러스터의 VPC를 선택해야함
```
![클러스터의 VPC를 선택해야함](https://user-images.githubusercontent.com/38099203/119364089-85048580-bce9-11eb-8001-1c20a93b8e36.PNG)

![EFS생성](https://user-images.githubusercontent.com/38099203/119343415-60041880-bcd1-11eb-9c25-1695c858f6aa.PNG)

2. EFS 계정 생성 및 ROLE 바인딩
```
kubectl apply -f efs-sa.yml

apiVersion: v1
kind: ServiceAccount
metadata:
  name: efs-provisioner
  namespace: airbnb


kubectl get ServiceAccount efs-provisioner -n airbnb
NAME              SECRETS   AGE
efs-provisioner   1         9m1s  
  
  
  
kubectl apply -f efs-rbac.yaml

namespace를 반듯이 수정해야함

  
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: efs-provisioner-runner
  namespace: airbnb
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-efs-provisioner
  namespace: airbnb
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
     # replace with namespace where provisioner is deployed
    namespace: airbnb
roleRef:
  kind: ClusterRole
  name: efs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
  namespace: airbnb
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-efs-provisioner
  namespace: airbnb
subjects:
  - kind: ServiceAccount
    name: efs-provisioner
    # replace with namespace where provisioner is deployed
    namespace: airbnb
roleRef:
  kind: Role
  name: leader-locking-efs-provisioner
  apiGroup: rbac.authorization.k8s.io


```

3. EFS Provisioner 배포
```
kubectl apply -f efs-provisioner-deploy.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: efs-provisioner
  namespace: airbnb
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: efs-provisioner
  template:
    metadata:
      labels:
        app: efs-provisioner
    spec:
      serviceAccount: efs-provisioner
      containers:
        - name: efs-provisioner
          image: quay.io/external_storage/efs-provisioner:latest
          env:
            - name: FILE_SYSTEM_ID
              value: fs-562f9c36
            - name: AWS_REGION
              value: ap-northeast-2
            - name: PROVISIONER_NAME
              value: my-aws.com/aws-efs
          volumeMounts:
            - name: pv-volume
              mountPath: /persistentvolumes
      volumes:
        - name: pv-volume
          nfs:
            server: fs-562f9c36.efs.ap-northeast-2.amazonaws.com
            path: /


kubectl get Deployment efs-provisioner -n airbnb
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
efs-provisioner   1/1     1            1           11m

```

4. 설치한 Provisioner를 storageclass에 등록
```
kubectl apply -f efs-storageclass.yml


kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: aws-efs
  namespace: airbnb
provisioner: my-aws.com/aws-efs


kubectl get sc aws-efs -n airbnb
NAME            PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
aws-efs         my-aws.com/aws-efs      Delete          Immediate              false                  4s
```

5. PVC(PersistentVolumeClaim) 생성
```
kubectl apply -f volume-pvc.yml


apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: aws-efs
  namespace: airbnb
  labels:
    app: test-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 6Ki
  storageClassName: aws-efs
  
  
kubectl get pvc aws-efs -n airbnb
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
aws-efs   Bound    pvc-43f6fe12-b9f3-400c-ba20-b357c1639f00   6Ki        RWX            aws-efs        4m44s
```

6. room pod 적용
```
kubectl apply -f deployment.yml
```
![pod with pvc](https://user-images.githubusercontent.com/38099203/119349966-bd9c6300-bcd9-11eb-9f6d-08e4a3ec82f0.PNG)


7. A pod에서 마운트된 경로에 파일을 생성하고 B pod에서 파일을 확인함
```
NAME                              READY   STATUS    RESTARTS   AGE
efs-provisioner-f4f7b5d64-lt7rz   1/1     Running   0          14m
room-5df66d6674-n6b7n             1/1     Running   0          109s
room-5df66d6674-pl25l             1/1     Running   0          109s
siege                             1/1     Running   0          2d1h


kubectl exec -it pod/room-5df66d6674-n6b7n room -n airbnb -- /bin/sh
/ # cd /mnt/aws
/mnt/aws # touch intensive_course_work
```
![a pod에서 파일생성](https://user-images.githubusercontent.com/38099203/119372712-9736f180-bcf2-11eb-8e57-1d6e3f4273a5.PNG)

```
kubectl exec -it pod/room-5df66d6674-pl25l room -n airbnb -- /bin/sh
/ # cd /mnt/aws
/mnt/aws # ls -al
total 8
drwxrws--x    2 root     2000          6144 May 24 15:44 .
drwxr-xr-x    1 root     root            17 May 24 15:42 ..
-rw-r--r--    1 root     2000             0 May 24 15:44 intensive_course_work
```
![b pod에서 파일생성 확인](https://user-images.githubusercontent.com/38099203/119373196-204e2880-bcf3-11eb-88f0-a1e91a89088a.PNG)


- Config Map

1: cofingmap.yml 파일 생성
```
kubectl apply -f cofingmap.yml


apiVersion: v1
kind: ConfigMap
metadata:
  name: airbnb-config
  namespace: airbnb
data:
  # 단일 key-value
  max_reservation_per_person: "10"
  ui_properties_file_name: "user-interface.properties"

  # 다수의 key-value
  room.properties: |
    room.types=hotel,pansion,guesthouse
    room.maximum-count=5    
  kakao-interface.properties: |
    kakao.font.color=blud
    kakao.color.bad=yellow
    kakao.textmode=true
```

2. deployment.yml에 적용하기

```
kubectl apply -f deployment.yml


.......
          env:
			# cofingmap에 있는 단일 key-value
            - name: MAX_RESERVATION_PER_PERSION
              valueFrom:
                configMapKeyRef:
                  name: airbnb-config
                  key: max_reservation_per_person
           - name: UI_PROPERTIES_FILE_NAME
              valueFrom:
                configMapKeyRef:
                  name: airbnb-config
                  key: ui_properties_file_name
          volumeMounts:
          - mountPath: "/mnt/aws"
            name: volume
      volumes:
        - name: volume
          persistentVolumeClaim:
            claimName: aws-efs
        - name: config
          configMap:
            # cofingmap에 있는 다수의 key-value
            name: game-demo
              items:
                - key: "room.properties"
                  path: "room.properties"
                - key: "kakao-interface.properties"
                  path: "kakao-interface.properties"
```

