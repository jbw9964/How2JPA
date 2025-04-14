<!-- how2jpa-logo-origin.png -->

<p align="center">
    <img src="assets/how2jpa-logo-origin.png" width="80%" height="80%" style="display: block; margin-left: auto; margin-right: auto;">
</p>

[자바 ORM 표준 JPA 프로그래밍 - 기본편, 김영한](https://www.inflearn.com/course/ORM-JPA-Basic) 강의를 듣고 <span style="color:#7898FB">**JPA 기초 사용법**</span> 을 정리한 Repo

---

### [1. JPA 의 Entity](./scripts/01-entities/README.md)

### [2. 연관관계의 개념](scripts/02-concepts-of-relationship-in-jpa/README.md)

### [3. 엔티티 연관관계 맺기 기본](scripts/03-establishing-basic-relationships/README.md)

### [4. 엔티티 연관관계 맺기 고급](scripts/04-establishing-advanced-relationships/README.md)

### [5. JPA 에서의 타입](scripts/05-types-in-jpa/README.md)

### [6. JPQL 기본](scripts/06-jpql-basic/README.md)

### [7. JPQL 고급](scripts/07-jpql-advance/README.md)

### [8. 기타 주요 개념](./scripts/08-extras)

---

### H2 & MySQL 셋업

[db-docker-compose.yml](./db-docker-compose.yml) 로 container 설정 변경 가능.

|          H2 관련 설정          |                             값                             |
|:--------------------------:|:---------------------------------------------------------:|
|          사용 image          | [`oscarfonts/h2`](https://hub.docker.com/r/oscarfonts/h2) |
|        Container 이름        |                       `h2-database`                       |
|     H2 shell port bind     |                        `1521:1521`                        |
| H2 web interface port bind |                          `81:81`                          |
|          기본 User           |                           `sa`                            |
|         기본 User PW         |                            없음                             |

|      MySQL 관련 설정      |                          값                           |
|:---------------------:|:----------------------------------------------------:|
|       사용 image        | [`mysql`](https://hub.docker.com/_/mysql) (Official) |
|     Container 이름      |                   `mysql-database`                   |
| MySQL shell port bind |                     `3306:3306`                      |
|        기본 User        |                        `root`                        |
|      기본 User PW       |                        `root`                        |

- DB container 띄우기

```bash
docker-compose -f ./db-docker-compose.yml up -d
```

- H2 shell 접속

```bash
docker exec -it h2-database h2cli
```

- MySQL shell 접속

```bash
docker exec -it mysql-database mysql -u root -p
```

```
Enter password: [root]
```

---
