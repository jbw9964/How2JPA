# 1. 연관관계 어노테이션 : `@OneToOne`, `@OneToMany`, `@ManyToOne`

DB 테이블에서 연관관계는 4 가지로 분류할 수 있다.

- One to One `[1:1]`
- One to Many `[1:N]`
- Many to One `[N:1]`
- Many to Many `[N:M]`

JPA 도 이에 맞춰 `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany` 어노테이션을 제공하고, 이들을 활용해 단방향 또는 양방향 참조를 이뤄낼 수 있다.

(`[N:M]` 관계는 지양하는 관계이므로 생략하겠다.)

여기서 다시 한번 주의하자. DB 와 ORM 의 세상은 다르다. DB 에서 `[1:1]` 등의 관계를 JPA 는 단방향 참조로 구성할 수 있고 양방향으로도 만들 수 있다.

즉, `@OneToMany` 등의 **어노테이션을 <span style="color:#7898FB">단방향 참조에 사용하는 경우</span> 와 <span style="color:#7898FB">양방향 참조에 사용하는 경우</span> 를 <span style="color:#7898FB">구별해 생각</span>** 하라는 것이다.

---

# 2. `@JoinColumn`

자 그럼 본격적으로 연관관계를 맺는 법을 알아보자.

먼저 엔티티간 연관관계를 맺기 위해선 실제 DB 에 연관관계가 존재해야 한다.

그 DB 속 테이블간 연관관계를 JPA 에게 알려주는 어노테이션이 `@JoinColumn` 이다.

```java

@Entity
class Team {

    private Long id;    // @Id, @GeneratedValue 생략 
    // 오직 id column 만 존재
}

@Entity
class Member {

    private Long id;    // @Id, @GeneratedValue 생략

    @ManyToOne
    @JoinColumn(name = "TEAM_ID")
    private Team team;
}
```

```
Hibernate: 
    create table Member (
       id bigint not null,
        TEAM_ID bigint,
        primary key (id)
    )
Hibernate: 
    create table Team (
       id bigint not null,
        primary key (id)
    )
Hibernate: 
    alter table Member 
       add constraint FKl7wsny760hjy6x19kqnduasbm 
       foreign key (TEAM_ID) 
       references Team
```

- `@JoinColumn` 주요 속성

|                             이름                             |    타입     | 설명                                                                                                                      |            default            |
|:----------------------------------------------------------:|:---------:|-------------------------------------------------------------------------------------------------------------------------|:-----------------------------:|
|                     `name` (Optional)                      | `String`  | 생성할 FK column 명. <span style="color:#7898FB">**FK 가 어느 테이블에 생성될 지는 상황에 따라 다름**</span>                                   | `[필드 이름]_[추론된 연관 엔티티의 PK 이름]` |
|             `referencedColumnName` (Optional)              | `String`  | FK column 을 reference 할 column 명. **단방향 `OneToMany` 사용 시 reference column 은 자기 자신에게 있다 가정하며, 그 외는 target 엔티티에 있다 가정함.** |     Reference 된 테이블의 PK 명     |
| `unique`, `nullable`, `insertable`, `updatable` (Optional) | `boolean` | `@Column` 어노테이션의 내용과 동일                                                                                                 |    `unique` 제외하고 모두 `true`    |

위 설명 중 `name` 에 주목하자.

`@JoinColumn` 은 테이블간 연관관계 정보를 JPA 에 제공한다 하였다. 이는 즉, **두 테이블이 존재할 때 어느 테이블에 FK 를 저장하는지 알려** 주는 것과 같다.

그래서 `@JoinColumn` `name` 속성의 주석을 보면 FK 가 어느 테이블에 생성되는지 자세히 설명하고 있다.
그 주석 내용을 정리하면 다음과 같다.

- <span style="color:#7898FB">**A. OneToOne, ManyToOne** 사용시</span>

  FK column 은 <span style="color:#7898FB">**자기 자신 (source entity)**</span> 에 존재한다.

- <span style="color:#7898FB">**B. 단방향 OneToMany 사용시**</span>

  FK column 은 <span style="color:#7898FB">**상대 (target entity)**</span> 에 존재한다.

- <span style="color:#7898FB">**+ C. `@JoinColumn` 이 필드에 생략되었을 시** [`[1]`](#reference)</span>

  <span style="color:#7898FB">**연관관계의 Owner**</span> table 에 FK column 이 존재한다.

즉, 단방향, 양방향, 어노테이션에 따라 <span style="color:#7898FB">**FK column 이 생성되는 위치가 달라질 수 있다**</span> 는 것이고, 이를 확실히 인지해 사용해야 한다.

더불어 이전 [2.3 단방향 연관관계 VS 양방향 연관관계](../02-concepts-of-relationship-in-jpa/script-01.md#3-단방향-연관관계-vs-양방향-연관관계) 에서 단방향 참조시 해당 엔티티 자체가 주인이라 설명하였다. 때문에 단방향 참조라도 `C. @JoinColumn 이 필드에 생략된 경우` 가 적용될 수 있음에 유의하자.

---

# 3. 단방향 연관관계 맺기

앞서 `@JoinColumn` 으로 실제 DB 에 연관관계를 맺어줄 수 있음을 확인하였다.

이제 ORM 으로 넘어와 엔티티간 연관관계를 맺어보자.

단방향 연관관계는 사실 아주 쉽다. `@JoinColumn` 과 다중성을 생각해 어노테이션만 붙이면 끝나기 때문이다.

---

## I. 단방향 `@OneToOne` & `@ManyToOne`

단방향 `[1:1]`, `[N:1]` 관계는 개인적으로 맺기 가장 쉬운 관계라고 생각한다. `@JoinColumn` 을 생략해도 default 가 적용되 우리가 생각한 그대로 적용되기 때문이다.

```java
@Entity
class Member {

    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    public Team team;

    @OneToOne
    private MemberInfo info;
}

@Entity
class Team {

  @Id @GeneratedValue
  private Long id;
}

@Entity
class MemberInfo {

  @Id @GeneratedValue
  private Long id;
}
```

```
Hibernate: 
    create table Member (
       id bigint not null,
        info_id bigint,
        team_id bigint,
        primary key (id)
    )
Hibernate: 
    create table MemberInfo (
       id bigint not null,
        primary key (id)
    )
Hibernate: 
    create table Team (
       id bigint not null,
        primary key (id)
    )
Hibernate: 
    alter table Member 
       add constraint FKm2hhj51w89w3u4mea5tws3bmb 
       foreign key (info_id) 
       references MemberInfo
Hibernate: 
    alter table Member 
       add constraint FK5nt1mnqvskefwe0nj9yjm4eav 
       foreign key (team_id) 
       references Team
```

위 DDL 을 보면 `Member` 테이블에 `info_id`, `team_id` 가 생성된 것을 볼 수 있다.

`@JoinColumn` 이 생략되 있어도 `Member - Team`, `Member - MemberInfo` 관계의 주인은 `Member` 이므로, `Member` 테이블에 FK column 이 추가된 것이다.

물론 `@JoinColumn(name = "TEAM_ID")` 처럼 추가해 column 이름을 변경할 수 있다.

암튼 엄청 쉽다.

---

## II. 단방향 `@OneToMany`

단방향 `[1:N]` 관계는 이전과 살짝 다르다. 아래 예시의 DDL 에 주목하자.

```java
@Entity
class Member {

  @Id
  @GeneratedValue
  @Column(name = "MEMBER_ID")
  private Long id;
}

// @JoinColumn 사용한 경우
@Entity
class Used {

  @Id @GeneratedValue
  private Long id;

  @OneToMany
  @JoinColumn(name = "MEMBER_ID")
  private List<Member> members;
}

// @JoinColumn 생략한 경우
@Entity
class Omitted {

  @Id @GeneratedValue
  private Long id;

  @OneToMany
  private List<Member> members;
}
```

```
// create
Hibernate: 
    create table Member (
       MEMBER_ID bigint not null,
        primary key (MEMBER_ID)
    )
Hibernate: 
    create table Omitted (
       id bigint not null,
        primary key (id)
    )
Hibernate: 
    create table Omitted_Member (     // <---- 이건 뭘까아요?
       Omitted_id bigint not null,
        members_MEMBER_ID bigint not null
    )
Hibernate: 
    create table Used (
       id bigint not null,
        primary key (id)
    )

// 제약조건 생성
Hibernate: 
    alter table Omitted_Member 
       add constraint UK_l920b1bmeonlw19hhb1bfsbvc unique (members_MEMBER_ID)
Hibernate: 
    alter table Member 
       add constraint FKh30ceydrhx03prjkkter3dgh6 
       foreign key (MEMBER_ID) 
       references Used
Hibernate: 
    alter table Omitted_Member 
       add constraint FK3ihr5uf3q14urum5kgar0vaff 
       foreign key (members_MEMBER_ID) 
       references Member
Hibernate: 
    alter table Omitted_Member 
       add constraint FKqsxyfbqp2e8a5j5759gjxcqvl 
       foreign key (Omitted_id) 
       references Omitted
```

DDL 을 보면 `Omitted_Member` 라는 알 수 없는 테이블이 생성되었다. 이는 `Omitted` 엔티티에 `@JoinColumn` 을 사용하지 않아 생성된 _"조인용 테이블"_ 이다.

아래의 내용은 Hibernate 문서에서 발취한 내용이다. [`[2]`](#reference), [`[3]`](#reference)

> 단방향 `[1:N]` 관계시 _"FK column 의 주도권"_ 을 owner 가 갖는 것은 흔치 않으며 권장하지 않습니다.
> 
> 때문에 join table 전략으로 단방향 `[1:N]` 관계를 해결하길 강력히 권장합니다.
> 
> ...
> 
> 단방향 `[1:N]` 관계에서 `@JoinColumn` 으로 physical mapping 정보를 제공하지 않으면 기본적으로 join table 전략을 사용합니다.

`[1:N]` 또는 `[N:1]` 관계시 DB 에서 FK 는 반드시 `N` 테이블에 존재할 수 밖에 없다.

때문에 만약 우리가 단방향 `[1:N]` 참고관계를 맺더라도 결국 FK 는 `N` 쪽, 상대방 테이블에 생성될 수밖에 없다.

**여기서 문제가 발생한다.** DB 세상에서는 몰라도 ORM 세상에서는 연관관계의 주인을 통해서만 관계를 update 할 수 있다.

그런데 FK 는 물리적으로 상대 엔티티에 속할 수 밖에 없으므로, **관계의 주인은 자신인데 그에 필요한 물리적 실체는 하인에게 있는 것이다!**

오! 말로만 들어도 무언가 이념적으로 맞지 않는 걸 알 수 있다.
거기다 이를 현실까지 끌고오면 _"자신 (하인 엔티티 테이블) 의 데이터가 누군지도 모르는 남 (주인 엔티티 테이블) 에 의해 수정되는, 원인을 추적하기 어려운 쿼리를 발생"_ 시킨다.

(ORM 에서는 `주인 - 하인` 관계가 중요할지 몰라도, DB 에서는 그렇지 않음에 주의하자.)

이러한 이유 때문에 문서에서 조차 단방향 `[1:N]` 관계를 지양하고 차라리 join table 전략을 사용하라 말하고 있다.

우리도 이에 맞춰 <span style="color:#7898FB">**단방향 `[1:N]` 관계가 정말로 필요한지 고민하고, `[N:1]` 관계로 해결할 수 있지 않을지 충분히 생각하자.**</span>

---

# 4. 양방향 연관관계 맺기

자 그럼 이제 양방향 연관관계에 대해 알아보자.

이전 [2.2 연관관계의 주인](../02-concepts-of-relationship-in-jpa/script-01.md#2-연관관계의-주인) 을 통해 양방향 엔티티 참조시, **둘 중 어느 쪽이 연관관계의 주인인지 명시** 해야 한다 언급하였다.

이를 위한 것이 **`mappedBy`** 이다. <span style="color:#7898FB">`mappedBy` 는 양방향 연관관계에서 **주인을 명시** 하는데 사용되며, **`@???ToOne` 어노테이션에만 존재한다.**</span>

어? 왜 `@ManyToMany`, `@OneToMany` 에는 없을까?
사실 생각해보면 매우 명료하다.

DB 입장에서 어느 테이블이 `N` 관계로 놓이기 위해선 (앞서 설명했듯) 해당 테이블에 FK column 이 있을 수 밖에 없다.
어? 그러면 애초에 `N` 관계에 놓인 엔티티를 주인으로 설정하면 되지 않는가? DB 와 ORM 의 괴리가 존재하지 않는 명확한 관계가 아닌가?

그렇다. 아주 절호의 찬스인 것이다!

때문에 `@ManyToMany`, `@OneToMany` 에는 `mappedBy` 속성이 없으며, 이를 반대로 생각해보면 양방향 관계 맵핑도 그리 어렵지 않음을 알 수 있다.

만약 아직도 `@???ToMany` 에 `mappedBy` 가 없는지 잘 와닿지 않는다면 아래 좋은 질문 글이 있으니 참고하자.
- [@ManyToOne 에는 왜 mappedBy 속성이 없을까요? - Inflearn 강의 질문](https://www.inflearn.com/community/questions/18042/manytoone-%EC%97%90%EB%8A%94-%EC%99%9C-mappedby-%EC%86%8D%EC%84%B1%EC%9D%B4-%EC%97%86%EC%9D%84%EA%B9%8C%EC%9A%94?srsltid=AfmBOopZTaLvRT0x4nUywuPIi3F2YsPsdNnUAYORN3s1Ajbfd2_6vRVU)

---

## I. 양방향 `@OneToOne`

먼저 `[1:1]` 관계를 양방향 연관관계로 구성해보자.

```java
@Entity
public class Member {

  @Id @GeneratedValue
  private Long id;

  @OneToOne
  private MemberInfo info;
}

@Entity
class MemberInfo {

    @Id @GeneratedValue
    private Long id;

    @OneToOne(mappedBy = "info")
    private Member mem;
}
```

기본적으로 <span style="color:#7898FB">**`mappedBy = ...` 에 명시하는 이름은 주인 엔티티 속 _"연관 객체의 필드 이름"_ 을 제시**</span> 해야 한다. 때문에 `@OneToOne(mappedBy = "info")` 처럼 `Member.info` 필드를 적어준다.

더불어 `@JoinColumn` 을 생략해도 우리가 생각한 대로 DDL 이 만들어지는데, 이는 앞서 [2. `@JoinColumn`](#2-joincolumn) 에서 설명하였으니 기억나지 않으면 참고하자.

```
Hibernate: 
    create table Member (
       id bigint not null,
        info_id bigint,
        primary key (id)
    )
Hibernate: 
    create table MemberInfo (
       id bigint not null,
        primary key (id)
    )
Hibernate: 
    alter table Member 
       add constraint FKm2hhj51w89w3u4mea5tws3bmb 
       foreign key (info_id) 
       references MemberInfo
```

(사실 `@OneToOne.mappedBy` 주석에 적혀있지만) 만약 `Member.info` 에도 `@OneToOne(mappedBy = "mem")` 를 적으면 아래와 같은 에러가 발생한다.

```
Exception in thread "main" org.hibernate.AnnotationException: Unknown mappedBy in: scripts.entities.MemberInfo.mem, referenced property unknown: scripts.entities.Member.info
	at org.hibernate.cfg.OneToOneSecondPass.doSecondPass(OneToOneSecondPass.java:171)
	...
```

---

## II. `[1:N]` 관계에서 양방향 맵핑

다음으로 `[1:N]` 관계를 양방향으로 맺어보자.

사실 생각해보면 정말 단순하다. 애초에 `@ManyToOne` 에 `mappedBy` 속성이 없기 때문이다.

```java
@Entity
class Member {

    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    public Team team;
}

@Entity
class Team {

  @Id @GeneratedValue
  private Long id;

  @OneToMany(mappedBy = "team")
  public final List<Member> members = new ArrayList<>();
}
```

`@OneToMany` 사용시 JPA 가 연관 엔티티를 담아줄 `Collection` 을 제공해줘야 한다.
위 예시처럼 굳이 `final` 인 필요는 없지만 그렇다고 `final` 이어서 나쁠 건 없다.

```
Hibernate: 
    create table Member (
       id bigint not null,
        team_id bigint,
        primary key (id)
    )
Hibernate: 
    create table Team (
       id bigint not null,
        primary key (id)
    )
Hibernate: 
    alter table Member 
       add constraint FK5nt1mnqvskefwe0nj9yjm4eav 
       foreign key (team_id) 
       references Team
```

DDL 을 보면 `@JoinColumn` 의 기본 행동으로 `Member` 테이블에 `team_id` column 이 추가되었고, 제약 조건이 설정된 것을 볼 수 있다.

이 때 만약 <span style="color:#7898FB">어느 `Member - Team` **관계를 수정** 하고 싶다면 아래처럼 연관관계 **주인, `Member` 를 통해서 수정** 할 수 있는 점을 유의하자.</span>

```java
// 허용됨      :   Member 를 통해서 수정
member.team = someTeam;

// 허용 X     :   Team 을 통해서 수정
// 이렇게 진행해도 어차피 영속 컨텍스트가 인식 못해서 ISNERT 쿼리 안나감.
team.member.add(someMember);
```

이 연관관계 수정이 잘 이해되지 않는다면 이전 [2.2 연관관계의 주인](../02-concepts-of-relationship-in-jpa/script-01.md#2-연관관계의-주인) 를 참고하자.

---

## III. 만약 `mappedBy` 를 생략하면 어떻게 될까?

좋다. 이제 왠만한 상황에서도 양방향 연관관계를 맺을 수 있게 되었다.

그런데 갑자기 의문이 들 수 있다. 만약 양쪽 모두 `mappedBy` 를 생략하면 어떻게 될까?

결론만 말하자면 <span style="color:#7898FB">**_"서로 독립적인 단방향 관계로 해석"_**</span> 되어 작동한다. [`[4]`](#reference) 아래 예시의 DDL 을 보자.

```java
@Entity
class Member {

    @Id @GeneratedValue
    private Long id;

    @ManyToOne
    public Team team;
}

@Entity
class Team {

  @Id @GeneratedValue
  private Long id;

  @OneToMany
  public final List<Member> members = new ArrayList<>();
}
```

```
// create
Hibernate: 
    create table Member (
       id bigint not null,
        team_id bigint,
        primary key (id)
    )
Hibernate: 
    create table Team (
       id bigint not null,
        primary key (id)
    )
Hibernate: 
    create table Team_Member (    // <---- 이건 뭘까아요?
       Team_id bigint not null,
        members_id bigint not null
    )

// 제약조건 생성
Hibernate: 
    alter table Team_Member 
       add constraint UK_s2bdqsiefihvxghf2fujxn2eg unique (members_id)
Hibernate: 
    alter table Member 
       add constraint FK5nt1mnqvskefwe0nj9yjm4eav 
       foreign key (team_id) 
       references Team
Hibernate: 
    alter table Team_Member 
       add constraint FKbh0ppoyhmwjgigc2gs7lr1kcf 
       foreign key (members_id) 
       references Member
Hibernate: 
    alter table Team_Member 
       add constraint FK267gfjokmdychsnik3u313q5j 
       foreign key (Team_id) 
       references Team
```

이전 [3.II 단방향 `@OneToMany`](#ii-단방향-onetomany) 의 내용을 상기해보자. `@OneToMany` 에서 `@JoinColumn` 이 생략되면 join table 전략으로 인해 _"조인용 테이블"_ 이 생성된다 하였다.

이에 더불어 `Member` 테이블을 보면 `team_id` column 이 존재하는 것을 볼 수 있다.

그렇다. <span style="color:#7898FB">**`Member -> Team`, `Team -> Member` 연관관계가 독립적으로 존재** 하는 것이다!</span> 

참 오묘하지 않은가? `mappedBy` 하나만 생략했을 뿐인데 이런일이 일어나다니.

이를 `@OneToOne` 에 적용해보면 더 가관이다.

```java
@Entity
class Member {

  @Id @GeneratedValue
  private Long id;

  @OneToOne
  private MemberInfo info;
}

@Entity
class MemberInfo {

  @Id @GeneratedValue
  private Long id;

  @OneToOne
  private Member mem;
}
```

```
Hibernate: 
    create table Member (
       id bigint not null,
        info_id bigint,
        primary key (id)
    )
Hibernate: 
    create table MemberInfo (
       id bigint not null,
        mem_id bigint,
        primary key (id)
    )

Hibernate: 
    alter table Member 
       add constraint FKm2hhj51w89w3u4mea5tws3bmb 
       foreign key (info_id) 
       references MemberInfo
Hibernate: 
    alter table MemberInfo 
       add constraint FK7gwfswvvtk8u9rnoxjkihkn0n 
       foreign key (mem_id) 
       references Member
```

위처럼 `mappedBy` 를 생략하면 `Member.info`, `MemberInfo.mem` 둘 중 어느 곳을 통해서도 관계를 수정할 수 있게 되버린다.

오! 말로만 들어도 끔찍하지 않은가? 
얼마나 많은 버그를 만들어낼지 두려워진다.

---

# 5. 연관관계 매핑시 고려사항 3 가지

이제 우리는 실 DB 에 존재하는 연관관계를 객체 수준으로 구성할 수 있게 되었다.

하지만 객체 참조 연관관계를 맺을때는 다음 사항을 **반드시 고려하여 신중해 맺어** 주도록 하자.

- I. 엔티티 연관관계의 다중성 (multiplicity)
  
  실 DB 에서의 연관관계를 고려해 `@OneToMany`, `@OneToOne` 을 제대로 붙여주자.

- II. 단방향 VS 양방향

  단방향 연관관계를 지향하고 양방향은 반드시 필요할 때만 사용하자.

- III. 연관관계의 주인

  양방향 연관관계시 관계의 주인을 통해서만 관계를 수정할 수 있음에 유의하자.

특히 `III. 연관관계의 주인` 을 명심하고 어플리케이션의 비즈니스 로직도 이를 고려해 작성하도록 하자.

---

## Reference

- [Hibernate Community Documentation - 2.2.5. Mapping entity associations/relationships](https://docs.jboss.org/hibernate/annotations/3.5/reference/en/html_single/#entity-mapping-association)

  - 2.2.5.1. One-to-one
      - `[1]` : If no `@JoinColumn` is declared on the owner side, the defaults apply. A join column(s) will be created in the owner table and its name will be the concatenation of the name of the relationship in the owner side, _ (underscore), and the name of the primary key column(s) in the owned side. In this example passport_id because the property name is passport and the column id of Passport is id.

  - 2.2.5.3.1.2. Unidirectional
    - `[2]` : A unidirectional one to many using a foreign key column in the owned entity is not that common and not really recommended. We strongly advise you to use a join table for this kind of association (as explained in the next section). This kind of association is described through a @JoinColumn
  
  - 2.2.5.3.1.4. Defaults
    - `[3]` : Without describing any physical mapping, a unidirectional one to many with join table is used. The table name is the concatenation of the owner table name, _, and the other side table name.

- [what if we do not specify mappedBy attribute in OneToMany annotation? - StackOverflow](https://stackoverflow.com/questions/24584411/what-if-we-do-not-specify-mappedby-attribute-in-onetomany-annotation)
  - `[4]` : Because if you don't specify `mappedBy`, you're not saying that the `OneToMany` between `Item` and `Bid` and the `ManyToOne` between `Bid` and `Item` are actually the two sides of a unique bidirectional association.
  So Hibernate considers that they are two, different, unidirectional associations. And since the default mapping for a `OneToMany` association is to use a join table, that's what Hibernate uses.

---

