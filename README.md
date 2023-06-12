# querydsl-study

### 프로젝트 환경설정
#### Querydsl 설정과 검증
- 설정이 되게 좀 까다롭네
- build.gradle에 설정 추가
  - plugins에 추가 id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
  - dependencies에 추가 implementation 'com.querydsl:querydsl-jpa'
  - 정확히는 모르겠는데, 컴파일 해서 Qtype 파일을 생성하는 경로를 잡아주는듯
    ```
    def querydslDir = "$buildDir/generated/querydsl"
    querydsl {
        jpa = true
        querydslSourcesDir = querydslDir
    }
    sourceSets {
        main.java.srcDir querydslDir
    }
    configurations {
        querydsl.extendsFrom compileClasspath
    }
    compileQuerydsl {
        options.annotationProcessorPath = configurations.querydsl
    }
    ```
- Qtype 파일 생성을 위한 컴파일
  - Gradle Tasks build clean
  - Gradle Tasks other compileQuerydsl
  - build generated querydsl..에 Q파일 생성되었나 확인. 
  - 어떤 원리인지는 모르겠으나, java에서 entity 폴더에 만들어둔 엔티티객체가 실제로 Q타입으로 변환된 clsss 파일이 생성되네. @Entity 애노테이션 달아준 클래스를 기준으로 만들어주는건가.


### 예제 도메인 모델
#### 예제 도메인 모델과 동작확인
- 테스트코드에서 실행시키면 자동 롤백되잖아. 그거 막을때 @Rollback(false) 했었는데 @Commit 이것도 동일한 기능임.


### 기본 문법
#### 시작 - JPQL vs Querydsl
- JPQL
  ```
  String qlString =
          "select m from Member m " +
          "where m.username = :username";
  Member findMember = em.createQuery(qlString, Member.class)
          .setParameter("username", "member1")
          .getSingleResult();
  ```
- Querydsl
  ```
  JPAQueryFactory queryFactory = new JPAQueryFactory(em); // 얘는 필드로 빼도 됨
  QMember m = new QMember("m");
  
  Member findMember = queryFactory
              .select(m)
              .from(m)
              .where(m.username.eq("member1"))//파라미터 바인딩 처리
              .fetchOne();
  ```
#### 기본 Q-Type 활용
- QMember m = new QMember("m"); "m" 이부분이 테이블의 별칭 개념
- 셀프 조인 하는 경우가 아니라면 여기서는 별칭을 뭐 따로 줄 필요 없고
- 때문에, QMember qMember = QMember.member; 이걸로 사용.
- 여기에 static import 까지 하면, 그냥 member로 사용가능
- 최종 형태
  ```
  Member findMember = queryFactory
                .select(member)
                .from(member)
                .where(member.username.eq("member1"))
                .fetchOne();
  ```
- 사실상 QueryDsl은 jpql의 빌더 역할.
- jpql 어떻게 나가는지도 확인하고 싶으면 설정 추가
  - spring.jpa.properties.hibernate.use_sql_comments: true

#### 검색 조건 쿼리
- 뭐 이걸 가능한 문법별로 다 기록할 필요는 없을것 같고, 그냥 . 찍으면 가능한 메소드들 쭉 나오니까.
- sql문을 먼저 떠올리고, 이런거 있지 않을까 싶으면 대충 거의 다 있는듯.

#### 결과 조회
#### 정렬
#### 페이징
#### 집합
#### 조인 - 기본 조인
#### 조인 - on절
#### 조인 - 페치 조인
#### 서브 쿼리
#### Case 문
#### 상수, 문자 더하기


### 중급 문법
#### 프로젝션과 결과 반환 - 기본
#### 프로젝션과 결과 반환 - DTO 조회
#### 프로젝션과 결과 반환 - @QueryProjection
#### 동적 쿼리 - BooleanBuilder 사용
#### 동적 쿼리 - Where 다중 파라미터 사용
#### 수정, 삭제 벌크 연산
#### SQL function 호출하기


### 실무 활용 - 순수 JPA와 Querydsl
#### 순수 JPA 리포지토리와 Querydsl
#### 동적 쿼리와 성능 최적화 조회 - Builder 사용
#### 동적 쿼리와 성능 최적화 조회 - Where절 파라미터 사용
#### 조회 API 컨트롤러 개발


### 실무 활용 - 스프링 데이터 JPA와 Querydsl
#### 스프링 데이터 JPA 리포지토리로 변경
#### 사용자 정의 리포지토리
#### 스프링 데이터 페이징 활용1 - Querydsl 페이징 연동
#### 스프링 데이터 페이징 활용2 - CountQuery 최적화
#### 스프링 데이터 페이징 활용3 - 컨트롤러 개발


### 스프링 데이터 JPA가 제공하는 Querydsl 기능
#### 인터페이스 지원 - QuerydslPredicateExecutor
#### Querydsl Web 지원
#### 리포지토리 지원 - QuerydslRepositorySupport
#### Querydsl 지원 클래스 직접 만들기














