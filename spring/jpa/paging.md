## Paging 처리와 동시에 수정시 주의사항(같은 트랜잭션일 경우에 해당)

```java
class PagingUpdateTest extends SpringTestSupport {

    @Autowired
    private UserRepository userRepository;

    @Transactional
    @DisplayName("페이징 업데이트 테스트")
    @Test
    void test() {
        givenUserInfo();

        final int size = 1;
        int page = 0;
        long totalPage = 1;
        List<User> updateTarget = new ArrayList<>();

        do {
            Page<User> users = userRepository.findByAge(25L, PageRequest.of(page, size, Sort.Direction.DESC, "id"));

            if (page == 0) {
                totalPage = users.getTotalPages();
            }

            List<User> list = users.getContent();
            list.forEach(e -> e.setAge(26L));
            updateTarget.addAll(list);

            page++;
        } while(page < totalPage);

        userRepository.saveAll(updateTarget);
        List<User> result = userRepository.findAll();
        result.forEach(e -> Assertions.assertEquals(26L, (long) e.getAge()));
    }

    void givenUserInfo() {
        userRepository.save(mockUser(1L, "한태웅", 25L));
        userRepository.save(mockUser(2L, "한태웅", 25L));
        userRepository.save(mockUser(3L, "한태웅", 25L));
        userRepository.save(mockUser(4L, "한태웅", 25L));
        userRepository.save(mockUser(5L, "한태웅", 25L));
        userRepository.save(mockUser(6L, "한태웅", 25L));
        userRepository.save(mockUser(7L, "한태웅", 25L));
    }

    User mockUser(Long id, String name, Long age) {

        return User.builder()
                .id(id)
                .name(name)
                .age(age)
                .build();
    }
}
```
- 위와 같이 같은 트랜잭션에서 페이징하여 데이터를 가져와 수정하는 경우 원하는대로 동작하지 않게된다.
- 원하는 동작 -> 트랜잭션이 적용되어있으므로 영속성 컨텍스트는 메서드가 끝날 때 까지 유지되어야함.
  1. 데이터를 하나씩 가져와서 entity를 수정
  2. 수정마다 updateTarget리스트에 추가
  3. 루프 종료 후 saveAll을 통해 리스트를 한번에 update한다.
- 실제 동작
  1. entity를 수정
  2. 루프의 다음 반복에서 find메서드 호출시 update 후 select를 하게됨.(응?)
  3. 페이징으로 가져오는 다음 entity의 id가 7이 아닌 5인 데이터를 가져옴.(응?)
  4. 결과적으로 데이터가 하나씩 스킵되어 update가 진행됨

위와 같은 이유로 정상적으로 update처리가 되지않는다(트랜잭션 처리를 안하면 정상 동작하게되지만 에러발생시 롤백이 안됨.) 따라서 다음과 같이 해결 하였다.

```java
@Transactional
@DisplayName("페이징 업데이트 테스트")
@Test
void test() {
    givenUserInfo();

    final int size = 1;
    int page = 0;
    long totalPage = 1;
    List<User> updateTarget = new ArrayList<>();

    do {
        Page<User> users = userRepository.findByAge(25L, PageRequest.of(page, size, Sort.Direction.DESC, "id"));

        if (page == 0) {
            totalPage = users.getTotalPages();
        }

        updateTarget.addAll(users.getContent());

        page++;
    } while(page < totalPage);

    updateTarget.forEach(e -> e.setAge(26L));
    userRepository.saveAll(updateTarget);

    userRepository.findAll().forEach(e -> Assertions.assertEquals(26L, (long) e.getAge()));
}
```
- entity 수정을 루프 밖에서 하였더니 테스트를 통과하였다. 많이 생각을 해보았지만 아직 이유를 잘 모르겠다. 아시는 분이 계시다 댓글로 알려주시면 감사하겠습니다.
> paging시 read한 데이터를 수정하면 당연히 발생하는 이슈...내가 너무 무지했다
