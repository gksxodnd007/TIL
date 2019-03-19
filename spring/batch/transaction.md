## Spring batch 트랜잭션 주의할 점

### 트랜잭션을 chunk size만큼 잘라서 job을 실행시킬 경우
- paging size와 chunk size를 같게 설정해주어야한다.
> chunk size만큼 read하기위해 여러번 조회 쿼리를 날리는데 여기서 문제가 발생한다. processor로 넘어가는 데이터는 마지막 쿼리를 날린 reader와 트랜잭션이 물려있기 때문에 chunk size를 맞추기위해 날렸던 전의 쿼리들은 모두 트랜잭션 세션이 끊긴 상태다. 따라서 마지막으로 가져온 데이터들은 LazyInitialize가 잘 되지만 그 전에 데이터들은 모두 LazyInitializeException이 발생하게 된다.
- 페이징을 거꾸로 하여 실행할 경우 페이징 사이즈에 맞춰지지않는 fragment가 생긴다. chunk size를 맞추기 위해 그 다음 페이지의 데이터도 가져온 후 processor가 실행되는데 이 때도 전자와 마찬가지로 트랜잭션 세션이 달라서 LazyInitializeException이 발생하게 된다.
