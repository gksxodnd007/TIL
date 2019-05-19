### 인덱스 선정 기준
**결합 인덱스를 생성할 때 어떤 컬럼을 앞에 둘 것인지는 생각보다 쉽게 정할 수 있음**
- Cardinality
  - cardinality는 상대적인 개념
  - cardinality가 높다는 것은 중복도가 낮다는 것을 의미(primary key값은 unique이므로 어떤 컬럼보다 가장 카디널리티가 높음.)
  - Mysql docs를 보면 다음과 같이 설명 되어있음.
  ```
  mysql> SHOW INDEX FROM City\G
  *************************** 1. row ***************************
          Table: city
     Non_unique: 0
       Key_name: PRIMARY
   Seq_in_index: 1
    Column_name: ID
      Collation: A
    Cardinality: 4188
       Sub_part: NULL
         Packed: NULL
           Null:
     Index_type: BTREE
        Comment:
  Index_comment:
        Visible: YES
     Expression: NULL
  *************************** 2. row ***************************
          Table: city
     Non_unique: 1
       Key_name: CountryCode
   Seq_in_index: 1
    Column_name: CountryCode
      Collation: A
    Cardinality: 232
       Sub_part: NULL
         Packed: NULL
           Null:
     Index_type: BTREE
        Comment:
  Index_comment:
        Visible: YES
     Expression: NULL
  ```
  > An estimate of the number of unique values in the index. To update this number, run ANALYZE TABLE or (for MyISAM tables) myisamchk -a.
  Cardinality is counted based on statistics stored as integers, so the value is not necessarily exact even for small tables. The higher the cardinality, the greater the chance that MySQL uses the index when doing joins.
  - Cardinality는 통계에의한 추정치이며 ANALYZE TABLE을 통해 update할 수 있다. Cardinality는 통계를 기반으로 만들어진 값이며 정확하지 않을 수 있다. Cardinality값이 높으면 Mysql의 Optimizer가 그 인덱스를 사용할 확률이 높다. 위의 SHOW INDEX FROM City 쿼리에 결과를 보면 PRIMARY 인덱스의 cardinality가 월등히 높은 것을 확인할 수 있다. 이는 중복도가 그만큼 낮기 때문이다.
