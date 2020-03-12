## Docker compose

-----------------

### 설치

```
sudo curl -L "https://github.com/docker/compose/releases/download/{1.24.1}/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose


## 권한부여 
sudo chmod +x /usr/local/bin/docker-compose
```
------------
### 정보

**1. 도커 컴포즈란?** 

컴포즈의 사전적 의미는 ‘짓다’, ‘조립하다’이다. 도커 컴포즈는 여러 개의 컨테이너를 짓고 조립(함께 사용)하는 데 유용하다. 여러 컨테이너에 대한 옵션을 docker-compose.yml이라는 파일로 작성하면, docker-compose up이라는 한 번의 명령어로 서비스를 시작할 수 있다.

즉 하나의 컨테이너로 이루어지지 않는 프로그램 ex) webapp <br>
같은 경우에 DB와 web 으로 연관되거나 지금 사용하는 airflow 같은 경우에도 DB 와 airflow 가 연동되는 형태이다. 그럴 경우 DB 컨테이너 run, airflow 컨테이너 run 이렇게 하며 run 할때 옵션으로 연결해주고 해야하는 작업을 통해서도 가능하지만 **부가 옵션이 많고 더군다나 순서에 의존적이다 (무조권 DB 먼저 run) 또한 실행할 때마다 명시를 해줘야함**   


**2. 구성 방법**

1. Dockfile로 애플리케이션 환경을 정의한다.
2. 앱을 구성하는 services를 docker-compose.yml에 정의해서 한꺼번에 실행 가능하도록 한다.
3. docker-compose up 명령어로 컴포즈를 실행해 앱을 시작한다.


2번에서 말하는 services에는 실행하려는 컨테이너들을 정의하면 된다. 도커 컴포즈에서는 services 항목에 앱 구성에 필요한 컨테이너들을 여러 개 정의할 수 있다.

또한, services 안 각 컨테이너 항목에서는 도커 이미지를 실행(run)할 때 쓰던 커맨드라인에 쓰던 여러 옵션(ex. ports)들을 적어둘 수 있다.

------------------------------

### 실습

airflow 로 실습

```
# git에서 repo 끌어옴 yml 파일같은것이나 파일 
git clone https://github.com/puckel/docker-airflow

# airflow 이미지 pull
docker pull puckel/docker-airflow 
```

#### 실행

```
 docker-compose -f docker-compose-LocalExecutor.yml up -d 
```

**localExecutor.yml 파일**
```
version: '2.1' 
        # docker-compose 버전입니다. 작성일 기준으로 3.0을 권장하지만 아직 2.1을 쓰고 있네요.  
        # 서비스는 `postgres`와 `webserver` 두개로 구성되어 있습니다. 

services:
    postgres:
        image: postgres:9.6                 # db는 postgres 공식 이미지를 가져옵니다.
        environment:                        # db 구성에 필요한 값들을 넣어줍니다.
            - POSTGRES_USER=airflow
            - POSTGRES_PASSWORD=airflow
            - POSTGRES_DB=airflow

    webserver:                              # 여기서 부터 airflow 네요.
        image: puckel/docker-airflow:1.10.3 # puckel 계정 아래 있는 이미지 사용
        restart: always                     # 컨테이너 중단된 경우, 자동으로 재시작 해주는 옵션
        depends_on:
            - postgres                      # postgres를 db로 사용하니까 의존성 설정을 해줍니다.
        environment:
            - LOAD_EX=n                     # 여기는 이미지를 구성할때 설정한 환경변수를 넣어줍니다.
            - EXECUTOR=Local                # 아래에서 이미지 구성 파일을 보며 확인하겠습니다.
        volumes:
            - ./dags:/usr/local/airflow/dags # 현재 경로의 dags 폴더를 컨테이너 dags에 마운트합니다.
            # Uncomment to include custom plugins
            # - ./plugins:/usr/local/airflow/plugins
        ports:
            - "8080:8080"                   # 컨테이너 port 8080을 localhost의 8080으로 맞춰줍니다.
        command: webserver                  # 다 끝나면 웹서버 명령어로 ui를 띄우네요.
        healthcheck:                        # -f 로 webserver pid가 잘 생성이 되었는지 확인합니다.
            test: ["CMD-SHELL", "[ -f /usr/local/airflow/airflow-webserver.pid ]"]
            interval: 30s  # 30초 마다 하네요.
            timeout: 30s   # 30초 동안 기다려 주고
            retries: 3     # 3번 재시도.. 합니다. 자주 죽어서 쓰나봅니다
```





현재 t2_small 까진 많은 job이 문제 있음


**위 git엔 celery 사용하는 yml 파일도 있음**

사용하기 전엔 아래 yml 파일에서
FERNT_KEY 를 내가 발급 받는 걸로 바꿔줘야함 

**발급명령어** 
```
docker run puckel/docker-airflow python -c "from cryptography.fernet import Fernet; FERNET_KEY = Fernet.generate_key().decode(); print(FERNET_KEY)"
```

**Celery Yml 파일** 
```
version: '2.1'
services:
    redis:
        image: 'redis:3.2.7' # 먼저 mq를 담당할 redis입니다.
        command: redis-server --requirepass redispass
                  # 이전에 rabbitMQ랑 db 연동해서 설치한다고 고생 좀 했었는데, 도커로 하니 좋네요.
                  # `requirepass` 부분으로 redis 비밀번호를 설정합니다.
    postgres:
        image: postgres:9.6 # 여기서도 db로 postgres를 씁니다.
        environment:
            - POSTGRES_USER=airflow
            - POSTGRES_PASSWORD=airflow # db 로그인 설정을 해주고
            - POSTGRES_DB=airflow
            - PGDATA=/var/lib/postgresql/data/pgdata # db의 경로를 설정해 줍니다.
        volumes:
            - ./pgdata:/var/lib/postgresql/data/pgdata # 위에서 설정한 db 경로를 마운트 해줍니다.

    webserver:
        image: puckel/docker-airflow:1.10.3
        restart: always 
        depends_on:
            - postgres # db
            - redis    # mq
        environment:
            - LOAD_EX=n
            - FERNET_KEY=46BKJoQYlPPOexq0OhDZnIlNepKFf87WFwLbfzqDDho= 
                    # endpoint.sh에서 본 것 같아요.
                    # airflow.cfg를 보니 이렇게 설명해주네요. 
                    # 'Secret key to save connection pass in the db' 
                    
            - EXECUTOR=Celery # executor를 설정 해줍니다.
            
            - POSTGRES_USER=airflow
            - POSTGRES_PASSWORD=airflow
            - POSTGRES_DB=airflow
            - REDIS_PASSWORD=redispass 
                    # 깃헙에서 받은 파일에서는 db 연동, redis 연동 부분이 주석처리 되어있는데, 
                    # entrypoint.sh에 위의 파라미터들이 기본값으로 입력되어 있습니다.
                    # 실 환경에서 사용할 때는 마찬가지로 바꿔주어야 합니다.
        volumes:
            - ./dags:/usr/local/airflow/dags
            # Uncomment to include custom plugins
            # - ./plugins:/usr/local/airflow/plugins
        ports:
            - "8080:8080"
        command: webserver
        healthcheck:
            test: ["CMD-SHELL", "[ -f /usr/local/airflow/airflow-webserver.pid ]"]
            interval: 60s
            timeout: 30s
            retries: 3
            
     flower: # flower는 샐러리 워커들의 상황을 모니터링하는 web ui입니다. 
        image: puckel/docker-airflow:1.10.3
        restart: always
        depends_on:
            - redis
        environment:
            - EXECUTOR=Celery
            - REDIS_PASSWORD=redispass
        ports:
            - "5555:5555"
        command: flower

    scheduler:
        image: puckel/docker-airflow:1.10.3
        restart: always
        depends_on:
            - webserver # scheduler를 db, mq에 의존성 설정을 할 줄 알았는데 webserver만 보고 있네요.
                        # 처음에 띄울때 설정만 따라가나보다, depends_on의 depends_on 느낌으로.
        volumes:
            - ./dags:/usr/local/airflow/dags
            # Uncomment to include custom plugins
            # - ./plugins:/usr/local/airflow/plugins
        environment:
            - LOAD_EX=n
            - FERNET_KEY=46BKJoQYlPPOexq0OhDZnIlNepKFf87WFwLbfzqDDho=
            - EXECUTOR=Celery
            - POSTGRES_USER=airflow
            - POSTGRES_PASSWORD=airflow
            - POSTGRES_DB=airflow
            - REDIS_PASSWORD=redispass
        command: scheduler

    worker: # 실제 작업을 수행하는 worker를 띄우는 컨테이너입니다. 
        image: puckel/docker-airflow:1.10.3
        restart: always
        depends_on:
            - scheduler
        volumes:
            - ./dags:/usr/local/airflow/dags
            # Uncomment to include custom plugins
            # - ./plugins:/usr/local/airflow/plugins
        environment:
            - FERNET_KEY=46BKJoQYlPPOexq0OhDZnIlNepKFf87WFwLbfzqDDho=
            - EXECUTOR=Celery
            - POSTGRES_USER=airflow
            - POSTGRES_PASSWORD=airflow
            - POSTGRES_DB=airflow
            - REDIS_PASSWORD=redispass
        command: worker 
```




참고

[도커컴포즈](https://roseline124.github.io/kuberdocker/2019/07/24/docker-study06.html)

[도커 에어플로우](https://moons08.github.io/programming/airflow-with-docker/)

[도커 에어플로우(celery)](https://moons08.github.io/programming/airflow-with-docker2/)



