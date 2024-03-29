---
description: 본 장에선 일반적인 웹 서버를 확장성 있는 구조로 구성하기 위해 사용되는 컴포넌트들에 대해 개괄적으로 설명하고 몇 가지 주의점을 전달합니다.
---

# Chapter 1. 사용자 수에 따른 규모 확장성

## 데이터베이스

<figure><img src="../.gitbook/assets/Untitled (5).png" alt="" width="375"><figcaption><p>웹 계층과 데이터 계층을 분리하면 각각의 상황에 따라 스케일링할 수 있다.</p></figcaption></figure>

* 대부분의 상황에 RDBMS는 좋은 선택지지만 아래와 같은 요구사항이 있을 경우엔 NoSql을 고려해볼 수도 있다.
  * 아주 낮은 응답 지연시간이 요구됨
  * 다루는 데이터가 비정형이라 관계형 데이터가 아님
  * 데이터를 직렬화하거나 역직렬화 할 수 있기만 하면 됨
  * 아주 많은 양의 데이터를 저장할 필요가 있음

***

## 수직적 확장 VS 수평적 규모 확장

* 수직적 확장 비교적 쉽긴 해도 결국 CPU, Memory의 성능상 한계를 맞이한다.
* 또한 장애에 대한 failover나 다중화 방안이 지원되지 않는다. 즉, 서버가 다운되면 서비스 자체가 중단된다.
* 때문에 로드 밸런서를 활용한 수평적 확장이 대규모 서비스에서 더 요구된다.

### Load Balancer

<figure><img src="../.gitbook/assets/Untitled 1 (2).png" alt="" width="375"><figcaption></figcaption></figure>

* 클라이언트가 DNS를 통해 서버가 아닌 로드밸런서로 요청을 보내고, 로드밸런서가 각 서버로 리퀘스트를 할당해준다. 일종의 프록시 서버의 역할
* 이를 통해 서버의 failover를 구현할 수 있다. 장애가 발생한 서버엔 리퀘스트를 전달하지 않음으로써 가용성이 향상된다. 또한 배포 전략을 롤링이나 블루/그린 등으로 선택하여 무중단 배포가 가능해진다.



### DB 다중화

<figure><img src="../.gitbook/assets/Untitled 2 (1).png" alt="" width="375"><figcaption></figcaption></figure>

* 읽기 + 쓰기 전용인 Main Server와 읽기 전용인 Replica Server를 통해 DB의 가용성을 증가시킬 수 있다. 본 도서에선 Master, Slave Server라고 명시했지만 해당 용어는 더 이상 쓰지 않는 점을 인지하고 있자. → 단, Mysql 같은 RDBMS에서 하위 호환성을 위해 설정 정보 등에 master, slave라는 단어를 쓰기는 한다. 알아만두자
* DB 다중화에 따라 쿼리의 병렬 처리가 가능해지고, 안정성이 증대된다. 또한 가용성도 증가하는 결과를 일으킨다.
* replica 서버가 다운되면 다른 레플리카 서버가 읽기 작업을 분산하여 수행하며, 레플리카가 한 대였는데 이게 다운된 경우엔 메인 서버가 읽기 작업까지 수행한다.
* 소스 서버가 다운되면 레플리카 서버 중 하나가 새로운 메인 서버로 추대된다. 이 때 동기화 되지 못한 데이터가 소실될 수 있기 때문에 Tranaction log, Relay log를 통한 복구가 필요 할 수 있다.

<figure><img src="../.gitbook/assets/Untitled 3 (1).png" alt="" width="375"><figcaption><p>지금까지의 내용들이 포함된 아키텍쳐</p></figcaption></figure>

***

## Cache

<figure><img src="../.gitbook/assets/Untitled 4 (1).png" alt="" width="563"><figcaption></figcaption></figure>

* 캐시 계층은 데이터를 잠시 보관하는 계층으로 DB보다 훨씬 빠르며 독립적으로 규모를 확장시킬 수 있다.
* 요청에 따른 결과를 반환하기 위해 캐시를 먼저 탐색하고 유무로 따라 데이터베이스를 탐색하는 방식을 읽기 주도형 캐시 전략이라고 한다. 이외의 캐시 전략들은 여러 가지가 있다. \[_6번 자료 참조,_ [_Caching Starategies and How to Choose the Right One_](https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one)_]_

### 유의점

* 데이터의 갱신이 잦지 않고, 조회가 잦은 경우에 적합하다.
* 휘발성 데이터만 캐시 저장소에 보관하는 것이 좋다. 중요 데이터가 있다면 DB같은 영속성 저장소에 계속 저장하고, 캐시에 부차적으로 저장하는 것이 좋다.
* 캐시 만료 정책을 잘 마련해둬야 한다. 너무 긴 보관 기간은 비효율을 야기하고 원본과 다르게 유지될 가능성이 증대된다. 또한 너무 짧은 보관 기간 역시 원본 데이터만 보관하는 것보다 좋은 효율성을 보이지 않는다.
* 데이터베이스와 캐시의 일관성을 유지할 수 있어야 한다. 원본 저장소와 캐시의 데이터 변경 트랜잭션이 깨지지 않도록 일관성을 유지해야한다.
* 캐시 저장소가 분산되지 않고 하나의 서버에 저장되어 있는 경우라면, 캐시 저장소가 단일 장애 지점이 될 수 있으므로 유의해야한다.
  * 단일 장애 지점이란, “어떤 특정 지점에서의 장애가 전체 시스템의 동작을 중단시켜버릴 수 있는 경우, 우리는 해당 지점을 단일 장애 지점이라고 부른다.” - 위키피디아
* 캐시 메모리의 크기도 중요하다. 메모리가 너무 작으면 캐시 성능이 떨어진다.
* 캐시가 꽉 찼을 떄, 새로운 데이터를 쌓기 위해 기존 데이터를 일부 날려야 할 수 있다. Least Recently Used 방식이나, Least Frequency Used, FIFO 등의 정책이 있으므로 경우에 맞게 적용 해야한다.

***

### CDN

<figure><img src="../.gitbook/assets/Untitled 5 (1).png" alt="" width="375"><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/Untitled 6.png" alt="" width="375"><figcaption></figcaption></figure>

* Contents Deliver Network의 약어. 즉, 정적 컨텐츠를 전달하는 지리적으로 분산된 서버의 네트워크다. 물리적으로 가까운 위치에서 정적 컨텐츠 (CSS, HTML 등)을 전송하기 위해 사용한다.
* CDN은 보통 3rd party를 통해 운영되며, CDN으로 입출력되는 데이터의 양에 따라 과금액이 결정된다. 따라서 CDN에 저장될 컨텐츠는 자주 사용되는 놈들만 골라서 저장하는게 좋다.
* 만료 시한이 너무 길거나 짧으면 곤란하다. 길면 컨텐츠 신선도가 떨어지고, 짧으면 효율성이 떨어진다.
* CDN 서버의 장애에 따른 대응 방안도 미리 마련되어 있어야 한다.
* 아직 만료되지 않은 컨텐츠라도 CDN 사업자가 제공하는 API나 오브젝트 버저닝을 통해 컨텐츠를 무효화 할 수 있다.

<figure><img src="../.gitbook/assets/Untitled 7.png" alt="" width="375"><figcaption><p>CDN이 추가된 서버 아키텍쳐</p></figcaption></figure>

***

### Stateless

* 상태 정보를 서버에 보관하는 방식은 확장성에 있어서 치명적인 약점을 갖는다. 따라서 서버가 별도로 상태 정보를 보관하기 보단 요청에 따라 처리되는 무상태 아키텍쳐를 유지하는 것이 바람직하다.

<figure><img src="../.gitbook/assets/Untitled 8.png" alt="" width="375"><figcaption><p>상태 정보를 저장하는 서버의 사례. 사용자와 서버가 꼬이면 오류가 발생할 가능성이 매우 높다.</p></figcaption></figure>

<figure><img src="../.gitbook/assets/Untitled 9.png" alt="" width="375"><figcaption><p>무상태 아키텍쳐 예. 사용자의 HTTP 요청이 어떤 웹 서버로 전달되더라도 이상이 없다.</p></figcaption></figure>

* 무상태 서버에선 상태 정보가 필요할 때 공유저장소에서 데이터를 조회한다.
* 따라서 상태 정보는 웹 서버로부터 물리적으로 분리되어 있다.

***

### 데이터센터

<figure><img src="../.gitbook/assets/Untitled 10.png" alt=""><figcaption><p>이중화된 데이터 센터</p></figcaption></figure>

* 데이터센터를 다중화하는 방식은 몇가지 어려운 부분들을 극복해야한다.
  * 트래픽 우회: 어느 데이터센터로 트래픽을 전송하는 것이 가장 올바른 방법일지를 정해야한다.
  * 데이터 동기화: 데이터 센터 별로 DB가 분리되어 있을 때 각 데이터 센터 간의 데이터가 일치하지 않는 상황이 발생할 수 있다. 이를 막기 위해 데이터를 여러 데이터센터에 걸쳐 다중화할 수도 있다.
  * 테스트와 배포: 다중 데이터센터를 사용할 때 시스템을 배포하는 상황에서 여러 위치에 배포하고 테스트 해봐야한다. 이 때 자동화된 배포와 테스트 도구가 큰 역할을 한다.
* 시스템을 더 큰 규모로 확장하기 위해선 시스템의 컴포넌트를 분리하여 각각의 독립적인 객체로 확장할 수 있도록 구성해야한다. 이를 위해 메시지 큐를 많이들 사용한다.

### 메시지 큐

* 메시지 큐는 메시지가 일단 큐에 들어가면 Consumer가 소비하기 까지`(메시지 큐에 따라 소비한 이후로도 유지되기도 한다. like kafka)` 무손실성을 갖는 비동기 통신 지원 컴포넌트다.
* 보통 발행자, 소비자로 구성된 pub/sub 형태고 중간에 메시지 큐가 매개체 역할을 한다.

<figure><img src="../.gitbook/assets/Untitled 11.png" alt=""><figcaption></figcaption></figure>

* 메시지 큐를 사용하면 서비스 간 결합이 느슨해지기 때문에 규모 확장성이 보장되어야 하는 어플리케이션에 적합하다.

<figure><img src="../.gitbook/assets/Untitled 12.png" alt=""><figcaption><p>메시지 큐가 사용되는 서비스 예시. 시간이 오래 걸리는 작업은 큐에 넣어두고 순차적으로 처리한다.</p></figcaption></figure>

***

### 로그, 메트릭 그리고 자동화

* 로그: 에러 로그 모니터링은 중요하며 유용하다. CloudWatch 사랑해…
* 메트릭: 메트릭은 사업 현황 파악 및 시스템 상태 파악에 유용하다.
  * 호스트 단위 메스틱: CPU, 메모리, 디스크IO에 대한 메트릭
  * 종합 메트릭 데이터베이스 계층의 성능, 캐시 계층의 성능
  * 핵심 비즈니스 메트릭: DAU, 수익, 재방문 등
* 자동화: 시스템이 크고 복잡해지면 생산성을 위해 자동화 도구를 활용해야한다. CI/CD, 자동화 테스트 등이 여기에 해당한다. 가성비가 아주 높은 도구들이니 잘 활용할 수 있도록 해야한다.

<figure><img src="../.gitbook/assets/Untitled 13.png" alt=""><figcaption><p>도구들까지 포함된 아키텍쳐</p></figcaption></figure>

***

### 데이터베이스 규모의 확장

<figure><img src="../.gitbook/assets/Untitled 14.png" alt=""><figcaption></figcaption></figure>

* 수직 확장: 서버 성능을 높여서 처리량과 저장 공간을 늘리는 방식이다. 으레 갈수록 훨씬 비싸지고 가용성이 떨어지기 때문에 권하기 어렵다. 또한 결국 물리적인 한계점이 존재한다.
* 수평 확장: DB의 수평적 확장을 샤딩이라고도 한다. 모든 샤드는 같은 스키마를 사용하지만, 샤드에 보관되는 데이터 사이에는 중복이 없다.

<figure><img src="../.gitbook/assets/Untitled 15.png" alt=""><figcaption></figcaption></figure>

* 샤딩 전략을 구성하기 위해선 샤딩 키(혹은 파티션 키) 전략이 가장 중요한 역할을 한다. 얘를 잘 쪼개야 데이터 조회나 변경에 따른 비효율이 줄어든다.
* 데이터베이스 샤딩에 따라 고려해야할 점
  * 데이터 재샤딩: 샤드가 늘어나면서 기존 샤드에 적채된 데이터를 분배해야 하는 상황이 발생한다. 혹은 하나의 샤드에 집중적으로 데이터가 몰리면서 특정 샤드만 빠르게 소진되는 문제가 발생할 수 있다. 안정 해시 기법을 활용하면 이 문제는 해결이 가능하다.
  * 유명인사 문제: 핫스팟 키 문제라고도 부르는데 특정 샤드에 질의가 집중되어서 샤드에 과부하가 걸리는 문제다. 특정 데이터로 인한 샤드 집중 포화가 가능하므로 유의해야한다.
  * 조인과 비정규화: 하나의 데이터베이스를 여러 샤드로 쪼개고 나면 각 샤드에 포함된 데이터를 조인하는 것은 어렵다. 이를 해결하기 위해 비정규화를 통해 하나의 테이블에서 질의가 이뤄지게 변경할 수도 있다.

***

### Summary

<figure><img src="../.gitbook/assets/Untitled 16.png" alt="" width="375"><figcaption></figcaption></figure>
