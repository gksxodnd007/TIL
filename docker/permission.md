### Containers Run As Root

어떤 시스템이 필요한 Resource에 접근할 때, 해당 시스템은 Resource에 대해 최소한의 필요 권한만을 부여하는 것이 좋다. 해당 시스템을 통해 악의적인 Action이 발생할 수도 있고, 버그로 인해 Resource에 피해를 줄 수도 있기 때문이다. 이를 해결하기 위해 가장 좋은 방법은 최소한의 권한만 부여하여 프로세스를 실행 시키는 것이다. Docker를 실행하기 위해서는 root 권한이 필요하지만 container자체에는 그럴 필요가 없다. 하지만 우리가 Dockerfile을 통하여 이미지를 만들 때 별도의 정의를 하지 않으면 container는 root로 실행된다. 예를 보면 이해할 수 있을 것이다. 먼저 다음과 같이 정의된 Dockerfile을 보자.
```
FROM ubuntu:focal

RUN apt-get update
RUN apt-get install -y git
```
크게 어려울 것 없는 docker image이다. base image인 ubuntu:focal위에 apt-get update를 한 layer를 씌우고 그 위에 git을 설치한 layer를 씌운 이미지이다. 즉, git이 설치된 ubuntu 도커 이미지이다. (docker는 overlayFS를 이용하여 이미지를 생성한다.) 위의 이미지를 다음 명령어로 이미지를 빌드 한 후 실행시켜보자.
```
docker build -t ubuntu:git .
docker run -it ubuntu:git
```
위의 명령어를 실행시키면 다음과 같이 shell에 root계정으로 접속된다.
```
root@a6956c10e0fa:/$
```
위와 같이 root계정으로 실행되면 docker가 file을 root권한으로 write하기 때문에 어플리케이션에서 write한 파일을 사용자가 root권한이 아니라면 수정하거나 삭제할 수 없다. container를 root로 실행시키지 않고 일반 유저로 실행시키고 싶다면 Dockerfile에 다음과 같은 RUN선언을 더 해주면된다.
```
FROM ubuntu:focal

RUN apt-get update
RUN apt-get install -y git
RUN groupadd -g 999 appuser
RUN useradd -r -u 999 -g appuser appuser

USER appuser
```
위와 같이 Dockerfile을 정의한 후 이미지를 빌드하여 실행시키면 이전과 다르게 appuser로 shell에 접속되는 것을 확인할 수 있다.
```
appuser@8d90c2a979de:/$
```
