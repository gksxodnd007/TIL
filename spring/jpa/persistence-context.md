## JPA 기본

### EntityManagerFactory & EntityManager
- EntityManagerFactory: EntityManager를 만드는 공장
  - Datasource하나당 하나의 EntityManagerFactory를 만들어 사용
  - EntityManagerFactory를 생성하는 작업은 비용이 크기 때문에 싱글톤으로 만들어 사용
  - Thread-safe
```java
EntityManagerFactory emf = Persistence.createEntityManagerFactory("kakaopay");
```

- EntityManager: entity를 생성, 수정, 삭제, 조회하는 등 entity와 관련된 일들을 처리
  - EntityManagerFactory를 이용하여 EntityManager를 생성하는 비용은 거의 들지 않음.
  - Thread-unsafe (쓰레드간 공유시 문제 발생)
  - EntityManager는 데이터베이스 연결이 꼭 필요한 시점까지 커넥션을 얻지 않음.
  - 데이터를 변경하려면 항상 트랜잭션 안에서 작업해야함.(트랜잭션이 없을 경우 예외 발생)

### 영속성 컨텍스트
- 논리적인 개념으로 EntityManager를 생성시 하나 만들어짐.
> 여러 EntityManager가 같은 영속성 컨텍스트에 접근할 수도 있다.
- 영속성 컨텍스트와 식별자 값
  - 영속성 컨텍스트는 entity를 식별자(@Id로 테이블의 pk와 매핑한 값) 값으로 구분한다.
    - IDENTITY전략과 최적화
      - IDENTITY전략은 데이터를 DB에 INSERT한 후에 기본 키값을 조회할 수 있다. 따라서 entity에 식별자 값을 할당하려면 추가로 DB를 조회해야 한다. JDBC3에 추가된 Statement.getGeneratedKeys()를 사용하면 데이터를 저장하면서 동시에 생성된 기본 키 값도 얻어 올 수 있다. 이를 통해 하이버네이트는 한 번만 통신한다.
      - entity가 영속 상태가 되려면 식별자가 반드시 필요하다. 하지만 IDENTITY식별자 생성 전략은 에티티를 DB에 저장해야 식별자를 구할 수 있으므로 em.persist()를 호출하는 즉시 INSERT SQL이 DB에 전달된다. 따라서 이 전략은 트랜잭션을 지원하는 쓰기 지연이 동작하지 않는다.
  - 영속 상태는 식별자 값이 반드시 있어야 한다. (식별자 값이 없으면 예외 발생)
- 영속성 컨텍스트와 데이터베이스 저장
  - JPA는 보통 트랜잭션을 커밋하는 순간 영속성 컨텍스트에 새로 저장된 entity를 저장한다.(flush()호출 시 저장, flush후에도 영속성 컨텍스트에서 객체가 지워지지 않는다.)
- 영속성 컨텍스트를 flush하는 방법
  - em.flush()를 직접 호출
  - 트랜잭션 커밋 시 flush()가 자동 호출
  - JPQL 쿼리 실행 시 flush()가 자동 호출

### Entity의 생명주기
- 비영속: 영속성 컨텍스트와 전혀 관계가 없는 상태
- 영속: 영속성 컨텍스트에 저장된 상태(em.find()나 JPQL을 사용해서 조회한 entity도 영속 상태임)
- 준영속: 영속성 컨텍스트에 저장되었다가 분리된 상태
- 삭제: 삭제된 상태

## JPA 심화

### Entity들 사이의 관계 설정
- 다음과 같은 관계가 있다고 가정하자.
```java
@Entity
public class User {

  @Id
  private Long id;

  @Column(name = "name");
  private String name;

  @Column(name = "age")
  private Long age;

  @ManyToOne(fetch = FetchType.EAGER)
  @JoinColumn(name = "account_id")
  private Account account;
}

@Entity
public class Account {

  @Id
  private Long id;

  @Column(name = "account_number")
  private Long accountNumber;

  @Column(name = "balance")
  private Double balance;
}
```
- User Entity의 연관관계를 살펴보면 Account와 다대일 관계임을 확인할 수 있다. 이때, FetchType이 Eager이면 User entity를 조회할 때마다 Account 정보도 즉시 join해서 데이터를 가져온다. 하지만 이는 비효율적이다. User데이터만 필요한데 Account정보까지 가져오게되면 자원 낭비 이다. 따라서 FetchType은 항상 Lazy로딩을 통해 데이터 로딩을 지연시켜야한다. 필요할 때만 JPQL을 이용하여 join하여야한다.
- Lazy 로딩일 때 주의해야 할 점
  - N + 1 문제발생
  - N + 1 문제해결
    - fetch join을 이용
    - projections를 통해 dto로 데이터를 가져올 경우 fetch join 하게 됨.
