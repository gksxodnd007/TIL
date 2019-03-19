## JPA에서의 Isolation With Lock
- JPA는 데이터베이스 트랜잭션 격리 수준을 READ COMMITTED 정도로 가정한다. 일부 로직에 더 높은 격리 수준이 필요하면 낙관적 락과 비관적 중 하나를 사용하면 된다.

### Optimistic Lock(낙관적 락)
- 버전정보를 이용하여 업데이트를 처리하는 방법
- 버전정보로 사용할 컬럼에 @Version 애너테이션을 부여해야함
- optimistic locking is based on detecting changes on entities by checking their version attribute
```sql
UPDATE {TABLE_NAME}
SET
  {COL_A} = ?,
  VERSION = ? (버전 + 1 증가)
WHERE
  ID = ?
  AND VERSION = ? (버전 비교)
```
- 위의 쿼리를 실행시켰을 때 변경된 row수가 0이면 다른 사용자가 데이터를 변경한 것으로 판단하여 Exception을 발생시킨다.
- **Mode**
  - NONE : 락 옵션을 적용하지 않아도 엔티티에 @Version이 적용된 필드만 있으면 낙관적 락이 적용된다.
  - OPTIMISTIC : @Version만 적용했을 경우 엔티티를 수정해야 버전을 체크하지만 이 옵션을 추가하면 에티티를 조회만 해도 버전을 체크한다. 즉, 한 번 조회한 엔티티는 트랜잭션을 종료할 때까지 다른 트랜잭션에서 변경하지 않음을 보장한다.
  - OPTIMISTIC_FORCE_INCREMENT : 낙관적 락을 사용하면서 버전 정보를 강제로 증가시킨다.
  - READ : it’s a synonym for OPTIMISTIC (JPA 1.0 호환 기능)
  - WRITE : it’s a synonym for OPTIMISTIC_FORCE_INCREMENT (JPA 1.0 호환 기능)
- **Exception**
  - **OptimisticLockException** : 버전 정보가 맞지않았을 경우 예외


### Pessimistic Lock(비관적 락)
- 트랜잭션의 충돌이 발생한다고 가정하고 우선 락을 걸고 보는 방법
- 트랜잭션안에서 서비스로직이 진행되어야함.
- pessimistic locking mechanism involves locking entities on the database level.
- **Type**
  - Exclusive Lock : in order to modify or delete the reserved data, we need to have an exclusive lock
  - Shared Lock : read but not write in data when someone else holds a shared lock
- **Mode**
  - PESSIMISTIC_READ : shared lock을 얻을 수 있다. 데이터를 반복 읽기만 하고 수정하지 않는 용도로 락을 걸 때 사용. 일반적으로 잘 사용하지 않음. 데이터베이스 대부분은 방언에 의해 PESSIMISTIC_WRITE로 동작
  - PESSIMISTIC_WRITE : 데이터베이스에 쓰기 락을 걸 때 사용.
  - PESSIMISTIC_FORCE_INCREMENT : 버전 정보를 사용. 버전 정보를 강제로 증가시킴. 하이버네이트는 nowait를 지원하는 데이터베이스에 대해서 for update nowait옵션을 적용.
- **Lock Scope**
  - PessimisticLockScope.EXTENDED : 연관된 entity들도 lock이 걸림
  - PessimisticLockScope.NORMAL : 해당 entity만 lock이 걸림
- **Exception**
  - **PessimisticLockException** : indicates that obtaining a lock or converting a shared to exclusive lock fails and results in a transaction-level rollback(락을 얻어오지 못했을 경우 발생하는 예외)
  - **LockTimeoutException** : indicates that obtaining a lock or converting a shared lock to exclusive times out and results in a statement-level rollback(락을 기다리다 설정해놓은 wait time을 지났을 경우 발생하는 예외)
  - **PersistanceException** : indicates that a persistence problem occurred. PersistanceException and its subtypes, except NoResultException, NonUniqueResultException, LockTimeoutException, and QueryTimeoutException, marks the active transaction to be rolled back.

```sql
insert into department values ( ... ); # statement
insert into employees values ( ... ); # statement
```
- transaction-level rollback : 위의 쿼리 모두 롤백
- statement-level rollback : insert employees에러가 났을 경우 해당 query만 롤백 department statement는 정상 처리

### JPA에서 락을 구현한 방법

- 각 DBMS의 query로 락을 거는 법은 다르다. 이 글에서는 MySQL기준으로 설명한다.

**LOCK IN SHARE MODE**
- LOCK IN SHARE MODE는 SELECT를 한 후에 트랜잭션이 끝날 때까지 해당 ROW 값이 변경되지 않을 것을 보장한다. 바꿔 말하면 해당 ROW를 UPDATE 하거나 DELETE 하려는 쿼리는 잠김 상태가 되어 트랜잭션이 끝날 때까지 대기하게 된다. 하지만, SELECT는 얼마든지 여러 세션이 동시에 수행하는 것이 가능하다. 기존 SELECT 쿼리문 맨 뒤에 LOCK IN SHARE MODE 문장을 추가하는 것만으로 사용이 가능하지만, 트랜잭션이 끝나기 전까지만 유효하므로 auto_commit을 꺼야 한다.
```sql
SELECT balance FROM tb_account  WHERE id = 1 LOCK IN SHARE MODE; # 5.7
SELECT balance FROM tb_account  WHERE id = 1 FOR SHARE; # 8.0
# 8.0 버전에서는 하위 호환성을 위해 LOCK IN SHARE MODE도 동작을 합니다. 하지만 FOR SHARE를 이용하였을 때 추가 옵션을 줄수 있습니다. 관련 글은 문서를 참조하면 좋을 것같습니다.
```
**FOR UPDATE**
- FOR UPDATE는 SELECT로 가져 온 데이터를 변경을 하려고 할 때 사용한다.
- FOR UPDATE로 SELECT를 가져온 이후로 해당 ROW에 대해 다른 세션의 SELECT, UPDATE, DELETE 등의 쿼리가 모두 잠김 상태가 된다. 즉, FOR UPDATE를 한 세션 외에 다른 세션들은 모두 해당 ROW에 접근을 할 수 없게 되고, 모두 대기 상태가 된다. FOR UPDATE도 LOCK IN SHARE MODE처럼 트랜잭션이 끝나는 시점에서 풀린다. 예로 든 플레이어 간 골드 이전과 같은 경우에 알맞는 잠금이다.
```sql
SELECT balance FROM tb_account  WHERE id = 1 FOR UPDATE # 5.7;
SELECT balance FROM tb_account  WHERE id = 1 FOR UPDATE # 8.0;
```

> 하지만 샘플코드로 FOR UPDATE쿼리로 락을 걸었을때 UPDATE, DELETE는 대기상태로 들어가지만 SELECT쿼리는 수행이된다. 좀 더 조사해보니 FOR UPDATE가 붙은 SELECT쿼리만 동시성이 보장되고 일반적인 SELECT 쿼리를 날렸을 때는 따로 락이 걸리진 않는다. 하지만 그 트랜잭션안에서 update가 발생되면 그 update쿼리는 락이 걸린다.
