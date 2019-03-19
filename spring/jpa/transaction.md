
## Spring Data JPA에서 트랜잭션 롤백
- 보통 spring data를 이용하면 영속성 컨텍스트의 생명주기는 트랜잭션의 유지시간과 동일함.
  - exception이 발생하여 롤백 시 트랜잭션이 닫히면서 자연스럽게 영속성 컨텍스트도 닫힌다. 문제 발생할 여지 없음
- OSIV처럼 영속성 컨텍스트의 생명주기가 트랜잭션보다 길 때 발생 할 수 있는 문제점.
  - 데이터베이스에는 데이터가 반영되지 않았지만 영속성 컨텍스트에는 데이터가 잔류 되어있다.
  - spring frame work에서는 다음과 같은 방법으로 문제를 해결한다.
    - 트랜잭션 롤백 시 entityManager.clear()를 호출하여 영속성 컨텍스트를 초기화한다.

**관련 코드**
```java
protected void doRollback(DefaultTransactionStatus status) {
    JpaTransactionManager.JpaTransactionObject txObject = (JpaTransactionManager.JpaTransactionObject)status.getTransaction();
    if (status.isDebug()) {
        this.logger.debug("Rolling back JPA transaction on EntityManager [" + txObject.getEntityManagerHolder().getEntityManager() + "]");
    }

    try {
        EntityTransaction tx = txObject.getEntityManagerHolder().getEntityManager().getTransaction();
        if (tx.isActive()) {
            tx.rollback();
        }
    } catch (PersistenceException var7) {
        throw new TransactionSystemException("Could not roll back JPA transaction", var7);
    } finally {
        if (!txObject.isNewEntityManagerHolder()) {
            txObject.getEntityManagerHolder().getEntityManager().clear();
        }

    }

}
```
- 위의 코드는 JpaTransactionManager의 메소드이다. try-catch-finally 구문중 finally 부분을 확인해보면 entityManager.clear()를 호출하는 것을 볼 수 있다.

## Spring Data JPA에서 Entity Lazy Loading
- Entity들이 Lazy fetch로 설정되어 있다면 주의해야할 것들이 많다. 특히 @Transactional이 적용되어있지 않은 서비스 메서드안에서 entity를 가져올 때 Lazy proxy exception이 발생하게 되는데 트랜잭션 유지기간이나 fetch join을 통해 이런 예외처리를 잘 피해가야한다.
