### Preview
#### GeoJson
- 위치정보를 갖는 점을 기반으로 체계적으로 지형을 표현하기 위해 설계된 개방형 공개 표준 형식
- JSON인 자바스그립트 오브젝트 노테이션(Object Notation)을 사용하는 파일 포맷

Example.
```json
{
  "geometry": {
    "type": "Point",
    "coordinates": [102.0, 0.5]
  }
},
{
  "geometry": {
    "type": "LineString",
    "coordinates": [
      [102.0, 0.0], [103.0, 1.0], [104.0, 0.0], [105.0, 1.0]
    ]
  }
},
{
  "geometry": {
    "type": "Polygon",
    "coordinates": [
      [
        [100.0, 0.0], [101.0, 0.0], [101.0, 1.0],
        [100.0, 1.0], [100.0, 0.0]
      ]
    ]
  }
}
```
- coordinates 필드는 좌표를 의미하며 [경도, 위도] 순서로 배열로 표시
- type 필드는 점, 선, 다각형 등을 의미

#### WGS84
- 세계 지구 좌표 시스템(World Geodetic System, WGS) 1984년에 제정된 범 지구적 측위 시스템으로 지도학, 측지학, 항법에 많이 사용 됨. GPS측량 시 WGS84 타원체를 사용
- 통칭 및 약칭은 WGS 84 (aka WGS 1984, EPSG:4326, WGS84)라고 부르며, 2004년에 마지막으로 개정 됨. 이전에 쓰던 초안으로 WGS 72, WGS 66, 그리고 WGS 60이 있음
- 지표면을 주상절벽으로 모델링하며, 이는 극지방에 약간의 평탄화가 존재함을 의미

### Geospatial Index (공간 정보 인덱스)
- MongoDB는 2dsphere와 2d라는 공간 정보 인덱스를 가진다.
- 2dsphere 인덱스는 WGS84 좌표계를 기반으로 지표면을 모델링하는 구면 기하학으로 작동한다.
- **2dsphere 인덱스를 사용하면 지구의 형태를 고려하므로 2d 인덱스를 사용할 때보다 더 정확한 거리 계산을 할 수 있다.**

#### Create Index
- 2dsphere 인덱스를 만들려면 인덱싱할 도형이 포함된 필드(GeoJson)를 지정하는 도큐먼트를 createIndex에 전달하고 "2dsphere"를 값으로 지정한다.

Example.
```
var restaurant = {
    _id: ObjectId("614b15433268712822ec197f"),
    location: {
        coordinates: [ 127.19824925675954, 37.565344228165934 ],
        type: 'Point'
    },
    name: '긴자 하남점'
}

db.{collection}.insert(restaurant);
db.{collection}.createIndex({"location": "2dsphere"});
```

### Geospatial Queries (공간 정보 쿼리)
- 공간 정보 쿼리는 교차(intersection), 포함(within), 근접(nearness)이라는 세 가지 유형이 있다. 찾을 항목을 `{"$geometry": GeoJson}`과 같은 GeoJson 객체로 지정한다.

Example. `"$geoIntersects"`연산자를 사용해 쿼리 위치와 교차하는 도큐먼트를 찾을 수 있다.
```
var eastVillage = {
  "type": "Polygon",
  "coordinates": [
    [
      [ -73.98255629297816, 40.73135013405199 ],
      [ -73.98246876214036, 40.731314884230514 ],
      [ -73.98150053433653, 40.73091807712006 ],
      [ -73.98082000817642, 40.730639159032464 ],
      [ -73.98034007566318, 40.73044246087391 ],
      [ -73.97926849214927, 40.729981428437476 ],
      [ -73.9780269795082, 40.72943306011252 ],
      [ -73.97805542869892, 40.72938308632418 ],
      [ -73.97853582898918, 40.72874608510769 ],
      ...
}

db.openStreetMap.find({"location": {"$geoIntersects": {"$geometry": eastVillage}}})
```
위의 예는 뉴욕의 East Village 내에 한 점을 갖는 점, 선, 다각형이 포함된 도큐먼트를 모두 찾는 쿼리이다.

- `"$geoWithin"`을 사용해 특정지역에 완전히 포함된 항목을 쿼리할 수 있다.
```
db.openStreetMap.find({"location": {"$geoWithin": {"$geometry": eastVillage}}})
```

#### 범위 내에서 레스토랑 찾기
- 특정 지점으로부터 지정된 거리 내에 있는 레스토랑을 찾을 수 있다. `"$centerSphere"`와 함께 `"$geoWithin"`을 사용하면 정렬되지 않은 순서로 결과를 반환하며, `"$maxDistance"`와 함께 `"$nearSphere"`를 사용하면 거리 순으로 정렬된 결과를 반환한다.
- 원형 지역 내 레스토랑을 찾으려면 `"$centerSphere"`와 함께 `$"geoWithin"`을 사용한다. `"$centerSphere"`는 중심과 반경을 radian으로 지정해 원형 영역을 나타내는 Mongo DB 전용 구문이다. `$"geoWithin"`은 도큐먼트를 특정 순서로 반환하지 않으므로 거리가 가장 먼 도큐먼트를 먼저 반환할 수도 있다.
- 다음은 사용자로부터 5km 이내에 있는 레스토랑을 모두 찾는다.
```
db.restaurants.find(
  {
    "location": {
      "$geoWithin": {
        "$centerSphere": [
          [-73.93414567, 40.82302903],
          5/6378.137
        ]
      }
    }
  }
)
```
- `"$centerSphere"`의 두 번째 인자는 반지름을 radian 값으로 받는다. 쿼리는 거리를 지구의 대략적인 적도 반경인 6378.137km로 나누어 radian으로 변환한다.
- 어플리케이션은 공간 정보 인덱스 없이도 `"$centerSphere"`를 사용할 수 있지만, 공간 정보 인덱스를 사용하면 훨씬 빨리 쿼리를 실행할 수 있다. `2dsphere`와 `2d` 둘 다 `"$centerSphere"`를 지원한다.
- `"$nearSphere"`를 사용하고 `"$maxDistance"`를 미터 단위로 지정할 수도 있다. 사용자로 부터 5km 이내에 있는 모든 레스토랑을 가장 가까운 곳에서 가장 먼 곳 순으로 반환한다.
```
db.restaurants.find(
  {
    "location": {
      "$nearSphere": {
        "$geometry": {
          type: "Point",
          coordinates: [-73.93414567, 40.82302903]
        },
        "$maxDistance": 5 * 1000
      }
    }
  }
)
```
