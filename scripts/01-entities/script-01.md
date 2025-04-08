# 1. 엔티티와 ORM

_Entity_ 영단어를 그대로 해석하면 _"존재자"_ 로, _"그 자체서 실존하는 무언가"_ 이다.

이 영문 해석과 유사하게, JPA 에서 `Entity` 는 `"데이터베이스의 정보를 객체로 끌어내 사용하는 무언가"` 라 통용된다. [`[1]`](#reference)


즉, JPA 에서 `Entity` 는 `ORM` 의 이념 그대로 <span style="color:#7898FB">**데이터베이스 내 존재를 객체로 구체화한 것**</span> 이라 생각할 수 있다.

`ORM` 은 `DB를 객체로 구체화해 관리` 하는 기술이념이다.

여기서 재미있는 점은 Spring 프레임웍처럼 **역할을 위임** 한다는 점이다.

Spring 은 객체를 Spring Context 에 넣어 필요에 따라 생성, 호출, 관리 등을 진행한다.

JPA 에서도 이와 유사한 **Persistence Context** 가 있다.
Spring 에서처럼 `Entity` 를 Persistence Context 에 제공해 SQL 쿼리 생성을 위임한다.

<span style="color:#7898FB">따라서 JPA 의 Persistence Context 는 `ORM` 을 이해하기 위해 필수적인 존재이며, **나의 행동(코드)으로 인해 context 가 어떻게 변화하고 작동할지 예측** 할 수 있어야 버그를 줄일 수 있다.</span>

---

# 2. `@Entity` & `@Table`

`@Entity` 는 Spring 의 `@Bean` 과 유사한 기능을 한다.

JPA 가 `DB를 객체로 구체화해 관리` 하기 위해선 먼저 `DB 가 어떻게 이루어져 있고, 어떻게 객체로 구체화` 할지 알려줘야 한다.

이를 위한 어노테이션이 `@Entity` 와 `@Table` 로, 이 둘 모두 `javax.persistence` 패키지에 있다.

```java

import javax.persistence.*;

@Entity(name = "Entity_name")
@Table(
        name = "TABLE_NAME",
        uniqueConstraints = {@UniqueConstraint(/* ... */)},
        indexes = {@Index(/* ... */)}
)
class TestEntity {
    /* ... */
}
```

- 어노테이션 주 속성

    - `@Entity`

  |        이름         |    타입    | 설명                                                           |  default   |
  |:-----------------:|:--------:|--------------------------------------------------------------|:----------:|
  | `name` (Optional) | `String` | 엔티티 이름, jpql 등의 쿼리에 사용할 때 이용된다. 이 때 이름이 jpql 의 예약어가 아니어야 한다. |  `클래스 이름`  |

    - `@Table`

  |               이름               |          타입          | 설명                                                                            |  default   |
    |:------------------------------:|:--------------------:|-------------------------------------------------------------------------------|:----------:|
  |       `name` (Optional)        |       `String`       | 엔티티와 맵핑될 DB 테이블 이름                                                            |  `클래스 이름`  |
  | `uniqueConstraints` (Optional) | `UniqueConstraint[]` | DB 테이블에 적용될 유니크 제약조건. JPA 에 의해 테이블이 생성될 때에만 적용됨. 제약조건은 엔티티 필드 제약 조건 적용 후 적용됨. |   `None`   |
  |      `indexes` (Optional)      |      `Index[]`       | 테이블에 적용될 `index`. JPA 에 의해 테이블이 생성될 때에만 적용됨.                                  |   `None`   |

만약 엔티티 클래스에 `@Table` 이 없을 시, `@Table` 속성의 기본값들이 적용되어 `엔티티-DB` 를 맵핑한다.

---

# 3. `@Id` & `@GeneratedValue`

DB 에서 테이블을 생성할 때 `PK` 가 없는 상태로 만들수도 있다.
하지만 이를 생각해보면 `식별자가 없는 데이터` 를 저장하는 것으로, `추적이 불필요한 데이터를 저장하는 테이블` 을 생성하는 것이라 생각할 수 있다.

<span style="color:#7898FB">즉, `PK` 가 없는 테이블은 **_"엔티티"_** 로 적합하지 않다는 것이다.</span>
때문에 JPA 의 모든 엔티티는 객체의 어떤 속성이 `PK` 에 해당하는지 정의해야 한다.

이를 위한 어노테이션이 `@Id` 와 `@GeneratedValue` 이다.

```java
import javax.persistence.*;

@Entity
public class TestEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
}
```

`@Id` 는 엔티티의 `PK field` 를 지정하는 어노테이션으로, 다음과 같은 타입의 속성에 붙여야 유효하다.

- Java primitive types 와 그들의 wrapper 들
- `String`, `java.util.Date`, `java.sql.Date`
- `java.math.BigDecimal`, `java.math.BigInteger`

`@GeneratedValue` 는 `PK 생성 전략` 을 명시하는 어노테이션으로, `TABLE`, `SEQUENCE`, `IDENTITY`, `AUTO` 전략이 존재한다.

각 전략을 간단히 정리하면 다음과 같다.

|     전략     |                   설명                   |       지원하는 RDBMS        |
|:----------:|:--------------------------------------:|:-----------------------:|
| `IDENTITY` | DB 의 `AUTO_INCREMENT` 를 이용해 `PK` 를 설정  | MySQL, PostgreSQL, H2 등 |
| `SEQUENCE` | DB 의 `Sequence Object` 를 이용해 `PK` 를 설정 |      Oracle, H2 등       |
|  `TABLE`   |      `PK` 생성을 위한 별도의 테이블을 만들어 이용       |           ALL           |
|   `AUTO`   |   JPA 가 DB dialect 를 통해 전략을 자동으로 선택    |           ALL           |

결과만 줄여 말하면 결국 `IDENTITY`, `SEQUENCE` 둘 중 DBMS 에 맞춰 사용한다.
사실 각 전략별 큰 주의점이 있는데, 관련된 내용은 [`@GeneratedValue` 세부 내용](../08-extras/01-generated-value-in-detail.md) 에 적어놓았다.

---

# 4. 필드 어노테이션

앞서 테이블의 `PK` 를 설정하는 방법을 보았다. 이번에는 테이블의 column 을 정의하는 방법이다.

---

## I. `@Column`

<span style="color:#7898FB">`@Column` 은 엔티티와 맵핑될 DB 의 column 을 속성을 명시하는 어노테이션이다.</span>
만약 엔티티 속성 (필드) 에 `@Column` 이 붙지 않으면 `@Column` 의 기본값들로 엔티티 필드가 적용된다.

즉, 엔티티 내 모든 속성에 `@Column` 을 붙이지 않아도 DB column 이 맵핑된다.

`@Column` 어노테이션은 다음과 같은 주 속성이 존재한다.

|              이름               |    타입     | 설명                                                                                    |              default              |
|:-----------------------------:|:---------:|---------------------------------------------------------------------------------------|:---------------------------------:|
|       `name` (Optional)       | `String`  | 맵핑될 DB column 의 이름                                                                    |              `필드 이름`              |
|      `unique` (Optional)      | `boolean` | 해당 column 에 유니크 제약조건을 생성할지 여부                                                         |              `false`              |
|     `nullable` (Optional)     | `boolean` | 해당 column 의 nullable 여부                                                               |              `true`               |
|    `insertable` (Optional)    | `boolean` | 해당 필드를 SQL INSERT 문 생성시 포함할지 여부                                                       |              `true`               |
|    `updatable` (Optional)     | `boolean` | 해당 필드를 SQL UPDATE 문 생성시 포함할지 여부                                                       |              `true`               |
| `columnDefinition` (Optional) | `String`  | DDL 생성 시 사용할 column 정의                                                                | 필드 타입에서 추론된 SQL DDL 파편 (fragment) |
|      `table` (Optional)       | `String`  | 해당 column 을 갖는 Secondary table 의 이름. 부재시 해당 필드가 Primary table (해당 엔티티 자체) 에 속한 것으로 간주 |              `None`               |
|      `length` (Optional)      |   `int`   | 해당 column 의 길이. 오직 `String` 관련 column 일 때에만 적용됨.                                      |               `255`               |

이 중 `columnDefinition`, `table` 에 대해서만 설명하겠다.

- `columnDefinition`

잘 알다시피 JPA 는 엔티티를 파악해 DDL 을 생성해 줄 수 있다. `(hibernate.ddl-auto)`
`columnDefinition` 은 DDL 을 생성할 때 column 정의를 직접 명시할 때 사용한다.

아래처럼 `columnDefinition` 을 정말 이상하게 만들고 생성되는 DDL 을 보자.

```java
@Entity
public class Test {

    @Id @GeneratedValue
    @Column(columnDefinition = "BIGINT default 0 hello world!!!!!")
    private Long id;
}
```
```
Hibernate: 
    create table Test (
       id BIGINT default 0 hello world!!!!! not null,
        primary key (id)
    )
```

출력을 보면 `BIGINT default 0 hello world!!!!!` 가 그대로 column 정의에 들어간 것을 볼 수 있다.

- `table`

`table` 은 엔티티의 해당 속성을 다른 테이블로 구성해 저장할 수 있게 해준다.
이를 올바르게 사용하려면 엔티티에 `@SecondaryTable` 어노테이션을 같이 사용해야 한다.

```java
@Entity
@SecondaryTable(
        name = "SECOND_VALUE_TABLE",
        pkJoinColumns = {
                @PrimaryKeyJoinColumn(name = "firstId"),
                @PrimaryKeyJoinColumn(name = "secondId")
        }
)
public class Test implements Serializable {

    @Id @GeneratedValue
    private Long firstId;

    @Id @GeneratedValue
    private Long secondId;

    @Column(table = "SECOND_VALUE_TABLE")
    private int secondValue;
}
```

참고로 위 예시는 `@SecondaryTable(pkJoinColumns = ... )` 을 어떻게 구성하는지 보여주기 위해 어거지로 만든 예제이다.

`firstId`, `secondId` 처럼 `@Id` 를 2 개로 만들바에 `@EmbeddedId` 등을 사용하는게 더 깔끔해 보일 것이다.

(+ `implements Serializable` 은 Composite-id 사용하기 위해 필수적이라 낑겨 넣었다.)

아무튼 위처럼 구성하고 실행되는 DDL 을 보면 아래와 같다.

```
Hibernate: 
    create table SECOND_VALUE_TABLE (
       secondValue integer,
        firstId bigint not null,
        secondId bigint not null,
        primary key (firstId, secondId)
    )
Hibernate: 
    create table Test (
       secondId bigint not null,
        firstId bigint not null,
        primary key (secondId, firstId)
    )
Hibernate: 
    alter table SECOND_VALUE_TABLE 
       add constraint FKc469gdobdnyxgwsueajx75cyg 
       foreign key (firstId, secondId) 
       references Test
```

새로운 테이블 `SECOND_VALUE_TABLE` 에 `Test.secondValue` 가 저장되고, `SECOND_VALUE_TABLE` 테이블이 `Test` 테이블과 `1:1` 식별관계로 alter 되는 것을 볼 수 있다.

이 때 그나마 주의할 점은 `SECOND_VALUE_TABLE` 에 해당하는 엔티티는 없다는 것이다.

앞서 [3. `@Id` & `@GeneratedValue`](#3-id--generatedvalue) 에서 `"PK 가 없는 테이블은 "엔티티" 로 적합하지 않다"` 고 서술했다.
그래서 엄밀하게 따지면 이 `"1:1 식별관계"` 가 DB 상 존재하는 것은 맞지만, ORM 의 입장에서는 유의미하지 않다. (`SECOND_VALUE_TABLE` 테이블에 해당하는 _"엔티티"_ 가 존재하지 않기 때문.)

때문에 `@SecondaryTable` 가 `1:1 식별관계` 를 DB 에 "구현" 해주는 것은 맞지만, 실제 JPA 에서 이 `1:1 식별관계` 를 활용하기에는 "의도가 엇나가있다" 고 생각하는 것이 맞아 보인다. 

---

## II. `@Temporal`

<span style="color:#7898FB">`@Temporal` 은 날짜타입을 맵핑</span>할 때 사용되어 3 가지 타입이 존재한다.

```java
@Entity
public class Test {

    @Id @GeneratedValue
    private Long firstId;
  
    @Temporal(TemporalType.DATE)
    private Date date1;
    private LocalDate date2;
  
    @Temporal(TemporalType.TIME)
    private Date time;
  
    @Temporal(TemporalType.TIMESTAMP)
    private Date dateTime1;
    private LocalDateTime dateTime2;
}
```

```
Hibernate: 
    create table Test (
       firstId bigint not null,
        date1 date,
        date2 date,
        dateTime1 timestamp,
        dateTime2 timestamp,
        time time,
        primary key (firstId)
    )
```

참고로 `LocalDateTime`, `LocalDate` 타입을 사용하면 Hibernate 가 인식해 `@Temporal` 을 생략할 수 있다.

---

## III. `@Enumerated`

<span style="color:#7898FB">`@Enumerated` 은 `Enum` 타입을 맵핑</span>하기 위한 어노테이션으로, `ORDINAL`, `STRING` 중 선택할 수 있다.

이 때 `ORDINAL` 은 사용하면 아주 위험하므로 `STRING` 예시만 보자.
왜 위험한지는 검색해보자.

```java
enum EnumTest {}

@Entity
class Test {

    @Id @GeneratedValue
    private Long firstId;
  
    @Enumerated(EnumType.STRING)
    private EnumTest type;
}
```

```
Hibernate: 
    create table Test (
       firstId bigint not null,
        type varchar(255),
        primary key (firstId)
    )
```

DDL 을 보면 column 이 String-like 한 varchar 로 정의되는 것을 볼 수 있다.
<span style="color:#7898FB">DB 의 실제 `ENUM` 이 아닌 것이다.</span>


그런데 생각해보면 이 방법이 적어도 실제 데이터가 없어질 일은 없으니까 훨씬 안전해 보인다. 

---

## IV. `@Lob`

<span style="color:#7898FB">_LOB_ 는 _Large OBject_</span> 의 약어로, 이름 그대로 대용량 개체를 뜻한다.

이는 크게 _BLOB, Binary Large OBject_, _CLOB, Character Large OBject_ 로 나뉜다.

따라서 JPA 는 엔티티 필드의 타입에 따라 column 을 BLOB-like 또는 CLOB-like 속성으로 정의해 맵핑하고, 이를 위한 어노테이션이 `@Lob` 이다.

`CLOB` 는 `String`, `char[]`, `java.sql.CLOB` 타입에 맵핑되고, `BLOB` 는 `byte[]`, `java.sql.BLOB` 타입과 맵핑된다.

```java
@Entity
public class Test {

    @Id @GeneratedValue
    private Long firstId;

    @Lob
    private String veryLongText;

    @Lob
    private byte[] veryLargeBytes;
}
```

```
# H2 Dialect
Hibernate: 
    create table Test (
       firstId bigint not null,
        veryLargeBytes blob,
        veryLongText clob,
        primary key (firstId)
    )
```

참고로 DBMS 마다 _BLOB_, _CLOB_ 에 해당하는 타입 명이 다를 수 있다. MySQL 경우 _BLOB_ 는 동일하지만 _CLOB_ 를 _TEXT_ 로 칭한다.

그래서 인터넷 예제를 보면 MySQL 에서 `@Column(columnDefinition = "TEXT")` 처럼 사용하는 예시가 많은데, 이 중 어떤 방식이 좋은지는 잘 모르겠다.

더 찾아보니 `@Lob` 와 `TEXT` 관련해서 깊이 실험한 글이 있었다.
- [JPA/Hibernate에서 @Column(columnDefinition = "TEXT") 사용하지 않고 MySQL TEXT 타입 설정하기](https://limvik.github.io/posts/mysql-set-text-type-in-jpa-not-using-column-columndefinition-text/)

글에 의하면 `@Column` 의 `length` 를 조절해 `text`, `longtext` 등이 지정되도록 유도할 수 있다되어 있다.
그런데 MySQL 에서 실험해보니 길이와 상관 없이 모두 `longtext` 로 설정되는 것을 확인하였다.

아마 Hibernate 버전에 따라 개선된 거지 않을까 싶다.

---

## V. `@Transient`

<span style="color:#7898FB">_Transient_ 를 직역하면 _"과도기", "일시적인 사물"_</span> 이다.

그래서 <span style="color:#7898FB">`@Transient` 는 해당 필드를 DB column 과 맵핑하지 않도록 명시</span> 하는데 사용된다.

```java
@Entity
public class Test {

    @Id @GeneratedValue
    private Long firstId;

    @Transient
    private String onlyInMemory;
}
```

```
Hibernate: 
    create table Test (
       firstId bigint not null,
        primary key (firstId)
    )
```

이를 어느곳에 잘 사용할 수 있을까 해서 검색했는데, 상당히 좋은 글 2 개를 발견하였다.
- [JPA에서 @Transient 애노테이션이 존재하는 이유](https://gmoon92.github.io/jpa/2019/09/29/what-is-the-transient-annotation-used-for-in-jpa.html)
- [[JPA] @Transient와 @Transient? (Spring Data와 JPA의 관계에 대해서)](https://dogfood.tistory.com/entry/JPA-Transient%EC%99%80-Transient-Spring-Data%EC%99%80-JPA%EC%9D%98-%EA%B4%80%EA%B3%84%EC%97%90-%EB%8C%80%ED%95%B4%EC%84%9C)

심지어 2 번째 글은 JPA 의 `@Transient` 와 Spring-data-jpa 의 `@Transient` 가 완전히 다른 역할을 하는 것도 알려준다. 

더 자세한 내용을 알고 싶으면 검색해서 알아보자.

---

## Reference

- [What's the difference between entity and class?](https://stackoverflow.com/questions/2550197/whats-the-difference-between-entity-and-class)
    - `[1]` : Entities are usually used to establish a mapping between an object and to a table in the database.

---
