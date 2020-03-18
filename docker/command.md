### Docker 명령어 정리

**root 계정으로 컨테이너에 접속**

`docker run --user root exec -it {container-id} /bin/bash`

**root 권한으로 jvm option과 함께 java application image를 실행**

`docker run --user root run -p 8080:8080 --env JAVA_OPTS="-Dphase=dev" {image-name}`

> root권한으로 컨테이너를 실행시키지않으면 application로그를 root 디렉토리 하위에 디렉토리를 생성할 수 없어 로그를 남기지 못한다.

**docker 이미지 삭제**

`docker rmi {image-id}`

**docker 이미지와 컨테이너를 동시 삭제**

`docker rmi -f {image-id}`

**가장 최근에 만들어진 컨테이너의 id를 획득**

`docker ps -alq`

**컨테이너 레이어의 변겨사항을 보여주는 명령어**

`docker diff`

**docker diff로 확인 가능한 변경 사항을 이미지로 저장하는 명령어**

`docker commit {container-id} {new-image-name}`
