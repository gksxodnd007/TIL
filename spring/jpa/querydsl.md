## QueryDsl이란?
- JPQL의 빌더(Criteria)클래스

## QueryDsl 사용전 설정
- dependency 추가
```groovy
dependencies {
  compile("com.querydsl:querydsl-core:4.2.1")
  compile("com.querydsl:querydsl-apt:4.2.1")
  compile("com.querydsl:querydsl-jpa:4.2.1")
  compile("com.querydsl:querydsl-collections:4.2.1")
  ...
}
```

- Q클래스를 먼저 생성 후 컴파일 되어야 하므로 task를 먼저 실행시켜야함
```groovy
def queryDslOutput = file("src-gen/main/java")

sourceSets {
    main {
        java {
            srcDir "src-gen/main/java"
        }
    }
}

clean {
    delete queryDslOutput
}

task generateQueryDSL(type: JavaCompile, group: "build") {
    doFirst {
        if (!queryDslOutput.exists()) {
            logger.info("Creating `$queryDslOutput` directory")

            if (!queryDslOutput.mkdirs()) {
                throw new InvalidUserDataException("Unable to create `$queryDslOutput` directory")
            }
        }
    }

    source = sourceSets.main.java
    classpath = configurations.compile
    options.compilerArgs = [
            "-proc:only",
            "-processor",
            "com.querydsl.apt.jpa.JPAAnnotationProcessor"
    ]
    destinationDir = queryDslOutput
    options.failOnError = false
}

compileJava {
    dependsOn generateQueryDSL //이 부분 dependsOn이 generateQueryDSL이다.
    source generateQueryDSL.destinationDir
}
```

## QueryDsl 사용법

- example entity(다음 두 entity는 1:N관계)
  - User
  - Account

- User.java
```java
@Entity
@Table(name = "tb_user")
@Getter @Setter
@EqualsAndHashCode
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @JsonIgnore
    private Long id;

    @Column(name = "name", nullable = false)
    private String name;

    @Column(name = "age", nullable = false)
    private Long age;

}
```

- Account.java
```java
@Entity
@Table(name = "tb_account")
@Getter @Setter
public class Account {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @JsonIgnore
    private Long id;

    @ManyToOne(fetch = FetchType.EAGER)
    @JoinColumn(name = "user_id", referencedColumnName = "id")
    @JsonIgnore
    private User userId;

    @Column(name = "account")
    private String account_num;

    @Column(name = "money")
    private Double money;

    @Column(name = "bank_name")
    private String bankName;

    @PrePersist
    public void initMoney() {
        this.money = 0D;
    }

}
```

- UserRepository.java
```
@Repository
public interface UserRepository extends JpaRepository<User, Long>, CustomUserRepository {

    Optional<User> findByName(String name);

}
```

- CustomUserRepository.java
```java
public interface CustomUserRepository {

    List<User> findAllUserByAgeLimit(Integer limitCount);
    List<String> findAllNameByAgeLimit(Integer limitCount);
    List<AccountUserJoinDto> findAllUserGreaterThanAge(Long age);

}
```

- UserRepositoryImpl.java
```java
@NoRepositoryBean
public class UserRepositoryImpl extends QuerydslRepositorySupport implements CustomUserRepository {

    //QuerydslRepositorySupport 클래스에는 기본생성자가 없음.
    public UserRepositoryImpl() {
        super(User.class);
    }

    @Override
    public List<User> findAllUserByAgeLimit(Integer limitCount) {
        QUser user = QUser.user;
        return from(user)
                .where(user.age.eq(25L))
                .limit(limitCount)
                .fetch();
    }

    @Override
    public List<String> findAllNameByAgeLimit(Integer limitCount) {
        QUser user = QUser.user;
        return from(user)
                .where(user.age.eq(25L))
                .limit(limitCount)
                .select(user.name)
                .fetch();
    }

    @Override
    public List<AccountUserJoinDto> findAllUserGreaterThanAge(Long age) {
        QUser user = QUser.user;
        QAccount account = QAccount.account;

        JPQLQuery<AccountUserJoinDto> query = from(account)
                .innerJoin(account.userId, user)
                .select(Projections.constructor(AccountUserJoinDto.class,
                        user.name,
                        user.age,
                        account.money,
                        account.bankName))
                .where(user.age.gt(age));

        return query.fetch();
    }
}
```

findAllUserGreaterThanAge 함수 호출시 다음과 같은 쿼리가 생성되어 DB에 전달됨.
다음 innerJoin함수를 보면 감이 오겠지만 account 변수의 위치에 주의하자.
```java
from(account).innerJoin(account.userId, user)...
```
```sql
select user1_.name as col_0_0_, user1_.age as col_1_0_, account0_.money as col_2_0_, account0_.bank_name as col_3_0_
from tb_account account0_
inner join tb_user user1_
on account0_.user_id=user1_.id
where user1_.age>?
```
