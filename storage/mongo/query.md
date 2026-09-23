
### Query

- 조회, 입력, 갱신, 삭제 쿼리 모두 document(json foramt)형식으로 질의하면 된다.

#### Select

- 단순 필드 쿼리
```
// field = value and field2 = value2 and ...
db.collection.find({"field": "value", "field2": "value2", ...})
```
- 비교 연산자 (`$lt` <, `$lte` <=, `$gt` >, `$gte` >=)
```
// field >= 3 and field <= 7
db.collection.find({"field": {"$gte": 3, "$lte": 7}})
```
- 특정 크기의 배열을 가진 document 조회
```
db.collection.find({"array_field": {"$size": 3}})
```

#### Insert

- 하나의 document 입력
```
db.collection.insertOne({"title": "Real MySQL 8.0"})
```

- 여러 document 입력
```
db.collection.insertMany(
    {"title": "Real MySQL 8.0"},
    {"title": "Effective Java 3/E"},
    ...
)
```

#### Update

- `$set`연산자를 이용하여 필드 갱신. 필드가 존재하지 않으면 새 필드가 생성
```
db.collection.updateOne(
    {"_id": ObjectId("4b253b067525f35f94b60a31")},
    {"$set": {"favorite book": "Programming in Scala"}}
)
```
- `$unset`연산자를 이용하여 필드 제거
```
db.collection.updateOne(
    {"_id": ObjectId("4b253b067525f35f94b60a31")},
    {"$unset": {"favorite book": "Programming in Scala"}}
)
```

#### Delete
- 특정 조건을 만족하는 document 한 개 삭제
```
db.collection.deleteOne({"field": "value"})
```
- 특정 조건을 만족하는 document 모두 삭제
```
db.collection.deleteMany({"field": "value"})
```
