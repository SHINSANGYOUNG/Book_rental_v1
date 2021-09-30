![image](https://user-images.githubusercontent.com/88864503/134761660-326438fa-dd98-4531-94d1-652ef508119e.png)

# 도서 대여

클라우드 네이티브 애플리케이션의 개발에 요구되는 체크포인트 확인
- 체크포인트 : https://workflowy.com/s/assessment-check-po/T5YrzcMewfo4J6LW

# Table of contents

- [도서대여](#---)
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
    - [ConfigMap 설정](#ConfigMap-설정)
    - [동기식 호출 / 서킷 브레이킹 / 장애격리](#동기식-호출-서킷-브레이킹-장애격리)
    - [오토스케일 아웃](#오토스케일-아웃)
    - [무정지 재배포](#무정지-재배포)
    - [Self healing](#Liveness-Probe)

# 서비스 시나리오

도서 대여 서비스

기능적 요구사항
1. 사용자가 도서를 예약한다.
2. 도서 예약 시 결제가 완료되어야 한다.
3. 사용자가 예약 중인 도서를 대여한다.
4. 사용자가 대여 중인 도서를 반납한다.
5. 사용자가 예약을 취소할 수 있다.
6. 도서 예약 취소 시에는 결제가 취소된다.
7. 사용자가 예약/대여 상태를 확인할 수 있다.

비기능적 요구사항
1. 트랜잭션
    1. 결제가 되지 않은 경우 예약할 수 없다 (Sync 호출) 
2. 장애격리
    1. 도서관리 기능이 수행되지 않더라도 대여/예약은 365일 24시간 받을 수 있어야 한다 Async (event-driven), Eventual Consistency
    1. 예약 요청 승인이 과중되면 고객을 잠시동안 받지 않고 예약 요청을 잠시후에 하도록 유도한다  Circuit breaker, fallback
3. 성능
    1. 사용자는 MyPage에서 본인 예약 및 대여 도서의 목록과 상태를 확인할 수 있어야한다 CQRS

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
![asis](https://user-images.githubusercontent.com/90441340/132832765-2ee6cd26-2841-43cd-b9ab-666664ee2de1.jpg)

## TO-BE 조직 (Vertically-Aligned)
![image](https://user-images.githubusercontent.com/88864503/134762117-ad908987-e5d5-4d27-abb6-6f88bd3280ff.png)

## Event Storming 결과
* MSAEz 로 모델링한 이벤트스토밍 결과 : https://www.msaez.io/#/storming/WwXHvrlj3pPbpwdkvR5NPhNGAj73/80b9a838bbd773a5578ac333ec7b1f2c

### 이벤트 도출
![image](https://user-images.githubusercontent.com/88864503/134762511-2285f6ed-34bf-4eb7-825f-0d2c7d0a7f7f.png)

### 부적격 이벤트 탈락
![image](https://user-images.githubusercontent.com/88864503/134762525-890972fb-b25f-4ffe-8124-abf9b3b8697c.png)

    - 과정중 도출된 잘못된 도메인 이벤트들을 걸러내는 작업을 수행함
        - 예약 시 > CustomerAuthenticatied : 고객인증이 완료되어야 예약 이벤트가 발생하는 ACID 트랜잭션을 적용이 필요하므로 Reserved이벤트와 통합하여 처리

### 액터, 커맨드 부착 및 어그리게잇으로 묶기
![image](https://user-images.githubusercontent.com/88864503/134762872-f8e70ae8-d124-4aa0-88f2-59d10dfbb01d.png)

- Customer의 Rental, Payment의 Payment관리, Book의 status관리는 그와 연결된 command 와 event 들에 의하여 트랜잭션이 유지되어야 하는 단위로 그들 끼리 묶어줌

### 바운디드 컨텍스트로 묶기
![image](https://user-images.githubusercontent.com/88864503/134762577-257c8502-825c-487f-8f99-7145d180fa62.png)

### 컨텍스트 매핑 (점선은 Pub/Sub, 실선은 Req/Resp)
![image](https://user-images.githubusercontent.com/88864503/134762996-ce5e27a6-bdd9-4206-8769-6cf072e186d7.png)

### 완성된 1차 모형!

- View Model 추가

### 1차 완성본에 대한 기능적/비기능적 요구사항을 커버하는지 검증
![image](https://user-images.githubusercontent.com/88864503/134763329-a91326bd-822e-4d28-8aec-30cb5144d3d7.png)

    - 고객이 도서를 선택하여 예약한다. (ok)
    - 결재를 진행한다. (ok)
    - 결재가 완료되면 예약 내역이 도서 관리 시스템에 업데이트 된다. (ok)
    - 도서 예약 취소시 환불처리 되고, 도서 관리 시스템에 업데이트 된다.(ok)
    - 고객은 중간중간 예약 현황을 조회한다. (View-green sticker 의 추가로 ok)

![image](https://user-images.githubusercontent.com/88864503/134763356-bf029e2e-8b97-44bc-a8d2-013d28434353.png)

    - 고객이 예약된 도서를 대여 / 반납 한다. (ok)
    - 도서에 대한 대여 / 반납이 이루어지면, 도서 관리 시스템에 업데이트 된다.(ok)  

### 비기능 요구사항에 대한 검증
![image](https://user-images.githubusercontent.com/88864503/134763371-1bed1eea-3c18-44dd-b663-0b917fdc2193.png)

- 마이크로 서비스를 넘나드는 시나리오에 대한 트랜잭션 처리
        - 도서예약 결제 처리:  결제가 완료되지 않은 예약은 처리되지 않는 다는정책에 따라, ACID 트랜잭션 적용. 예약 요청시 결제처리에 대해서는 Request-Response 방식 처리
        - 결제 완료 시 예약 완료 및 상태 변경 처리:  걸제에서  마이크로서비스로 결제내역이 전달되는 과정에 있어서 Book 마이크로 서비스가 별도의 배포주기를 가지기 때문에 Eventual Consistency 방식으로 트랜잭션 처리함.
        - 나머지 모든 inter-microservice 트랜잭션: 예약상태, 대여상태 등 모든 이벤트에 대해 MyPage처리 등, 데이터 일관성의 시점이 크리티컬하지 않은 모든 경우가 대부분이라 판단, Eventual Consistency 를 기본으로 채택함.
	- 도서 관리 기능이 수행되지 않더라도 예약 승인은 365일 24시간 받을 수 있어야 한다  Async (event-driven), Eventual Consistency
        - 결제시스템이 과중되면 사용자를 잠시동안 받지 않고 승인을 잠시후에 하도록 유도한다  Circuit breaker, fallback

## 헥사고날 아키텍처 다이어그램 도출
![image](https://user-images.githubusercontent.com/88864503/134763509-ed24a9c1-4a7d-44d7-8a8b-54ba19643847.png)

    - Chris Richardson, MSA Patterns 참고하여 Inbound adaptor와 Outbound adaptor를 구분함
    - 호출관계에서 PubSub 과 Req/Resp 를 구분함
    - 서브 도메인과 바운디드 컨텍스트의 분리:  각 팀의 KPI 별로 아래와 같이 관심 구현 스토리를 나눠가짐


## 구현:
분석/설계 단계에서 도출된 헥사고날 아키텍처에 따라, 각 BC별로 대변되는 마이크로 서비스들을 스프링부트로 구현하였다. 구현한 각 서비스를 로컬에서 실행하는 방법은 아래와 같다 (각자의 포트넘버는 8081 ~ 808n 이다)
```
cd book
mvn spring-boot:run

cd mypage
mvn spring-boot:run 

cd payment
mvn spring-boot:run  

cd rental
mvn spring-boot:run
   
```


## CQRS

도서 예약/결제/취소 등 총 Status 및 도서 ID에 대하여 고객이 조회 할 수 있도록 CQRS 로 구현하였다.
- rental, payment 등 개별 Aggregate Status 를 통합 조회하여 성능 Issue 를 사전에 예방할 수 있다.
- 비동기식으로 처리되어 발행된 이벤트 기반 Kafka 를 통해 수신/처리 되어 별도 Table 에 관리한다

 - mypage MSA PolicyHandler를 통해 구현
   ("reserved" 이벤트 발생 시, Pub/Sub 기반으로 별도 테이블에 저장)   
   ![image](https://user-images.githubusercontent.com/88864503/135394739-3a5ca6b3-ec5e-41e2-bac2-47e03f41dbff.png)
   
   
   ("statusUpdated" 이벤트 발생 시, Pub/Sub 기반으로 별도 테이블에 저장)
   ![image](https://user-images.githubusercontent.com/88864503/135394857-9e278dfa-257e-4b01-aa5b-1ae03655c6d3.png)


- 실제로 view 페이지를 조회해 보면 모든 사용자 ID 정보, 상태, 도서ID 등 의 정보를 종합적으로 알 수 있다.
  
  ![image](https://user-images.githubusercontent.com/88864503/135395090-e9dc5f3c-20f8-4e25-bb4a-9a5ccc0779a1.png)



## API 게이트웨이

 1. gateway 스프링부트 App을 추가 후 application.yaml내에 각 마이크로 서비스의 routes 를 추가하고 gateway 서버의 포트를 8080 으로 설정함
          - application.yaml 예시

       
       ![image](https://user-images.githubusercontent.com/88864503/134764372-341badf5-8472-4dd0-baa8-1e89eb0ed572.png)



## Correlation
vaccinereservation 프로젝트에서는 PolicyHandler에서 처리 시 어떤 건에 대한 처리인지를 구별하기 위한 Correlation-key 구현을 
이벤트 클래스 안의 변수로 전달받아 서비스간 연관된 처리를 정확하게 구현하고 있습니다. 

아래의 구현 예제를 보면

예약(Reserved)을 하면 동시에 연관된 도서관리(Book), 결제(payment) 등의 서비스의 상태가 적당하게 변경이 되고,
예약건의 취소를 수행하면 다시 연관된 서관리(Book), 결제(payment) 등의 서비스의 상태값 등의 데이터가 적당한 상태로 변경되는 것을
확인할 수 있습니다.

- 도서 예약 요청

http POST localhost:8081/rentals memberId=2 bookId=2

![image](https://user-images.githubusercontent.com/88864503/135395578-87113e6e-1f79-4cf0-8fde-3d6c93915b56.png)


- 사용자 예약 후 결제확인

http GET localhost:8082/payments/2

![image](https://user-images.githubusercontent.com/88864503/135395697-c76f5823-1a41-41dd-b300-b64a63f9ec6d.png)


- 사용자 도서 예약취소

http PATCH localhost:8081/rentals/2 reqState="cancel" 

![image](https://user-images.githubusercontent.com/88864503/135396367-93fe35a9-3b08-4f81-afbc-29bb05e7c5b0.png)


- 사용자 도서 예약 취소 후 - bookStatus가 "refunded" 된 것 확인

http GET localhost:8084/books   

![image](https://user-images.githubusercontent.com/88864503/135396723-15941d1d-559e-4902-bbbe-33bc8e9e74a4.png)


- 사용자 도서 대여 후 - bookStatus가 "rentaled" 된 것 확인

http PATCH localhost:8081/rentals/3 reqState="rental" 

![image](https://user-images.githubusercontent.com/88864503/135397498-5075dda3-0475-4015-99db-322ccf2149ed.png)
![image](https://user-images.githubusercontent.com/88864503/135397650-ce19ebd0-b0fe-4d34-ac42-95be84459850.png)


- 사용자 도서 반납 후 - bookStatus가 "reed" 된 것 확인

http PATCH localhost:8081/rentals/3 reqState="return"

![image](https://user-images.githubusercontent.com/88864503/135397828-ac12ebf7-b117-4f1d-b2ea-12d6ffb21f2b.png)





## DDD 의 적용

- 각 서비스내에 도출된 핵심 Aggregate Root 객체를 Entity 로 선언하였다. (예시는 Reservation 마이크로 서비스). 이때 가능한 현업에서 사용하는 언어 (유비쿼터스 랭귀지)를 그대로 사용하려고 노력했다. 현실에서 발생가는한 이벤트에 의하여 마이크로 서비스들이 상호 작용하기 좋은 모델링으로 구현을 하였다.

```
package library;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Book_table")
public class Book {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;
    private Long bookId;
    private String bookStatus;
    private Long memberId;
    private Long rendtalId;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }
    public Long getBookId() {
        return bookId;
    }

    public void setBookId(Long bookId) {
        this.bookId = bookId;
    }
    public String getBookStatus() {
        return bookStatus;
    }

    public void setBookStatus(String bookStatus) {
        this.bookStatus = bookStatus;
    }
    public Long getMemberId() {
        return memberId;
    }

    public void setMemberId(Long memberId) {
        this.memberId = memberId;
    }
    public Long getRendtalId() {
        return rendtalId;
    }

    public void setRendtalId(Long rendtalId) {
        this.rendtalId = rendtalId;
    }
}


```

- Entity Pattern 과 Repository Pattern 을 적용하여 JPA 를 통하여 다양한 데이터소스 유형 (RDB or NoSQL) 에 대한 별도의 처리가 없도록 데이터 접근 어댑터를 자동 생성하기 위하여 Spring Data REST 의 RestRepository 를 적용하였다

```
package library;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface BookRepository extends PagingAndSortingRepository<Book, Long>{
}
```

- 적용 후 REST API 의 테스트

```
#rental 서비스의 도서 예약 요청

#Book 서비스의 상태 업데이트

#Book 서비스의 도서 예약 상태 및 도서 유형 확인
```


## 동기식 호출(Sync) 과 Fallback 처리

분석단계에서의 조건 중 하나로 예약(reservation)->승인(approval) 간의 호출은 동기식 일관성을 유지하는 트랜잭션으로 처리하기로 하였다. 
호출 프로토콜은 이미 앞서 Rest Repository 에 의해 노출되어있는 REST 서비스를 FeignClient로 이용하여 호출하도록 한다.

- 결제서비스를 호출하기 위하여 Stub과 (FeignClient) 를 이용하여 Service 대행 인터페이스 (Proxy) 를 구현 

```
#PaymentService.java

@FeignClient(name="payment", url="${api.payment.url}") public interface PaymentService {

@RequestMapping(method= RequestMethod.POST, path="/payments")//, fallback = PaymentServiceFallback.class)
public void payship(@RequestBody Payment payment);
}

```

- 예약 이후(@PostPersist) 결제를 요청하도록 처리

```
#Rental.java

@PostPersist
public void onPostPersist(){
    Reserved reserved = new Reserved();
    BeanUtils.copyProperties(this, reserved);
    reserved.publishAfterCommit();


    //Following code causes dependency to external APIs
    // it is NOT A GOOD PRACTICE. instead, Event-Policy mapping is recommended.
    library.external.Payment payment = new library.external.Payment();
    // mappings goes here
    payment.setId(this.id);
    payment.setMemberId(this.memberId);
    payment.setBookId(this.bookId);
    payment.setReqState("reserve");

    RentalApplication.applicationContext.getBean(library.external.PaymentService.class)
        .payship(payment);
}
    
```

- 동기식 호출에서는 호출 시간에 따른 타임 커플링이 발생하며, 결제 시스템이 장애가 나면 예약도 못받는다는 것을 확인

```
# 결제 (payment) 서비스를 잠시 내려놓음

#예약처리

http POST localhost:8081/rentals memberId=4 bookId=4  #Fail 

![image](https://user-images.githubusercontent.com/88864503/135398361-f06ffa40-2f1d-4ae0-bbdc-b11bb04acb34.png)

#결제서비스 재기동

cd payment
mvn spring-boot:run

#주문처리

http POST localhost:8081/rentals memberId=4 bookId=4   #Success

![image](https://user-images.githubusercontent.com/88864503/135398559-b694ad56-3ae2-4f9e-945d-95fb67795dc5.png)

```

- 또한 과도한 요청시에 서비스 장애가 도미노 처럼 벌어질 수 있다. 


## 비동기식 호출 / 시간적 디커플링 / 장애격리 / 최종 (Eventual) 일관성 테스트

결제 이후 도서관리(book)시스템으로 결제 완료 여부를 알려주는 행위는 비 동기식으로 처리하여 
도서관리 시스템의 처리로 인해 결제주문이 블로킹 되지 않도록 처리한다.
- 이를 위하여 결제이력에 기록을 남긴 후에 곧바로 결제승인(paid)이 되었다는 도메인 이벤트를 카프카로 송출한다(Publish)

```
# Payment.java

@Entity
@Table(name="Payment_table")
public class Payment {

 ...
    @PostPersist
    public void onPostPersist(){
        Paid paid = new Paid();
        BeanUtils.copyProperties(this, paid);
        paid.publishAfterCommit();
 ...
}
    



- 도서관리 서비스는 결제완료 이벤트를 수신하여 자신의 정책을 처리하도록 PolicyHandler 를 구현한다:

# PolicyHandler.java (book)
...

@Service
public class PolicyHandler{

    @StreamListener(KafkaProcessor.INPUT)
    public void wheneverPaid_(@Payload Paid paid){
        // 결제완료(예약)
        if(paid.isMe()){
            Book book = new Book();
            book.setId(paid.getBookId());
            book.setMemberId(paid.getMemberId());
            book.setRendtalId(paid.getId());
            book.setBookStatus("reserved");

            bookRepository.save(book);
        }
    }
}
```
도서관리 시스템은 대여/결제와 완전히 분리되어있으며, 이벤트 수신에 따라 처리되기 때문에, 도서관리시스템이 유지보수로 인해 잠시 내려간 상태라도 주문을 받는데 문제가 없다:


```
#도서관리 서비스 (book) 를 잠시 내려놓음

#예약 처리

http POST localhost:8081/rentals memberId=5 bookId=5  #Success  

![image](https://user-images.githubusercontent.com/88864503/135398862-48ac573d-95fa-42e0-8e33-7edd52617e60.png)


#상점 서비스 기동

cd book
mvn spring-boot:run

#주문상태 확인

http GET localhost:8083/mypages/7     # 주문의 상태가 "reserved"으로 확인

![image](https://user-images.githubusercontent.com/88864503/135399664-88d41054-9967-438c-92a9-78e7a2ac40e4.png)


## 폴리글랏 퍼시스턴스
전체 서비스의 경우 빠른 속도와 개발 생산성을 극대화하기 위해 Spring Boot에서 기본적으로 제공하는 In-Memory DB인 H2 DB를 사용하였다.

```
package library;

import javax.persistence.*;
import org.springframework.beans.BeanUtils;
import java.util.List;

@Entity
@Table(name="Rental_table")
public class Rental {

    @Id
    @GeneratedValue(strategy=GenerationType.AUTO)
    private Long id;         //예약번호
    private Long memberId;  // 사용자번호
    private Long bookId;    // 책번호
    private String reqState;//요청: "reserve", "cancel", "rental", "return"



# 운영

## CI/CD 설정
각 구현체들은 각자의 source repository 에 구성되었고, 사용한 CI/CD 플랫폼은 AWS를 사용하였으며, pipeline build script 는 각 프로젝트 폴더 이하 buildspec.yml 에 포함되었다.

AWS CodeBuild 적용 현황
![1](https://user-images.githubusercontent.com/88864503/133553458-2ecf1f10-3c01-4b3d-bcaa-84f268f7848a.JPG)

webhook을 통한 CI 확인
![2](https://user-images.githubusercontent.com/88864503/133553865-81afb01b-dbec-4167-bac9-c8fd62ea5718.JPG)

AWS ECR 적용 현황
![3](https://user-images.githubusercontent.com/88864503/133553933-30d2ba69-ec96-4b26-838b-33e2d061bb70.JPG)

EKS에 배포된 내용
![4](https://user-images.githubusercontent.com/88864503/133554057-b6c08a0a-04ce-4dd5-bc47-f01e9776373d.JPG)

## ConfigMap 설정


 동기 호출 URL을 ConfigMap에 등록하여 사용


 kubectl apply -f configmap

```
 apiVersion: v1
 kind: ConfigMap
 metadata:
   name: vaccine-configmap
   namespace: vaccines
 data:
   apiurl: "http://user02-gateway:8080"

```
buildspec 수정

```
              spec:
                containers:
                  - name: $_PROJECT_NAME
                    image: $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$_PROJECT_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
                    ports:
                      - containerPort: 8080
                    env:
                    - name: apiurl
                      valueFrom:
                        configMapKeyRef:
                          name: vaccine-configmap
                          key: apiurl 
                        
```            
application.yml 수정
```
prop:
  aprv:
    url: ${apiurl}
``` 

동기 호출 URL 실행
![5](https://user-images.githubusercontent.com/88864503/133554760-b8d8b524-ebbf-46dc-ba32-1820cffcc023.JPG)

## 무정지 재배포

* 먼저 무정지 재배포가 100% 되는 것인지 확인하기 위해서 Autoscaler 이나 CB 설정을 제거함

- seige 로 배포작업 직전에 워크로드를 모니터링 함.
```
siege -c100 -t10S -v --content-type "application/json" 'http://af9a234af8e354f5299f1d049a1b21c0-1150269307.ap-northeast-1.elb.amazonaws.com:8080/reservations

```

```
# buildspec.yaml 의 readiness probe 의 설정:

                    readinessProbe:
                      httpGet:
                        path: /actuator/health
                        port: 8080
                      initialDelaySeconds: 10
                      timeoutSeconds: 2
                      periodSeconds: 5
                      failureThreshold: 10
```

Customer 서비스 신규 버전으로 배포
![9](https://user-images.githubusercontent.com/88864503/133559167-4a2ede3c-ad33-43b6-b101-8759d56dd0c4.png)


배포기간 동안 Availability 가 변화없기 때문에 무정지 재배포가 성공한 것으로 확인됨.

## Liveness Probe

테스트를 위해 buildspec.yml을 아래와 같이 수정 후 배포

```
livenessProbe:
                      # httpGet:
                      #   path: /actuator/health
                      #   port: 8080
                      exec:
                        command:
                        - cat
                        - /tmp/healthy
```


![6](https://user-images.githubusercontent.com/88864503/133556583-3315fae7-de8a-4882-ad1d-8493fbd2daa8.png)

pod 상태 확인
 
 kubectl describe ~ 로 pod에 들어가서 아래 메시지 확인
 ```
 Warning  Unhealthy  26s (x2 over 31s)     kubelet            Liveness probe failed: cat: /tmp/healthy: No such file or directory
 ```

/tmp/healthy 파일 생성
```
kubectl exec -it pod/reservation-5576944554-q8jwf -n vaccines -- touch /tmp/healthy
```
![7](https://user-images.githubusercontent.com/88864503/133556724-7693dec2-41dd-430c-a3d3-389cc309bfca.png)

성공 확인
