# FLUSH LOGS 시 발생하는 일과 RELOAD 권한

MySQL에 대한 로그 파일이 지속적으로 커져 추후 로그를 보며 문제를 분석할 때 확인하기 어려울 거 같다는 생각이 들었다.

때문에 날짜별로 로그를 나눠서 저장함으로써 로그를 분할 저장하는 방식을 생각했다.

MySQL을 찾아보니 FLUSH LOGS라는 SQL Statement를 사용해서 전반적인 로그 파일을 재시작시킬 수 있다고 한다.

<img width="1130" alt="1" src="https://github.com/user-attachments/assets/c96b5fc2-9789-4759-8e6a-73bb644ee26c" />


[MySQL :: MySQL 8.4 Reference Manual :: 15.7.8.3 FLUSH Statement](https://dev.mysql.com/doc/refman/8.4/en/flush.html)

그리고 FLUSH LOGS를 실행했을 때 이후 반응은 로그의 종류에 따라 달라진다고 한다.(저는 Binary Log, Error Log, Slow Query Log만 추출할 예정이었으므로 해당 부분에 대해서만 서술합니다.)

- General Log, Error Log, Slow Query Log: 로그 파일이 그냥 닫혔다가 열림(정확히 말하면 fd 테이블에서 없어졌다가 다시 등록됨) 따라서, 일자별로 분리된 로그 파일들을 얻고 싶다면 파일의 이름을 바꿔야 한다. 만약 파일의 이름을 바꾸게 되면, MySQL 프로세스에서는 해당 로그 파일들이 없다는 것을 인지하고 로그 파일들을 새로 만들고 거기다 이후 로그를 기록하게 된다.
    
    <img width="1157" alt="2" src="https://github.com/user-attachments/assets/169bd44b-12e9-45aa-9625-43d710e44179" />

    
    <img width="1099" alt="3" src="https://github.com/user-attachments/assets/9b248137-7f6a-4c97-9c36-7a1b4c689fb6" />

    
- Binary Log: General Log, Error Log, Slow Query Log와 다르게 단순히 닫혔다 열리는 것이 아니라 기존 시퀀스 넘버에 +1을 한 이름을 가진 Binary Log 파일을 생성하고 거기에 이후 로그들을 기록하게 된다.
    
    ![image.png](attachment:4b5545d0-3591-4bec-a827-2f5135701215:image.png)
    

그리고 공통적으로 RELOAD 권한이 필요하다고 나와있다.

RELOAD 권한에 대해서 알아본 결과 FLUSH에 관련된 SQL Statement를 실행할 수 있도록 하는 권한이고 DML, DCL, DDL에 관련된 SQL Statement는 실행할 수 없다고 나온다.

<img width="1133" alt="4" src="https://github.com/user-attachments/assets/50f92f93-ac44-41db-a7b4-d908fe0cd58a" />


[MySQL :: MySQL 8.4 Reference Manual :: 8.2.2 Privileges Provided by MySQL](https://dev.mysql.com/doc/refman/8.4/en/privileges-provided.html#priv_reload)

또한 mysqladmin에서 해당 권한을 가지고 있는 계정을 통해 mysqladmin flush-logs라는 명령을 입력할 수 있게 된다. 이 명령어는 FLUSH LOGS SQL Statement를 실행하는 것이라고 한다.

<img width="1160" alt="5" src="https://github.com/user-attachments/assets/b78101b7-8eda-4e70-9e8d-d9d094d9c284" />


[MySQL :: MySQL 8.4 Reference Manual :: 6.5.2 mysqladmin — A MySQL Server Administration Program](https://dev.mysql.com/doc/refman/8.4/en/mysqladmin.html)

그래서 테스트를 해보기로 했다. 테스트가 성공한다면 이전에 mysqladmin ping 명령어에 대해서 분석하면서 mysqladmin ping 명령어를 사용하는데 아무 권한도 필요하지 않다는 것을 확인했으므로 로그를 일자별로 분리하는 것, MySQL 서버를 헬스체크하는 것 두 가지 모두를 보안 상의 위협 없이 해결할 수 있을 것이라 판단했다.

### 기존에 만들어진 데이터베이스에 접근 가능한지 테스트 ⇒ 불가능

<img width="1168" alt="6" src="https://github.com/user-attachments/assets/c20b90bd-5eb1-4e06-aae0-780744d35583" />


### DDL 테스트 ⇒ 불가능

<img width="904" alt="7" src="https://github.com/user-attachments/assets/9b83cf68-178b-4f95-9582-0fee953220c0" />


### DML 테스트 ⇒ 불가능(DDL을 못하므로)

### DCL 테스트 ⇒ 불가능(DDL을 못하므로)

# mysqladmin flush-logs 실행에 대한 고찰

테스트를 통해 RELOAD에 권한이 상당히 제한적이라는 것을 확인하고 mysqladmin 계정의 권한을 USAGE에서 RELOAD로 올렸다.

그 다음 문제는 mysqladmin flush-logs 명령을 어떻게 실행할지를 고민했다.

1. mysqladmin 명령어를 쓸 때 사용하는 계정을 호스트 머신이 MySQL 컨테이너에 접속할 수 있도록 허용하기.
2. docker exec 명령어를 활용한 컨테이너 내부 실행

고민 끝에, 1번의 방식은 MySQL 컨테이너가 외부로 노출될 위험이 존재하다고 판단하고 2번의 방식으로 결정했다. 무엇보다 2번의 방식이 끌렸던 것은 아주 간단하게 외부에서 mysqladmin 명령어를 쓸 수 있었기 때문이다. 

<img width="1008" alt="8" src="https://github.com/user-attachments/assets/c122f435-a899-4421-a8a3-1c7abd668470" />


그래서 이제 flush-logs를 어떻게 처리할지 고민도 끝났고 본격적으로 작업을 시작했다.

# 본격적인 로그 분할 작업 시작

## 로그를 분할하는 방식

MySQL의 General Log, Slow Query Log, Error Log는 재밌는 특징이 있다. FLUSH LOGS를 하기 전까지는 파일 이름이 변경되거나 파일의 위치가 이동되도 계속해서 최신 로그가 추가된다는 점이다.

이는 어찌보면 당연한 것인데, MySQL 프로세스가 general.log, slow.log, error.log 등의 로그 파일(이 파일들의 이름은 MySQL 설정파일을 통해 지정할 수 있다.)을 열면, 운영체제로부터 파일 디스크립터(fd)를 할당받는다.

MySQL은 이 fd를 통해 파일을 참조하기 때문에, 파일의 이름이나 경로를 변경(mv 명령어 등)하더라도, 기존 fd를 통해 계속 로그를 기록할 수 있다.

그러므로, 파일의 이름을 바꿨다고 끝이 아니라, `FLUSH LOGS`를 통해 MySQL이 기존 fd를 닫아 더 이상 사용하지 않게 하고, 운영체제가 새로운 로그 파일에게 발급한 fd를 **새로** fd 테이블에 등록하는 과정이 필요한 것이다.

이 특징을 알고 난 뒤(우연찮게도 mv 명령어를 하는 도중 알게 되었다.) 이를 활용하면 로그가 끊기지 않고 유지시킬 수 있을거라고 생각했다.

그래서 생각한 로그 분할 과정은 다음과 같았다.

1. 우선 mv 명령어를 통해 기존 로그 파일들의 위치와 이름을 바꾼다.
2. mysqladmi flush-logs를 실행한다.
3. 이후 새로운 로그 파일(MySQL 설정 파일에서 지정해 놓았던 로그 파일의 이름과 경로를 통해 결정된다.) 생성되고 거기에 최신 로그들이 추가된다.

## logrotate 분석을 통해 확인하는 로그 분할 방식

이렇게 로그를 연속적으로 보관할 방법을 생각하게 된 후, 한 가지 궁금증이 들었다. 많은 사람들이 로그 분할 소프트웨어로 사용하고 있는 logrotate는 내 방식으로 로그의 연속성을 유지할지? 아니면 로그의 연속성을 실제로 유지 안하고 있을지? 그것이 궁금했다.(사실 이 부분 때문에 logrotate를 쓰는 것에 대해서 부정적이었다.)

그래서 logrotate의 시스템 콜을 분석함으로써 확인하고 싶었다.

예를 들어 mv를 실행한다면 다음과 같은 exec 계열 함수가 오게되고 이후 rename 시스템 콜을 사용하여 경로를 옮기게 된다.

<img width="673" alt="9" src="https://github.com/user-attachments/assets/bacbe612-fe54-4f47-9010-3becd19681be" />


<img width="274" alt="10" src="https://github.com/user-attachments/assets/807b29a4-fbfa-4790-8b23-07ab0bb15d71" />


만약 logrotate의 설정 파일이 이렇다면

<img width="620" alt="11" src="https://github.com/user-attachments/assets/d65e9f62-6f17-4883-8962-b37870f526bd" />


custom2.log 파일을 옮긴 다음 echo “seogeonhyuk logda”를 출력할 것이다.

참고로 echo “seogeonhyuk logda”를 strace로 분석하면 다음과 같다.

<img width="1168" alt="12" src="https://github.com/user-attachments/assets/3a9b40ca-9abe-4beb-9bdf-87fa9b37281b" />


맨 끝의 write를 보면 1번 fd에(stdout)에 seogeonhyuk logda라는 메시지를 작성하는 것을 확인할 수 있다.

그럼 방법은 간단하다. rename 시스템 콜 → write(1, “seogeonhyuk logda”) 시스템 콜이 시간 순서대로 보인다면 MySQL 상황에 대입해봤을 때 mv를 하고 mysqladmin flush-logs를 한다는 뜻이 되고, 그 말은 연속적인 로그를 유지한다는 뜻이 된다.

직접 확인해보았다.

```bash
# -f 옵션은 포크된 자식 프로세스까지 추적
# -tt 옵션은 밀리세컨드 단위까지 타임스탬프 남김
# -s 옵션은 문자열 최대 보여지는 길이 늘리기(기본값 32)
sudo strace -f -tt -s 200 logrotate -f /etc/logrotate.d/custom-log
```

<img width="934" alt="13" src="https://github.com/user-attachments/assets/76753868-ea0b-4bc0-97f3-0171d3323cad" />


확인해보면 먼저 rename 시스템 콜을 활용해 custom2.log를 custom2.log.1로 옮기고 chmod에 따른 권한 설정을 한 뒤(fchmod는 chmod의 시스템 콜이다.)

<img width="987" alt="14" src="https://github.com/user-attachments/assets/91eb196f-c1c3-46e2-8934-20fc00b95e0e" />


새로운 프로세스를 포크되어 맨 마지막에 포크된 프로세스가 write(1, “seogeonhuk logda”)를 호출하는 것을 확인할 수 있다.



# 결론

strace를 사용해서 logrotate를 사용해도 로그가 끊기지 않고 누적된다는 것을 확인했으므로, 그냥 logrotate를 써서 구현하기로 결정했다. strace는 이전부터 한 번 써보고 싶었는데, 이렇게 간단하게나마 써볼 수 있어서 좋은 경험이었다.

# 도움 받은 자료들

[strace 사용법 - OS - 한국오라클사용자그룹](http://www.koreaoug.org/os/1207)

[DTrace: [Even Better Than] Strace for OS X | 8th Light](https://8thlight.com/insights/dtrace-even-better-than-strace-for-os-x)

[Using dtrace with SIP enabled](https://poweruser.blog/using-dtrace-with-sip-enabled-3826a352e64b)

[Logrotate를 이용한 로그파일 관리](https://m.blog.naver.com/ncloud24/220942273629)
