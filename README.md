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
- member.age.goe(30) // age >= 30 // greater than or equal
- member.age.gt(30) // age > 30 // greater than
- member.age.loe(30) // age <= 30 // less than or equal
- member.age.lt(30) // age < 30 // less than

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
- select 절에 여러개의 매개변수가 있을 경우 List\<Tuple>로 반환됨.
- Tuple은 queryDsl 종속적인 객체라 되도록 repository 안에서 만 사용하고 밖으로 나갈때는 dto로 변환해서 데이터를 보내는게 좋음.
- tuple.get(member.username);

#### 프로젝션과 결과 반환 - DTO 조회, @QueryProjection
- 3가지 방식이 있어. 파라미터방식(setter), 필드방식, 생성자방식
- 그리고 생성자 방식은 @QueryProjection을 활용하는 방식도 추가적으로 더 있음
- 파라미터방식과 필드 방식은 dto 객체의 필드명이 동일해야하고(별칭으로 바꿔줄 수 있음), 컴파일단계에서 오류를 잡아주지 않고 실행할때 오류를 잡아줌.
- 가장 실용적인 방식은 @QueryProjection 방식인데, DTO도 Qtype을 생성해야되고, DTO가 service계층, controller계층까지 이동을 하는애인데, 얘를 querydsl에 의존적이게 만들게 됨으로서 좀 깔끔하지 못하긴 하지.
- 근데 실무에서는 그냥 @QueryProjection 방식 쓰는듯. DTO 객체에 필드들을 다 담은 생성자를 만들어주고 해당 생성자 메소드 위에 @QueryProjection 선언함

#### 동적 쿼리 - BooleanBuilder 사용
- 생략

#### 동적 쿼리 - Where 다중 파라미터 사용
- 생략
  
#### 수정, 삭제 벌크 연산
- 수정
```
long count = queryFactory
          .update(member)
          .set(member.username, "비회원")
          .where(member.age.lt(28))
          .execute();
```
- 삭제
```
long count = queryFactory
            .delete(member)
            .where(member.age.gt(18))
            .execute();
```

#### SQL function 호출하기
- 생략


### 실무 활용 - 순수 JPA와 Querydsl
#### 순수 JPA 리포지토리와 Querydsl
- JPAQueryFactory queryFactory 주입 방법1
  -  리포지토리 선언된 클래스에서.
  ```
  private final EntityManager em;
  private final JPAQueryFactory queryFactory;
  public MemberJpaRepository(EntityManager em) {
      this.em = em;
      this.queryFactory = new JPAQueryFactory(em);
  }
  ```
- JPAQueryFactory queryFactory 주입 방법2
  - 빈으로 등록해놓고, 리포지토리 선언된 클래스에서 @RequiredConstructor
  ```
  @Bean
  JPAQueryFactory jpaQueryFactory(EntityManager em) {
      return new JPAQueryFactory(em);
  }
  ```
- 동시성 문제 : 동시성 문제는 걱정하지 않아도 된다. 왜냐하면 여기서 스프링이 주입해주는 엔티티 매니저는 실제 동 작 시점에 진짜 엔티티 매니저를 찾아주는 프록시용 가짜 엔티티 매니저이다. 이 가짜 엔티티 매니저는 실제 사용 시점에 트랜잭션 단위로 실제 엔티티 매니저(영속성 컨텍스트)를 할당해준다.

#### 동적 쿼리와 성능 최적화 조회 - Builder 사용
- 생략

#### 동적 쿼리와 성능 최적화 조회 - Where절 파라미터 사용
- where 조건에 null 값은 무시된다.
- 메서드를 다른 쿼리에서도 재활용 할 수 있다.
- 쿼리 자체의 가독성이 높아진다.
- 메소드로 뺄 수 있으니까, 여러가지로 조합도 가능
- 동적 쿼리 & Where절 파라미터 사용 & @QueryProjection 방식을 사용한 DTO로 조회의 예시. 이게 핵심!
```
public List<MemberTeamDto> search(MemberSearchCondition condition) {
      return queryFactory
              .select(new QMemberTeamDto(
                      member.id,
                      member.username,
                      member.age,
                      team.id,
                      team.name))
              .from(member)
              .leftJoin(member.team, team)
              .where(usernameEq(condition.getUsername()),
                      teamNameEq(condition.getTeamName()),
                      ageGoe(condition.getAgeGoe()),
                      ageLoe(condition.getAgeLoe()))
              .fetch();
}
private BooleanExpression usernameEq(String username) {
    return isEmpty(username) ? null : member.username.eq(username);
}
private BooleanExpression teamNameEq(String teamName) {
    return isEmpty(teamName) ? null : team.name.eq(teamName);
}
   private BooleanExpression ageGoe(Integer ageGoe) {
      return ageGoe == null ? null : member.age.goe(ageGoe);
}
  private BooleanExpression ageLoe(Integer ageLoe) {
      return ageLoe == null ? null : member.age.loe(ageLoe);
}
```
#### 조회 API 컨트롤러 개발
- @PostConstruct 를 통해서 샘플 데이터들을 db에 저장할때, @Transactional을 같은 위치에 선언할 수 없어서,
- MemberServiceInit static 클래스를 내부에 정의한다음에 memberServiceInit.init(); 을 호출해서 init() 메소드에다가 @Transcational를 선언.
- 내 프로젝트에서는 고려 안했던게 나는 em.persist()를 바로 호출하지 않고 그냥 @Transactional 선언되어있는 서비스계층의 join()을 호출했음
- profile을 나누기
  - application.yml
  ```
  spring:
    profiles:
      active: local
  ```
  - 이렇게 설정을 잡아놓으면 @Profile("local") 이라 선언해놓은 클래스의 스프링 빈이 등록 되고, @Profile("test") 라 선언된 클래스는 빈 등록이 안됨.
  - 이 기능을 사용하여, test에 있는 application.yml에는 active: test 라 명시하면, 실제 애플리케이션을 돌릴떄와 테스트를 돌릴때 다른 빈을 등록하고 안하고를 제어할 수 있어.
  - 사실 내가 원하는건 db 경로를 다르게 주고, 외부환경설정을 주입하고 이런것들에 대해 자세히 알고 싶은건데 이게 "스프링 부트 - 핵심 원리와 활용" 에 나오네..
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














