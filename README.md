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
- fetch() : 리스트 조회, 데이터 없으면 빈 리스트 반환
- fetchOne() : 단 건 조회
  - 결과가 없으면 : null
  - 결과가 둘 이상이면 : com.querydsl.core.NonUniqueResultException
- fetchFirst() : limit(1).fetchOne()
- fetchResults() : 페이징 정보 포함, total count 쿼리 추가 실행
  - Spring Data JPA에서도 나온거긴 한데, count 쿼리가 단독으로 사용되면 조인을 하지 않아도 되는 상황에서도 이런경우 조인을 하게 될 수도 있어.
  - 결과에는 당연히 문제가 없지만, 복잡한 쿼리라면 성능상 이점을 위해서 count 쿼리를 단독으로 쓰는게 나음
- fetchCount() : count 쿼리로 변경해서 count 수 조회

#### 정렬
- 생략
#### 페이징
- 생략
#### 집합
- 생략

#### 조인 - 기본 조인
- join(조인 대상, 별칭으로 사용할 Q타입)
```
List<Member> result = queryFactory
              .selectFrom(member)
              .join(member.team, team)
              .where(team.name.eq("teamA"))
              .fetch();
```
- 원래 기본 sql 문법은 "select * from member m join team t on m.id = t.id" 이거나 "select * from member m, team t where m.id = t.id" 이런식으로다가
- 어떤 조건을 가지고 조인을 걸어줄것인지 따로 명시하는데
- 여기서는 연관관계가 서로 걸려있는것이기에 조인 대상만 member.team 으로 해주면 pk fk id 잘 찾아서 조인 해줌.
- JPA 기본 강의에서 JPQL 파트 보면 얘도 이런식으로 조인하는걸 알 수 있음. 만약 id값을 기준으로 조인하고 싶지 않다면,
- .from(member).join(team).on(조인조건) 으로 하면됨

#### 조인 - on절
- join할때 on 절은 join 할 대상을 필터링 하는 느낌이잖아.
- 근데 이게 leftJoin이나 rightJoin에서는 의미가 있지만
- 사실 innerJoin 에서는 QueryDsl 문법상 어차피 join() 매개변수로 member.team가 들어가면 id를 기준으로 조인할지 알려주는것이기에
- 굳이 on절에 필터링 조건을 걸 필요 없고 그냥 where절에 쓰는게 낫다. 물론 outerJoin의 결과를 원할경우에는 where로는 해결이 불가하니 on절 따로 써줘야지
- 테스트코드 extracting
  - 만약 Member List에서 해당 Member들의 이름을 테스트하려면?
  ```
  List<String> names = new ArrayList<>();
  for (Member member : members) {
      names.add(member.getName());
  }
  assertThat(names).containsOnly("dexter", "james", "park", "lee");
  ```
  - 엄청 번거로움. 이때 사용하는게 extracting
  ``` 
  assertThat(members)
          .extracting("name")
          .containsOnly("dexter", "james", "park", "lee");
  ```
  - 리스트인 members에서 member 객체들마다 name 필드를 추출해서 비교해줌.
  - containsOnly : 순서, 중복을 무시하는 대신 원소값과 갯수가 정확히 일치
  - containsExactly: 순서를 포함해서 정확히 일치

#### 조인 - 페치 조인
- .join(member.team, team).fetchJoin()

#### 서브 쿼리
```
QMember memberSub = new QMember("memberSub");
      List<Member> result = queryFactory
              .selectFrom(member)
              .where(member.age.eq(
                      JPAExpressions
                              .select(memberSub.age.max())
                              .from(memberSub)
              ))
              .fetch();
```
- 구별되는 별칭이 필요하기에 첫줄에 저렇게 따로 별칭 명시해서 QType 생성함.
- JPAExpressions 얘는 static import 가능
- jpql 배울때도 언급한거지만 from절에 서브쿼리 지원 안함.
- 해결방법
  - 조인으로 푼다
  - 쿼리를 나눈다
  - 정 한방 쿼리로 나타내고 싶다면 그때는 nativeSQL 쓴다.

#### Case 문
- 물론 case 문 다 있는데, 쓸수도 있는데.
- db에서는 로우 데이터를 최소한의 필터링하고 그룹핑해서 가져오는 역할만 하고,
- 값이 a면 b로 바꾸고 뭐 이런 전환하고 바꾸고 보여주는건 데이터 가져와서 애플리케이션에서 혹은 프래젠테이션 계층에서 진행하는게 나음.

#### 상수, 문자 더하기
- .stringValue() : 문자가 아닌 타입을 문자로 바꿔줌. int도 그렇지만 enum 타입들한테 많이 사용함

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














