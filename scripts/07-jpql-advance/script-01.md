# 1. 서브 쿼리와 다형성 쿼리

## I. 서브 쿼리

SQL 에서 서브 쿼리란 이름 그대로 어느 쿼리 속에 존재하는 쿼리를 말한다.

서브 쿼리는 주로 어느 한 요청의 원자성, 지속성을 유지하기 위해 사용되며, 이러한 성격으로 인해 어떤 _"집계"_ 를 수행할 때 많이 사용된다.

JPQL 에서도 서브 쿼리를 이용할 수 있는데, Oracle 문서에 따르면 `WHERE`, `HAVING` Clause 에만 사용할 수 있다. [`[1]`](#reference)

(+ Hibernate 6.x 버전부터는 HQL 로 `FROM` Clause 에도 이용할 수 있다고 한다. [`[2]`](#reference))

다음은 대표적인 JPQL 서브쿼리 예시들이다.

- 평균보다 높은 score 를 가진 엔티티 조회

    ```hql
    SELECT e FROM Entity e where m.score > (
        SELECT AVG(e.score) FROM Entity e
    )
    ```

- 특정 팀에 소속된 Member 를 조회

    ```hql
    SELECT m FROM Member m
    JOIN FETCH m.team      // N + 1 방지용
    WHERE EXISTS (
        SELECT t FROM m.team t WHERE t.name = 'Team 1'
    )
    ```

- N 번 이상 주문한 고객을 조회

    ```hql
    SELECT cu FROM Customer cu where :N <= (
        SELECT COUNT(o.orderedCustomer) FROM Orders o
        WHERE o.orderedCustomer = cu
    )
    ```

참고로 SQL 서브쿼리에서처럼 JPQL 도 `ANY`, `ALL`, `IN`, `EXISTS` 등의 함수를 사용할 수 있다.

---

## II. 다형성 쿼리

이전 [4. 엔티티 연관관계 맺기 고급](../04-establishing-advanced-relationships/README.md) 에서 `@Inheritance` 를 통해 테이블간 상속 관계를 구성하였다.

```java

@Entity
@Inheritance(strategy = InheritanceType.JOINED)
@DiscriminatorColumn(name = "ITEM_TYPE")
class Item {

    @Id
    @GeneratedValue
    public Long id;

    public String name;
    public int price;
}

@Entity
class Book extends Item {

    public String author;
    public String publisher;
}

@Entity
class Electronics extends Item {

    public String madeBy;
    public String model;
    public String serialNumber;
}
```

JPQL 에서 엔티티 다형성 구분을 위한 함수가 존재하는데, `type` 과 `treat` 이다.

```hql
SELECT i FROM Item i
WHERE type (i) = Book
```

```hql
SELECT i FROM Item i
WHERE TREAT (i AS Book).author = 'Samsung' OR 
      TREAT (i AS Electronics).madeBy = 'Samsung'
```

`type` 은 참조된 엔티티의 타입을 제한시키고, `treat` 는 엔티티를 타입 캐스팅하여 사용할 수 있게 해준다.

---

# 2. `@NamedQuery` 와 벌크 연산

## I. `@NamedQuery`

우리가 지금까지 JPQL 을 사용할 때 `String query = ...` 처럼 JPQL 을 문자열로 사용했다.

이러한 방식의 문제는 JPQL 구문이 `String` 이기 때문에 구문이 valid 한지 Run time 시점에 알 수 있다는 점이다.

이를 극복하는 것이 Named query 로, 엔티티 클래스에 `@NamedQuery` 어노테이션을 붙여 사용할 수 있다.

```java

@Entity
@NamedQueries({
        @NamedQuery(
                name = "selectAll",
                query = "SELECT m FROM Member m"
        ),
        @NamedQuery(
                name = "invalidQuery",
                query = "abcdef"
        )
})
public class Member {

    /* ... id 생략 ... */
}

em.

createNamedQuery("selectAll")
        .

getResultList();
```

```
ERROR: HHH90003001: Error in named query: invalidQuery
org.hibernate.query.SyntaxException: At 1:0 and token 'abcdef', no viable alternative at input '*abcdef' [abcdef]
	at org.hibernate.query.hql.internal.StandardHqlTranslator$1.syntaxError(StandardHqlTranslator.java:109)
	...
```

위에서 볼 수 있듯 어플리케이션 시작 시 `invalidQuery` 로 인해 에러가 발생하는 것을 볼 수 있다.

참고로 Named query 를 이용할 때는 `em.createNamedQuery` 메서드를 통해 사용할 수 있다.

또한 Spring-data-jpa 는 `@Query` 어노테이션으로 JPQL 을 작성하는데, 이 `@Query` 또한 Named query 의 일종으로 JPQL 검증이 진행된다.

---

## II. 벌크 연산

HQL 은 `INSERT`, `UPDATE`, `DELETE` JPQL DML 을 지원하며 이를 통해 벌크 연산을 진행할 수 있다.

- 특정 시각 이전에 생성된 엔티티를 모두 삭제

    ```hql
    DELETE FROM TestEntity t WHERE t.createdAt < :time
    ```

- 특정 횟수 이하로 구매된 제품 가격을 N% 할인 조정

    ```hql
    UPDATE Product p
    SET p.price = ((100 - :N) * p.price) / 100
    WHERE p IN (
        SELECT p2 FROM Product p2 WHERE :num <= (
            SELECT COUNT (pr) FROM PurchaseRecord pr WHERE pr.product = p2
        )
    )
    ```

이전 [6. JPQL 기본](../06-jpql-basic/README.md) 에서 언급했듯, `UPDATE`, `DELETE`, `INSERT` JPQL DML 은 `.executeUpdate()` 메서드를 통해 영향된 row 의 개수만 파악할 수 있으며, 이들은 영속 컨텍스트를 무시하고 수행된다.

---

# 3. Join 과 `N + 1` "문제"

SQL 에서 JOIN 은 두 테이블을 엮어 하나의 결과로 만드는 문법으로 RDBMS 의 핵심 중 하나이다.

때문에 JPQL 에서도 `INNER JOIN`, `LEFT JOIN`, `RIGHT JOIN` 을 지원한다.

```hql
SELECT m, t FROM Member m
INNER JOIN m.team t       // INNER 는 생략 가능하다. 
```

```
Hibernate: 
    /* SELECT
        m,
        t 
    FROM
        Member m 
    INNER JOIN
        m.team t  */ select
            m1_0.id,
            m1_0.team_id,
            t1_0.id 
        from
            Member m1_0 
        join
            Team t1_0 
                on t1_0.id=m1_0.team_id
```

Join 자체는 SQL 과 연관이 깊으니 잘 이해되지 않으면 아래 영상을 참고하자.

- [SQL Joins: Difference Between Inner/Left/Right/Outer Joins - JomaClass's YouTube channel](https://www.youtube.com/watch?v=zGSv0VaOtR0&t=2s)

JPQL 에서 Join 을 사용할 때 주의점이 있는데, 흔히 말하는 `N + 1` 문제이다.

Join JPQL 에서 `N + 1` 의 근본적인 원인은 <span style="color:#7898FB">**개발자의 잘못된 Join 사용으로 인해 JPA 가 연관된 엔티티를 영속 컨텍스트에 저장하려 시도** </span> 함에 있다.

다음 예시를 보자.

```java

@Entity
class Member {

    /* ... id 생략 ... */

    @ManyToOne
    public Team team;
}

@Entity
class Team {

    /* ... id 생략 ... */

    public String name;
    public String description;
}
```

```hql
SELECT m FROM Member m
JOIN m.team
```

위 JPQL 을 보고 어딘가 이상하다 느낄 수 있다. 쿼리의 의도는 _"모든 멤버와 그들에 연관된 팀을 조회"_ 함에 있다.

하지만 여기서 묻고 싶다. <span style="color:#7898FB">**연관된 팀을 같이 조회하고 싶어 Join 을 사용했는데, 그럼 도대체 왜 SELECT 구문에는 팀 정보가 선택하지 않았는가?**</span>

위 JPQL 을 실행하면 다음과 같은 실 쿼리를 볼 수 있다.

```
Hibernate: 
    /* SELECT
        m 
    FROM
        Member m 
    JOIN
        m.team  */ select
            m1_0.id,
            m1_0.team_id 
        from
            Member m1_0 
        join
            Team t1_0 
                on t1_0.id=m1_0.team_id
```

쿼리에서 볼 수 있듯, `select m1_0.id, m1_0.team_id ...` 처럼 오직 `Member` 와 관련된 속성만 SELECT column 에 정의되는 것을 볼 수 있다.

이를 실제 쿼리 결과로 확인해보면 다음과 같다.

```sql
SELECT 
    m.id AS MEM_ID, 
    m.team_id AS TEAM_ID
FROM MEMBER AS m 
    JOIN TEAM AS t
        ON t.id = m.team_id;
```

```
+--------+---------+
| MEM_ID | TEAM_ID |
+--------+---------+
|      1 |       1 |
|      2 |       2 |
+--------+---------+
2 rows in set (0.01 sec)
```

여기서 당신은 <span style="color:#7898FB">**위 쿼리 결과"만"으로 `Team1`, `Team2` 정보를 맵핑해 JPA 에 영속시킬 수 있는가?**</span>

쿼리 결과에 `Team` 의 `name`, `description` 가 존재하지 않는데, JPA 가 어떻게 맵핑할 수 있는가? 불가능하다는 것이다!
때문에 JPA 는 연관된 `Team` 들의 정보를 가져오기 위해 아래와 같은 추가적인 쿼리를 발생시킨다.

```
Hibernate: 
    select
        t1_0.id,
        t1_0.description,
        t1_0.name 
    from
        Team t1_0 
    where
        t1_0.id=?
... (연관된 Team 개수만큼 반복)
```

<span style="color:#7898FB">**즉, `N + 1` 문제는 어느 연관된 엔티티를 영속하려던 도중, SQL 쿼리 결과에 엔티티 정보가 부족해 "어쩔수 없이" 추가 쿼리를 발생시켜 나타나는 것이다.**</span>

이러한 원인 때문에 개인적으로 _"`N + 1` 문제"_ 를 "문제"라 부르고 싶지 않을 정도이다.
`N + 1` 문제는 JPA 의 고질적 문제가 아니라 개발자의 오해로 발생하는 문제이기 때문이다.

그래서 이를 해결하는 방법은 아주 간단하다. <span style="color:#7898FB">**연관된 엔티티 정보도 포함해 SELECT column 을 정의하면 되는 것이다.**</span> 

```hql
SELECT m, t FROM Member m
JOIN m.team t
```

```
Hibernate: 
    /* SELECT
        m,
        t 
    FROM
        Member m 
    JOIN
        m.team t  */ select
            m1_0.id,
            m1_0.team_id,
            t1_0.id,
            t1_0.description,
            t1_0.name 
        from
            Member m1_0 
        join
            Team t1_0 
                on t1_0.id=m1_0.team_id
```

위처럼 프로젝션에 `Team` 엔티티를 포함시키면 실 쿼리에 `Team` 정보도 포함시키는 것을 볼 수 있다.

```sql
SELECT 
    m.id AS MEM_ID,
    t.id AS TEAM_ID,
    t.name AS name,
    t.description AS description
FROM MEMBER AS m
    JOIN TEAM AS t
        ON t.id = m.team_id;
```

```
+--------+---------+------+-------------+
| MEM_ID | TEAM_ID | name | description |
+--------+---------+------+-------------+
|      1 |       1 | NULL | NULL        |
|      2 |       2 | NULL | NULL        |
+--------+---------+------+-------------+
2 rows in set (0.00 sec)
```

이를 통해 JPA 는 연관된 `Team` 정보를 쿼리 결과에서 모두 확인할 수 있고, 때문에 추가적인 쿼리가 발생하지 않는다.

---

# 4. Fetch Join

앞서 JPQL 의 Join 과 `N + 1` _문제_ 를 확인하였고, `N + 1` 을 해결하기 위해 연관 엔티티를 프로젝션하라 설명하였다.

그런데 만약 연관된 엔티티가 매우 많을 경우, 연관 엔티티를 모두 투영하기에는 불편한 상황이 일어난다.

```hql
SELECT m, t, i, os FROM Member m
JOIN m.team t
JOIN m.info i
JOIN m.orders os
```

즉, `N + 1` 을 피하기 위해 위처럼 `m, t, i, os` 여러 엔티티를 투영하는게 귀찮을 수 있다.
이를 위한 것이 Fetch Join 이다.

```hql
SELECT m FROM Member m
JOIN FETCH m.team
```

```
Hibernate: 
    /* SELECT
        m 
    FROM
        Member m 
    JOIN
        
    FETCH
        m.team  */ select
            m1_0.id,
            t1_0.id,
            t1_0.description,
            t1_0.name 
        from
            Member m1_0 
        join
            Team t1_0 
                on t1_0.id=m1_0.team_id
```

위 실 쿼리를 `SELECT m, t FROM Member m JOIN m.team t` JPQL 의 실 쿼리와 비교해보면 거의 동일한 것을 볼 수 있다. (뭐 `m.team_id`, `t.id` 를 구분하지 않는 정도? 그런데 이것도 `natural join` 으로 가능)

<span style="color:#7898FB">**즉, Fetch Join 은 연관 엔티티의 속성들을 실 쿼리에 포함시키고, 연관관계 맵핑을 root 엔티티 속에 알아서 설정해주는 "편의 기능" 인 것이다.**</span>

> Fetch Join : `N + 1` 문제의 해결책?
>
> 여러 자료에 JPQL Fetch Join 을 검색하면 대게 _"`N + 1` 의 해결책"_ 이라 설명한다.
>
> 하지만 개인적으로 Fetch Join 은 프로젝션의 편의 기능일 뿐이라 생각한다.
>
> 앞서 이야기했듯 `N + 1` 은 잘못된 JPQL 사용으로 인해 일어나고, Fetch Join 은 Join 할 엔티티를 EAGER FETCH, 연관관계를 root 엔티티에 맺고 프로젝션 결과에서 제외할 뿐이기 때문이다.

따라서 Fetch Join 을 통해 별도의 투영 DTO 를 만들지 않으면서 연관 엔티티 정보를 영속 컨텍스트에 넣을 수 있다.

---

# 5. 번외 : `[1:N]` 관계 페이징

어플리케이션에 따라 다르지만 종종 `[1:N]` 관계에서 페이징을 구현해야 할 때가 존재한다.

```java
@Entity
class Member {

    /* ... id 생략 ... */

    @ManyToOne
    public Team team;
}

@Entity
class Team {

    /* ... id 생략 ... */

    @OneToMany(mappedBy = "team")
    public final List<Member> members = new ArrayList<>();
}
```

위 상황에선 "모든 `Team` 을 조회 (+ 페이징) 하며 `Team` 에 소속된 `Member` 를 조회" 하는 상황일 것이다.

이를 다음과 같은 JPQL 을 이용하면 Hibernate 가 warning 을 뱉는 것을 볼 수 있다.

```java
System.out.println("===========================================");
String query = """
        SELECT t FROM Team t
        LEFT JOIN FETCH t.members
        """;
List<Team> paging = em.createQuery(qq, Team.class)
        .setFirstResult(0)
        .setMaxResults(5)
        .getResultList();
System.out.println("===========================================");

paging.forEach(System.out::println);
```

```
===========================================
Hibernate: 
    /* SELECT
        t 
    FROM
        Team t 
    LEFT JOIN
        
    FETCH
        t.members  */ select
            t1_0.id,
            m1_0.team_id,
            m1_0.id 
        from
            Team t1_0 
        left join
            Member m1_0 
                on t1_0.id=m1_0.team_id
===========================================
Team{id=1}
Team{id=2}
Team{id=3}
Team{id=4}
Team{id=5}
WARN: HHH90003004: firstResult/maxResults specified with collection fetch; applying in memory
```

이는 실 SQL 결과를 집적 확인해 보면 원인을 파악할 수 있다.

```sql
SELECT 
    t.id AS TEAM_ID, 
    m.id AS MEM_ID 
FROM TEAM AS t 
    LEFT JOIN MEMBER AS m 
        ON t.id = m.team_id;
```

```
+---------+--------+
| TEAM_ID | MEM_ID |
+---------+--------+
|       1 |      1 |
|       1 |      2 |
|       2 |      3 |
|       3 |   NULL |
|       4 |   NULL |
|       5 |   NULL |
|       6 |   NULL |
+---------+--------+
7 rows in set (0.00 sec)
```

위 결과를 보면 `Member1, 2` 모두 `Team1` 에 속해있음을 알 수 있다. 그런데 우리는 페이징 API 로 `첫 5 번째 Team` 까지의 결과만 반환할 것을 요구했다.

즉, `Team1 ~ 5` 까지의 결과만 반환되어야 한다.
그럼 여기서 질문하겠다. <span style="color:#7898FB">**당신은 실제 DB 내용을 보지 않고 쿼리 결과를 어디까지 잘라내야 하는지 예상할 수 있는가?**</span>

그렇다. 이는 실 쿼리를 날려보지 않는 한 알 수 없는 사실이고, 때문에 <span style="color:#7898FB">**JPA 는 페이징 처리를 하기 위해 쿼리 결과를 메모리에 임시 저장한다.**</span>

따라서 콘솔 출력을 보면 `firstResult/maxResults specified with collection fetch; applying in memory` 처럼 메모리에 저장함음 warning 함을 알 수 있다.

지금이야 `Team` 개수가 그리 많지 않지 않아 상관 없지만, 이가 늘어날 수록 _OOM, Out Of Memory_ 위험성이 증가한다.

따라서 `[1:N]` 관계에서 페이징 처리를 할 땐 다음 처럼 2 단계로 구분해 처리하는 것이 안전하다.

- I. 페이징 결과에 포함할 `Team` (`One` 관계 엔티티) PK 를 조회

    ```java
    List<Long> targetIds = em.createQuery("SELECT t.id FROM Team t", Long.class)
            .setFirstResult(0)
            .setMaxResults(5)
            .getResultList();
    ```

- II. PK 값들에 해당하는 `Team` (+ `Member` Fetch Join) 정보를 조회

    ```java
    List<Team> paging = em.createQuery("""
                    SELECT t FROM Team t
                    LEFT JOIN FETCH t.members
                    WHERE t.id IN :targetIds
                    """, Team.class)
            .setParameter("targetIds", targetIds)
            .getResultList();
    ```

```
===========================================
Hibernate:              // [I]
    /* SELECT
        t.id 
    FROM
        Team t */ select
            t1_0.id 
        from
            Team t1_0 
        limit
            ?, ?
Hibernate:              // [II]
    /* SELECT
        t 
    FROM
        Team t 
    LEFT JOIN
        
    FETCH
        t.members 
    WHERE
        t.id IN :targetIds  */ select
            t1_0.id,
            m1_0.team_id,
            m1_0.id 
        from
            Team t1_0 
        left join
            Member m1_0 
                on t1_0.id=m1_0.team_id 
        where
            t1_0.id in (?, ?, ?, ?, ?)
===========================================
Team{id=1}
Team{id=2}
Team{id=3}
Team{id=4}
Team{id=5}
```

---

## Reference

- [Kodo™ 4.2.0 Developers Guide for JPA/JDO - Oracle documentation](https://docs.oracle.com/html/E13946_04/index.html)
    - [10.2. JPQL Language Reference](https://docs.oracle.com/html/E13946_04/ejb3_langref.html)
        - [10.2.5.15. JPQL Subqueries](https://docs.oracle.com/html/E13946_04/ejb3_langref.html#ejb3_langref_subqueries)
            - `[1]` : Subqueries may be used in the `WHERE` or `HAVING` clause.
- `[2]` : [Hot features of Hibernate ORM 6.1 - Hibernate Blog](https://in.relation.to/2022/06/24/hibernate-orm-61-features/)

---
