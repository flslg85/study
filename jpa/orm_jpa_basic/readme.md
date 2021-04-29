# ORM JPA Basic

## Chap 0. 강좌 소개

* jdbc -> jdbc template, mybatis -> jpa
  * sql 작성할 필요가 없음
  * sql 작성은 생산성이 떨어질수 밖에 없음
  * 개발 속도와 유지보수에서 차이가 남


#### jpa 실무에서 어려운 이유
* 처음 jpa 나 스프링 데이터 jpa 만나면?
* SQL 자동화, 수십줄의 코드가 한두줄로!
  -> sql 안짜도 되고 예제가 쉬워서 금방 할꺼 같은 느낌이 듬

* 실무에 바로 도입하면?
  * 예제들은 보통 테이블이 한 두개로 단순함
  * 실무는 수십개 이상의 복잡한 객체와 테이블 사용


#### 목표 -> 객체와 테이블 설계 매핑
* 객체와 테이블을 제대로 설계하고 매핑하는 방법
* 기본키와 외래키 매핑
* 1:N, N:1, N:M 매핑
* 어떠한 복잡한 시스템도 JPA 도 설계 가능


#### 목표 - JPA 내부 동작 방식 이해
* JPA 의 내부 동작 방식을 이해하지 못하고 사용
* JPA 내부 동작 방식을 그림과 코드로 자세히 설명
* JPA 가 어떤 SQL을 만들어내는지 이해
* JPA 가 언제 SQL을 실행하는지 이해


#### JPA 기본편 학습 방법
* JPA 표준 스펙만 500 페이지로 방대함
* 강의는 이론 + 라이브 코딩
* 강의를 메인으로하고 책은 참고서로 추천


#### JPA 를 많이 사용하나요?
* 대한민국 구글 검색량 비교
* https://trends.google.com/trends/explore?geo=US&q=jpa,mybatis,ibatis


#### JPA 실무 경험
* 단순한 SQL 작성으로 시간을 낭비하지 않음
* 남는 시간에 더 많은 설계 고민, 테스트 코드 작성


## Chap 1. JPA 소개

### SQL 중심적인 개발의 문제점

* 객체를 관계형 디비에 관리하는 시대
  * 애플리케이션 - 객체지향 언어
  * 데이터베이스 - 관계형 디비


* 대부분 개발이 SQL 쿼리 작성으로 SQL 에 의존적인 개발을 피하기 어려움
  * SQL 무한반복, 지루한 코드

* 패러다임의 불일치 => 객체답게 모델링 할수록 매핑 작업만 늘어남
  * 객체를 영구 보관하는 다양한 저장소
    * RDB, NoSQL, File
    * 현실적인 대안은 관계형 데이터 베이스

  * 객체 -> SQL 변환 -> SQL -> RDB
    * 개발자 = SQL 매퍼..

  * 객체와 관계형 데이터베이스의 차이

* 객체를 자바 컬렉션에 저장하듯이 DB 에 저장할 수는 없을까?

  
### JPA 소개

* JPA ?
  * Java Persistence API
  * 자바 진영의 ORM 기술 표준
  * JPA 는 애플리케이션과 JDBC 사이에서 동작

* ORM
  * Object-relational mapping(객체 관계 매핑)
  * ORM 프레임워크가 중간에서 매핑
  * 대중적인 언어에는 대부분 ORM 기술이 존재

* JPA 표준 명세
  * JPA 는 인터페이스의 모음
  * JPA 2.1 표준 명세를 구현한 3가지 구현체
  * 하이버네이트, EclipseLink, DataNucleus

* JPA 버전
  * JPA 1.0(JSR 220) 2006년: 초기 버전, 복합키와 연관관계 기능이 부족
  * JPA 2.0(JSR 317) 2009년: 대부분의 ORM 기능을 포함, JPA Criteria 추가
  * JPA 2.1(JSR 338) 2013년: 스토어드 프로시저 접근, Converter, 엔티티 그래프 기능이 추가


* JPA를 왜 사용해야 하는가?
  * SQL 중심적인 개발에서 객체 중심을 개발
  * 생산성
    * 저장: jpa.persist(member)
    * 조회: Member member = jpa.find(memberId) 
    * 수정: member.setName(“변경할 이름”)
    * 삭제: jpa.remove(member)
  * 유지보수
    * 필드 변경시 모든 SQL 수정 - mybatis
    * jpa 필드만 수정하면 됨
  * 성능
    * 1차 캐시와 동일성(identity) 보장
      * 같은 트랜잭션 안에서는 같은 엔티티를 반환
      * DB Isolation Level 이 Read Commit 이어도 애플리케이션에서 Repeatable Read 보장
    * 트랜잭션을 지원하는 쓰기 지연(transactional write-behind)
      * 트랜잭션을 커밋할 때까지 insert sql 을 모음
      * jdbc batch sql 기능을 사용해서 한번에 SQL 전송
    * 지연 로딩(lazy loading)
      * 객체가 실제 사용될때 로딩
      * 필요시 조인하여 즉시 로딩할수도 있음
  * 패러다임의 불일치 해결
  * 데이터 접근 추상화와 벤더 독립성
  * 표준

