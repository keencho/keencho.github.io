---
title: Hibernate, JPA 에서 발생하는 Cross Join 문제 해결하기
author: keencho
date: 2022-07-20 20:12:00 +0900
categories: [Hibernate]
tags: [JPA, Hibernate]
---

# **Hibernate, JPA 에서 발생하는 Cross Join 문제 해결하기**

## **개요**
평화롭게 코딩하던 어느날, 쿼리 결과가 의도한대로 리턴되지 않는 문제가 발생하였습니다. 쿼리에 문제가 있나 싶어 자세히 살펴봤지만 (queryDSL을 사용중이었습니다.) 자바 코드상의 문제는 찾을 수 없었습니다.  

데이터베이스에 날라가는 sql문을 보고 나서야 문제를 찾을수 있었고 이 포스팅에서는 그 문제와 해결 과정을 적어보려고 합니다.  

## **Cross Join 발생**  
문제는 cross join 발생한다는 것이었습니다. 당연히 left join이 발생할 것이라고 생각하였지만 cross join이 발생하여 결과가 의도한대로 리턴되지 않은 것이지요.  

cross join은 두 테이블에서 행의 모든 조합을 반환하는 join입니다. 하이버네이트가 쿼리를 생성하는 과정에 where 조건절에 cross join과 관련된 조건이 추가되어 원하는 결과가 넘어오지 않았던 것이지요.  

하이버네이트는 join을 명시적으로 지정하지 않으면 암묵적으로 cross join으로 지정하는 경향(?) 이 었다고 합니다. 하이버네이트의 창시자인 `Gavin King`, `Christian Bauer`의 책인 `Hibernate in Action` 에는 이런 내용이 있습니다.  

> Implicit joins are always directed along many-to-one or one-to-one association, never through a collection-valued association.  

암묵적 join은 항상 many-to-one 혹은 one-to-one 연관성에 따라 결정되며 컬렉션 값에 따라 지정되지 않는다고 합니다.  

또한 대부분의 데이터베이스 엔진들은 cross join에 대해 where 조건을 추가하기 때문에 결과적으로 cross join / where 에 의해 원하는 결과값이 나오지 않게 되는 것입니다.  

## **문제 재구현**  
jpa / querydsl을 사용해 문제를 재구현 해보겠습니다.   

'일'을 위한 테이블이 있습니다. 이 일은 개발자나 영업사원에게 배정될 수 있습니다. 개발자나 영업사원은 한 회사에 속해 있어야 합니다.  

```java
@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@Table(name = "COMPANY")
public class Company {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;

    public Company(String name) {
        this.name = name;
    }
}

@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@Table(name = "DEV_TEAM_MEMBERS")
public class DevTeamMember {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "COMPANY_ID", foreignKey = @ForeignKey(name = "DEV_TEAM_MEMBERS___COMPANY"))
    private Company company;

    public DevTeamMember(String name, Company company) {
        this.name = name;
        this.company = company;
    }
}

@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@Table(name = "SALES_TEAM_MEMBERS")
public class SalesTeamMembers {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;

    @ManyToOne(optional = false, fetch = FetchType.LAZY)
    @JoinColumn(name = "COMPANY_ID", foreignKey = @ForeignKey(name = "FK_SALES_TEAM_MEMBERS___COMPANY"))
    private Company company;

    public SalesTeamMembers(String name, Company company) {
        this.name = name;
        this.company = company;
    }
}

@Entity
@Data
@AllArgsConstructor
@NoArgsConstructor
@Table(name = "JOB")
public class Job {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;

    private String name;

    private LocalDateTime dtCreatedAt = LocalDateTime.now();

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "DEV_TEAM_MEMBERS_ID", foreignKey = @ForeignKey(name = "FK_JOB___DEV_TEAM_MEMBERS"))
    private DevTeamMember devTeamMembers;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "SALES_TEAM_MEMBERS_ID", foreignKey = @ForeignKey(name = "FK_JOB___SALES_TEAM_MEMBERS"))
    private SalesTeamMembers salesTeamMembers;

    public Job(String name, DevTeamMember devTeamMembers, SalesTeamMembers salesTeamMembers) {
        if (devTeamMembers == null && salesTeamMembers == null) {
            throw new RuntimeException("Work must be assigned to one person.");
        }
        this.name = name;
        this.devTeamMembers = devTeamMembers;
        this.salesTeamMembers = salesTeamMembers;
    }
}
```  

순서대로 회사, 개발자, 영업사원, 일 순서입니다. 테이블을 만들었으니 repository들을 각각 생성합니다.  

repository를 생성하였다면 test코드를 작성해 보겠습니다. 일단 더미데이터 생성 코드입니다.  

```java
@BeforeEach
void initDummyData() {
    var kakao = companyRepository.save(new Company("카카오"));
    var naver = companyRepository.save(new Company("네이버"));

    var kakao_dev_1 = devTeamMembersRepository.save(new DevTeamMember("카카오_개발자_1", kakao));
    var kakao_dev_2 = devTeamMembersRepository.save(new DevTeamMember("카카오_개발자_2", kakao));

    var naver_dev_1 = devTeamMembersRepository.save(new DevTeamMember("네이버_개발자_1", naver));
    var naver_dev_2 = devTeamMembersRepository.save(new DevTeamMember("네이버_개발자_2", naver));

    var kakao_sales_1 = salesTeamMembersRepository.save(new SalesTeamMembers("카카오_영업사원_1", kakao));
    var kakao_sales_2 = salesTeamMembersRepository.save(new SalesTeamMembers("카카오_영업사원_2", kakao));

    var naver_sales_1 = salesTeamMembersRepository.save(new SalesTeamMembers("네이버_영업사원_1", naver));
    var naver_sales_2 = salesTeamMembersRepository.save(new SalesTeamMembers("네이버_영업사원_2", naver));

    jobRepository.save(new Job("AWS EC2 인스턴스 생성", kakao_dev_1, null));
    jobRepository.save(new Job("AWS RDS 인스턴스 생성", kakao_dev_2, null));
    jobRepository.save(new Job("Spring Boot 프로젝트 생성", naver_dev_1, null));
    jobRepository.save(new Job("React 프로젝트 생성", naver_dev_2, null));

    jobRepository.save(new Job("매출목표 설정", null, kakao_sales_1));
    jobRepository.save(new Job("외근", null, kakao_sales_2));
    jobRepository.save(new Job("제품 소개서 만들기", null, naver_sales_1));
    jobRepository.save(new Job("월말 수금", null, naver_sales_2));
}
```  

다음은 테스트 코드입니다.

```java
@Test
void test() {

    var q = QJob.job;

    var r = jpaQueryFactory
            .select(
                    q.id,
                    new CaseBuilder()
                            .when(q.devTeamMembers.isNotNull()).then(q.devTeamMembers.name)
                            .otherwise(q.salesTeamMembers.name),
                    new CaseBuilder()
                            .when(q.devTeamMembers.isNotNull()).then(q.devTeamMembers.company.name)
                            .otherwise(q.salesTeamMembers.company.name)
            )
            .from(q)
            .fetch();

    Assert.isTrue(r.size() == 8, "예상된 결과와 다릅니다.");
}
```  

job을 8개 생성하였기 때문에 위 테스트는 정상적으로 통과해야 합니다. 그러나 쿼리 결과 list의 size가 0이되어 테스트에 실패합니다. 

```sql
select job0_.id                                                                                          as col_0_0_,
       case when job0_.dev_team_members_id is not null then devteammem1_.name else salesteamm2_.name end as col_1_0_,
       case when job0_.dev_team_members_id is not null then company4_.name else company6_.name end       as col_2_0_
from job job0_
         cross join dev_team_members devteammem1_
         cross join company company4_
         cross join sales_team_members salesteamm2_
         cross join company company6_
where job0_.dev_team_members_id = devteammem1_.id
  and devteammem1_.company_id = company4_.id
  and job0_.sales_team_members_id = salesteamm2_.id
  and salesteamm2_.company_id = company6_.id
```  

생성된 sql 문입니다. 당연히 left join이 걸릴줄 알았으나 cross join이 걸렸습니다. 또한 where 조건이 추가되어 원하는 결과가 나오지 않았습니다.  

## **문제 해결**
문제 해결은 간단합니다. join을 명시적으로 지정해주면 됩니다.  

```java  
@Test
void test() {
    var q = QJob.job;

    var r = jpaQueryFactory
            .select(
                    q.id,
                    new CaseBuilder()
                            .when(q.devTeamMembers.isNotNull()).then(q.devTeamMembers.name)
                            .otherwise(q.salesTeamMembers.name),
                    new CaseBuilder()
                            .when(q.devTeamMembers.isNotNull()).then(q.devTeamMembers.company.name)
                            .otherwise(q.salesTeamMembers.company.name)
            )
            .from(q)
            .leftJoin(q.devTeamMembers).leftJoin(q.devTeamMembers.company)
            .leftJoin(q.salesTeamMembers).leftJoin(q.salesTeamMembers.company)
            .fetch();

    Assert.isTrue(r.size() == 8, "예상된 결과와 다릅니다.");
}
```  

join을 명시적으로 지정하고 테스트를 수행하니 정상적으로 통과합니다!

```sql
select job0_.id                                                                                          as col_0_0_,
       case when job0_.dev_team_members_id is not null then devteammem2_.name else salesteamm5_.name end as col_1_0_,
       case when job0_.dev_team_members_id is not null then company3_.name else company6_.name end       as col_2_0_
from job job0_
         left outer join dev_team_members devteammem1_ on job0_.dev_team_members_id = devteammem1_.id
         left outer join dev_team_members devteammem2_ on job0_.dev_team_members_id = devteammem2_.id
         left outer join company company3_ on devteammem2_.company_id = company3_.id
         left outer join sales_team_members salesteamm4_ on job0_.sales_team_members_id = salesteamm4_.id
         left outer join sales_team_members salesteamm5_ on job0_.sales_team_members_id = salesteamm5_.id
         left outer join company company6_ on salesteamm5_.company_id = company6_.id;
```  

생성된 sql문을 보면 의도한대로 join이 걸린 것을 확인할 수 있습니다.  

## **결론**  
join을 명시적으로 지정하지 않으면 cross join이 걸려 원하는 결과를 얻을수 없는 문제가 발생할 수 있습니다. 이를 피하기 위한 설정(?) 같은게 있으면 좋겠지만 그런 설정은 아직까지 없는 것으로 보입니다.  

위 문제는 join을 명시적으로 지정하여 문제를 쉽게 해결할 수 있으니 문제 해결에 도움이 되면 좋겠습니다.
