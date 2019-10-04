### 용어 및 컴포넌트 정리

#### MapReduce
- 여러 노드에 태스크를 분배하는 방법으로 각 노드 프로세스 데이터는 가능한 경우, 해당 노드에 저장.
- map과 reduce 총 두 단계로 구성

#### Shuffling
- 파티션간의 물리적인 데이터 이동

#### Spark Cluster
- 병렬연산이 가능하고 네트워크로 연결된 머신의 집합

#### Sequence File
- Hadoop은 block 단위로 데이터를 저장하는 데 파일이 너무 작으면 효율성이 떨어진다. 이를 보완하기 위해 작은 파일들을 묶어서 바이너리 key, value형태로 생성되는 파일을 Sequence File이라한다.
