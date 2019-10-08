### Spark 기초 정리

#### Spark
- Apache Spark는 강력한 클러스터 Computing engine이다.(big data processing에 많이 사용됨)

#### SparkContext
- SparkContext를 생성하는 것은 Spark driver application에서 중요한 단계이다.
- Spark application이 resource manager를 통해 클러스터에 접근할 수 있게 해준다.
> resource manager는 SparkStandalone, YARN, Apache Mesos중 하나가 될 수 있음
- SparkContext는 다음과 같은 기능을 제공한다.
  - Spark application의 현재 상태를 가져올 수 있다.
  - configuration을 세팅할 수 있다.
  - 다양한 서비스에 접근할 수 있다.
  - Job을 취소할 수 있다.
  - Stage를 취소할 수 있다.
  - Closure를 초기화 할 수 있다.
  - SparkListener를 등록 할 수 있다.
  - Persistent RDD에 접근할 수 있다.
- Spark 2.0버전 이전에서는 SparkContext는 Spark의 모든기능에 접근할 수 있는 채널이었다.
  >spark driver program은 SparkContext를 사용하여 클러스터에 접속하였다. (resource manager를 통해)
- SparkContext를 생성하려면 appName(identify spark driver), core number, memory size of executor running on a worker node와 같은 설정 파라미터들이 저장되어있는 SparkConf가 요구된다.
- SQL APIs, Hive, Streaming을 사용하려면 각각 다른 context를 생성해야한다.
  - For SQL : sqlContext, For Hive : hiveContext, For Streaming : streamingContext
```scala
/* example create SparkContext */
val conf = new SparkConf()
.setMaster("local")
.setAppName("Spark Practice")

val sparkContext = new SparkContext(conf)
```
#### SparkSession
- 2.0버전 이전에는 Spark application의 진입점은 위에서 설명한 SparkContext였다. 2.0버전 이후 부터는 SparkSession이 새로운 진입점이 되었다. 즉, SparkSession이라 불리는 드라이버 프로세스로 Spark application을 제어한다.
- SparkSession은 SQLContext, HiveContext, StreamingContext를 모두 조합하고 있다. 모든 API는 SparkSession에서 사용가능하다. (SparkSession은 실제 계산에 사용되는 sparkContext역시 가지고있다.)
- SparkSession 인스턴스는 사용자가 정의한 처리 명령을 클러스터에서 실행한다. 하나의 SparkSession은 하나의 Spark application에 대응한다.
```scala
/* example create SparkSession */
val spark = SparkSession.builder() //SparkSession을 구성하기 위한 메서드
.master("local") //master url을 세팅
.appName("eample of SparkSession") //appName을 세팅
.config("spark.some.config.option", "some-value")
.getOrCreate()

/*
config(...) : Set a config option using this method that are automatically propagated to both ‘SparkConf’ and ‘SparkSession’ configurations.
Its arguments consist of key-value pairs.

getOrCreate() : Gets an existing SparkSession or, if there is a valid thread-local SparkSession, it returns that one.
It then checks whether there is a valid global default SparkSession and, if so, returns that one.
If no valid global SparkSession exists, the method creates a new SparkSession and assigns newly created SparkSessions as the global default.

In case an existing SparkSession is returned, the config option specified in this builder will be applied to existing SparkSessions.
*/
```

- HiveContext를 사용해야한다면 다음과 같이 SparkSession을 생성하면된다.
```scala
val spark = SparkSession.builder()
.master("local")
.appName("example of SparkSession")
.config("spark.some.config.option", "some-value")
.enableHiveSupport() // HiveContext를 사용하기 위한 설정
.getOrCreate()
```

- Access the Underlying SparkContext
```scala
spark.sparkContext
```
