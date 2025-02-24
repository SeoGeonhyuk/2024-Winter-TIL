# Overlayfs란

overlayfs는 리눅스가 채택하고 있는 파일시스템의 한 종류로 최근 도커에서 기본적으로 사용하고 있는 스토리지 드라이버이기도 하다.

overlayfs는 마운트를 세 가지 종류로 나누고 이 세 종류를 합쳐서 전반적인 파일과 디렉토리를 구성한다.

lower dir: 읽기 전용으로 제공되는 레이어

upper dir: 쓰기 전용으로 제공되는 레이어

merge dir: upper dir과 lower dir을 종합해서 나타내는 레이어

그리고 추가적으로 work dir도 존재하는데, 리눅스 공식 문서에서 해당 work dir은 upper dir에서 변경된 파일을 merge dir로 또는 merge dir에서 변경된 파일을 work dir로 옮길 때 사용한다. 만약 work dir 옵션을 마운트 시 추가하지 않는다면, upper dir 또한 읽기 전용으로 제공된다.

upper dir을 통해 기존 파일을 수정할 수 있는 데, 이때는 COW 작업과 비슷한 COPY UP 작업이 발생한다. 다시 말해 기존 파일에 대한 정보를 그대로 복사해서 쓰기 가능한 레이어인 upper dir로 옮기고(여기서는 merge dir도 해당된다.) 거기서 작업된 내용이 기존 lower dir에 있는 내용을 가리는 방식으로 동작하는 것이다.

# 도커에서의 Overlayfs 주의점

데이터가 큰 상황에서는 writable layer를 통해 이미지 레이어의 데이터를 수정할 경우 기존 읽기 전용의 데이터를 writable layer로 COPY하기 위해서 COPY UP 연산이 발생한다. 문제는 이 COPY UP 연산은 데이터의 크기가 크면 클 수록 속도가 느리다는 것이다. 이 문제를 해결하기 위해서는 스토리지 드라이버 자체를 우회하는 방법을 사용하여야 하는데 그 이유는 스토리지 드라이버 자체가 COPY UP 연산을 만드는 원인이기 때문이다. 도커 공식 문서에서는 스토리지 드라이버를 우회하는 방법으로 볼륨 마운트나 바인드 마운트를 제안하고 있다.

![image.png](attachment:5549e042-45b4-41de-8573-ec5fcd95cfed:image.png)

[OverlayFS storage driver](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/#use-volumes-for-write-heavy-workloads)

## 여담

글쓴이는 혹시나 바인드 마운트와 볼륨 마운트에 대해서도 성능 차이가 있지 않을까 생각했다. 볼륨 마운트는 도커에서 관리하고 있어, 볼륨 마운트 또한 데이터를 기록할 때 Overlayfs를 쓸 거 같은 느낌이 들었기 때문이다. 하지만 테스트 결과 한 번 마운트 된 이후에는 COPY UP 연산을 하지 않고 바인드 마운트처럼 동작한다는 사실을 확인했다.(사실 바인드 마운트에는 COPY UP이라는 개념이 없다. 바인드 마운트가 컨테이너를 덮어씌우기 때문이다.)

실험 방법은 간단했다.

바인드 마운트는 당연히 Overlayfs를 쓰지 않으니까 제외하고, 볼륨 마운트를 했을 경우와 안 했을 경우만을 비교하는 것이다. 최초 볼륨 마운트를 통해서 컨테이너 내의 10GB 데이터를 가져오고, 이후 다시 한 번 같은 볼륨으로 컨테이너를 연결했을 때 최초 동작보다 훨씬 빠른 동작을 보여준다면, 더 이상 COPY UP 연산이 발생하지 않은 것으로 판단할 수 있을 것이다.

반대로 볼륨 마운트 없이 컨테이너를 반복적으로 실행할 경우 컨테이너가 실행될 때마다 COPY UP 연산으로 인해 컨테이너 종료 속도가 일관되게 느릴 것이다.

다음과 같은 스크립트를 기반으로 테스트하였다.

> 테스트 환경
기기: m1 macbook pro 16 inch(m1 pro, 16GB)
도커: Docker version 27.5.1, build 9f9e405
사용 이미지: ubuntu
> 

```bash
#!/bin/bash
echo "Starting file modification..."
start_time=$(date +%s)

echo "Appending data to the file..." >> /test-volume/testFile

end_time=$(date +%s)
elapsed_time=$((end_time - start_time))

echo "File modification took $elapsed_time seconds."

```

```docker
FROM ubuntu:latest

RUN mkdir /test-volume

RUN dd if=/dev/zero of=/test-volume/testFile bs=1M count=10240

COPY ./test-bind.sh /test-bind.sh

RUN chmod +x /test-bind.sh

CMD ["/test-bind.sh"]

```

### 볼륨 마운트 없이 컨테이너를 실행했을 때(5회) ⇒ 평균 19.8초(19초 + 20초 + 20초 + 21초 + 19초)

![image.png](attachment:86d8b908-0ad3-4270-8825-5a6f3668fb75:image.png)

### 최초 볼륨 마운트 적용 상태로 컨테이너를 실행했을 때(5회, 결과를 확인할 수 없어 스톱워치를 통해 측정) ⇒ 평균 0.2초 미만

![image.png](attachment:4a3bc276-0600-481a-aab7-8f90916328f8:image.png)

결과적으로 최초 볼륨 마운트 이후 동작은 정말 빠르게 이루어지므로 볼륨 마운트 시에도 독립적인 파일 시스템을 사용해 COPY UP 연산이 일어나지 않을 가능성이 높다.(더 구체적으로 하려면 strace 등을 활용해야 할 거 같은데, 실험 결과가 워낙 극단적이어서 간단하게만 체크했습니다. 사실 프로젝트 구현 중에 진행한 실험이기도 하고 추후에 보완 작업을 할 수 있다면 남겨두겠습니다.) 뭔가 이럴 거라고 예상은 했지만… 이렇게 직접 실험해보면서 추후 남들에게 관련 질문을 받을 때 확실하게 말할 수 있는 사람이 되고 싶었다.

# 참고 자료

[만들면서 이해하는 도커(Docker) 이미지: 도커 이미지 빌드 원리와 OverlayFS](https://www.44bits.io/ko/post/how-docker-image-work)

[168. [Linux] 투명 셀로판지 이론을 통한 Overlay FS 사용 방법과 유니온 마운트 (Union Mount) 이해하기](https://blog.naver.com/alice_k106/221530340759)

[OverlayFS storage driver](https://docs.docker.com/engine/storage/drivers/overlayfs-driver/)

[1. [Docker] IBM의 도커 파일 시스템 성능 논문 리뷰](https://blog.naver.com/alice_k106/220973165893)

[linux/fs/overlayfs at master · torvalds/linux](https://github.com/torvalds/linux/tree/master/fs/overlayfs)

[The Overlay Filesystem](https://web.archive.org/web/20220930060750/http://windsock.io/the-overlay-filesystem/)