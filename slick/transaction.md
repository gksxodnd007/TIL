### Slick에서 Transaction 적용 (slick version 3.3.1)

Slick을 설정함에 있어 여러가지가 있겠지만 개인적으로 Cake Pattern을 이용하여 Config를 구성해 나가는 것을 선호한다.
> 참조 링크: [Slick Cake Pattern Config](https://scala-slick.org/doc/3.3.1/database.html#the-cake-pattern)

- slick은 DBIOAction을 통해 쿼리를 실행한다. 보통은 type alias된 DBIO를 통해 사용한다.
- future를 DBIOAction으로 바꾸고 싶다면 DBIO.from(...)을 통해 간단하게 바꿀 수 있다.
```scala
// Import the Scala API from the profile
import profile.api._

...

(for {
  _ <- ...
  _ <- DBIO.from(future)
  _ <- ...
} yield (): Unit).transactionally

...
```
- slick은 위와 같이 for comprehension을 통해 여러 DBIOAction을 쉽게 transaction으로 묶을 수 있다.

개인적으로 선호하는 패턴을 적용하여 트랜잭션을 적용한 예제 코드를 통해 설명을 이어 나가겠다.(Slick 설정은 이 글에서 자세히 설명하지 않을 것이다.)

다음은 준비 코드이다.

- build.sbt
```
libraryDependencies ++= Seq(
    "mysql" % "mysql-connector-java" % "8.0.16",
    "com.typesafe.slick" %% "slick" % "3.3.2" exclude("com.typesafe", "config"),
    "org.slf4j" % "slf4j-api" % "1.6.4",
    "com.typesafe.slick" %% "slick-hikaricp" % "3.3.2" exclude("com.typesafe", "config"),
    "com.typesafe" % "config" % "1.3.3"
  )
```

- application.conf
```hocon
slick.db {
  profile = "slick.jdbc.MySQLProfile$"
  db {
    url = "jdbc:mysql://localhost:3306/tmp?useUnicode=true&characterEncoding=UTF-8&cacheServerConfiguration=true&zeroDateTimeBehavior=convertToNull&autoReconnect=false&useSSL=false"
    driver = com.mysql.cj.jdbc.Driver
    user = "user"
    password = "password"
    poolName = "transaction-test"
  }
}
```

- BaseComponent.scala
```scala
trait BaseComponent {

  private lazy val dbConfig: DatabaseConfig[JdbcProfile] = DatabaseConfig.forConfig[JdbcProfile](path = "slick.db", config = ConfigFactory.defaultApplication())
  protected lazy val db: JdbcBackend#DatabaseDef = dbConfig.db
  protected lazy val profile: JdbcProfile = dbConfig.profile
}
```

- User.scala & UserComponent.scala
```scala
case class User(id: Option[Long], name: String, email: String)

trait UserComponent extends BaseComponent {

  import profile.api._

  class UserTable(tag: Tag) extends Table[User](tag, "users") {

    val id: Rep[Option[Long]] = column[Option[Long]]("id", O.AutoInc, O.PrimaryKey)
    val name: Rep[String] = column[String]("name")
    val email: Rep[String] = column[String]("email")

    override def * : ProvenShape[User] = (id, name, email <> (User.tupled, User.unapply)
  }

  protected val users = TableQuery[UserTable]
}
```

- BaseRepository
```scala
trait BaseRepository {

  this: BaseComponent => // Repository가 Component trait mixin을 강제하기 위함
  def ddl: profile.DDL
}

```

- UserRepository
```scala
class UserRepository()(implicit ec: ExecutionContext) extends BaseRepository with UserComponent {

  import profile.api._

  def save(entity: User): Future[User] = {
    db.run(saveQuery(entity))
  }

  def saveQuery(entity: User): FixedSqlAction[User, NoStream, Effect.Write] = {
    users returning users.map(_.id) into ((e, id) => e.copy(id = id)) += entity
  }

  override def ddl: profile.DDL = users.schema
}
```

지금까지 준비한 코드는 간단하다. users테이블의 정보를 User, UserComponent를 통해 mapping하였고 이를 이용하여 Repository layer를 준비하였다. Spring기반 어플리케이션을 개발할 때는 Service layer에서 @Transactional을 통해 손쉽게 메서드를 트랜잭션으로 묶어 트랜잭션 처리를 손쉽게 구현할 수 있었다. 하지만 scala로 개발을 시작하면서 (play or akka) & persistence 계층을 slick을 이용하였는데 트랜잭션 관련 처리를 어느 layer에서 해야하는지 고민이 많았다. slick을 통해 transaction처리를 하려면 `import profile.api._` 구문이 필요한데 여기서 profile은 위의 준비된 코드를 보다시피 repository영역에서 필요한 구문이다. 개인적으로 layer간의 침범을 좋아하지않아 트랜잭션을 처리하기 위해 Service layer에 저 구문을 사용하고 싶지 않았다. 그렇다고 Repository layer에서 비즈니스 로직에 속하는 트랜잭션처리를 하고 싶지도 않았다. 그래서 TxService layer를 별도로 두어 이를 해결하였다. Spring기반 어플리케이션을 개발할 때도 master와 slave로 쿼리를 구분해서 보내는 등 트랜잭션 처리를 하드하게 처리 해야한다면 TxService layer를 별도로 두어 처리하였는데 여기서 영감을 얻었다. 그럼 다음 코드를 살펴보자.

User를 등록할 때 외부시스템에도 해당 유저를 우리 서비스의 유저로 등록한다고 알려야한다고 가정하자.

- BaseTxService
```scala
trait BaseTxService {

  // import profile.api._ 구문이 필요하기 때문에 profile을 정의한 BaseComponent가 필요하다.
  this: BaseComponent =>
}
```

- UserTxService
```scala
// ExtenralClient는 외부 시스템과의 통신을 위한 클래스이다. 따로 코드를 준비하진 않았다.
class UserTxService(
  private val userRepository: UserRepository,
  private val externalClient: ExtenralClient) extends BaseTxService with UserComponent {

  import profile.api._

  def save(entity: User): Future[User] = {
    val tx = (for {
      user <- userRepository.saveQuery(entity) // DBIO.from(userRepository.save(entity))로도 가능
      _ <- DBIO.from(externalClient.send(user).map {
          case Success(_) => ():Unit
          case Failure(_) => throw new RuntimeException("외부 시스템 통신 실패") // Future.failed(...)는 트랜잭션 롤백 사유로 해당되지 않음.
        })
    } yield user).transactionally

    db.run(tx).recover { ... }
  }
}
```
이제 UserService 에서 UserTxService를 통해 유저를 등록하면 된다. 비즈니스 로직은 UserService에 트랜잭션 관련 처리 로직은 UserTxService로 책임을 나누면 layer도 깔끔하게 분리되면서 어떤 layer에 종속적인 부분이 관련 없는 layer로 전파되는 것을 막을 수 있다. Spring기반 어플리케이션은 트랜잭션으로 묶는 과정이 너무 쉬워 자칫하면 트랜잭션을 남용하게 될 수도 있다. 트랜잭션은 비용이 비싼 기능이므로 신중하게 이용하는 것이 개인적으로 좋다고 생각한다.

> 추가로 DBIOAction들을 트랜잭션으로 묶을 때 DBIO.failed로 실패 처리를 할 수 없는 경우 exception을 throw 해주면된다(DBIO.from(...) 안에 있는 Future에서 Exception throw되면 따로 throw 해줄 필요없음). 다만 db.run(tx).recover { ... }를 통해 에러로그를 남기는 등 핸들링을 해주어야한다.
