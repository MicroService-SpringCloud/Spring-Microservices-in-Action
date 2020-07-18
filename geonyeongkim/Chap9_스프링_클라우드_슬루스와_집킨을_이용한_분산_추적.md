# 스프링 클라우드 슬루스와 집킨을 이용한 분산 추적

마이크로서비스 아키텍처는 더 작고 다루기 쉬운 부분으로 분해하는 강력한 설계 패러다임이지만, 복잡성이 증가하는 단점이 있습니다.

떄문에, 분산환경에서 디버깅은 중요하며 일반적으로 분산 디버깅을 할 수 있는 기법은 아래와 같습니다.

- 상관관계 ID를 사용해 여러 서비스 사이의 트랜잭션을 서로 연결한다.
- 여러 서비스 사이의 로그 데이터를 검색 가능한 단일 소스로 수집한다.
- 여러 서비스 사이의 사용자 트랜잭션 흐름을 시각화하고 트랜잭션 각 부분의 성능 특성을 이해합니다.


위 기법을 만족하는 기술로 이 책에서는 아래 기술들을 사용합니다.

1. 스프링 클라우드 슬루스 : 상관관계 ID를 사용해 HTTP 호출을 로깅

2. 페이퍼 트레일 : 로그 수집 및 단일 저장소에 저장

3. 집킨 : 수집 로그의 시각화


## 스프링 클라우드 슬루스와 상관관계 ID

앞장에서는 상관관계 ID를 HTTP 헤더에 담기위해 주울 필터, 히스트릭스의 쓰레드 자원 공유 등의 여러 작업을 했습니다.

슬루스는 이러한 작업을 간편하게 제공합니다.

슬루스는 아래와 같은 일을 수행합니다.

`서비스로 들어오는 HTTP 호출을 검사하고, 상관관계 ID가 없다면 생성하여 수집할 수 있도록 로깅`


### 사용 방법

사용 방법은 의존성만 추가하면 됩니다.

compile 'org.springframework.cloud:spring-cloud-starter-sleuth:2.2.3.RELEASE'

추가 후 서비스 호출시 아래와 같이 로그가 남게 됩니다.

1) 
![image](https://user-images.githubusercontent.com/31622350/87863069-25836e00-c992-11ea-89e9-436bb5b823eb.png)

2) 
![image](https://user-images.githubusercontent.com/31622350/87863097-6e3b2700-c992-11ea-87c8-9b9772d0e55e.png)


## 로그 수집과 스프링 클라우드 슬루스

마이크로 서비스에서는 무수한 서비스 인스턴스가 존재합니다.
로그 수집이 단일로 이루어지지 않을 시 아래와 같은 작업을 해야합니다.

1. 여러 서버의 로그를 검사 - 서비스마다의 로그 양이 달라 롤오버 속도도 제각각이며, 수동의 한계
2. 로그를 파싱하고 관련 로그 항목을 식별하는 자체 스크립트 작성
3. 서버에 있는 로그 백업

위 작업들을 하지 않기 위해서는 `로그 데이터를 인덱싱하고 검색할 수 있는 중앙 수집 지점을 만들어 서빙`해야 합니다.

로그 수집 및 저장에 대한 전반적인 그림은 아래와 같습니다.

![image](https://user-images.githubusercontent.com/31622350/87863161-3680af00-c993-11ea-91af-f5a6c449b7a4.png)

이를 지원하는 기술들은 아래와 같습니다.

- ELK
- 그레이로그
- 스플렁크
- 수모 로직
- 페이퍼 트레일

이 책에서는 페이퍼 트레일을 사용하였으며, 사용법은 아래와 같습니다.

예제 서비스들은 모두 도커 컨테이너에서 구동되는것으로 가정하고 있기 때문에, 아래와 같이 docker.sock 에서 나오는 로그를 수집하도록 하였습니다.

예제에서는 `도커 스파우트`란 것을 띄워 페이퍼 트레일로 리다이렉션을 하고 있습니다.
`도커 스파우트`는 간단히 아래 docker.compose.yml 파일을 이용하여 run만 하시면 됩니다.
> ELK에서 사용하는 파일비트라고 생각하면 됩니다.

```yml
logspout:
    image: gliderlabs/logspout
    command: syslog://logs7.papertrailapp.com:27040
    volums:
        - /var/run/docker.sock:/var/run/docker.sock
```


### 주울로 HTTP 응답에 상관관계 ID 추가

아쉽게도 슬루스는 상관관계 ID를 호출에만 부여하고 응답에는 담아주지 않습니다.
> 잠재적인 보안 문제로 인해서 그렇다고 합니다.

때문에, 이 부분은 앞에서 만들었었던 주울 사후 필터를 사용하여 부여하도록 만들 수 있습니다.

먼저 주울 서비스에 의존성을 추가합니다.

compile 'org.springframework.cloud:spring-cloud-starter-sleuth:2.2.3.RELEASE'

이제 슬루스에서 제공하는 Tracer 클래스를 사용하여 부여하면 끝입니다.

```java
@Component
public class ResponseFilter extends ZuulFilter{
    private static final int  FILTER_ORDER=1;
    private static final boolean  SHOULD_FILTER=true;
    private static final Logger logger = LoggerFactory.getLogger(ResponseFilter.class);


    @Autowired
    Tracer tracer;

    @Override
    public String filterType() {
        return "post";
    }

    @Override
    public int filterOrder() {
        return FILTER_ORDER;
    }

    @Override
    public boolean shouldFilter() {
        return SHOULD_FILTER;
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        ctx.getResponse().addHeader("tmx-correlation-id", tracer.currentSpan().context().traceIdString());

        return null;
    }
}
```

## 오픈집킨으로 분산 추적

집킨은 여러 서비스 호출 사이의 트랜잭션을 추적하는 분산 추적 플랫폼입니다.
집킨을 사용하면 트랜잭션의 소요 시간을 그래픽으로 확인할 수 있습니다.

특히, 스프링 클라우드 슬루스와 궁합이 좋습니다.

### 사용법
<br>
1. 각 서비스에 의존성 설정

compile 'org.springframework.cloud:spring-cloud-sleuth-zipkin:2.2.3.RELEASE'

<br>
2. 각 서비스 application.yml 수정

spring.zipkin.baseUrl 에 집킨 서버주소를 넣습니다.

```yml
spring:
  zipkin:
    baseUrl:  localhost:9411
```

<br>
3. 집킨 서버 설치 및 구성

집킨 서버가 별도로 필요하며 스프링 부트 프로젝트로 생성가능합니다.

의존성 추가 후, 어노테이션을 달아 주면 끝입니다.

compile 'io.zipkin.java:zipkin-server:2.12.9'<br>
compile 'io.zipkin.java':zipkin-autoconfigure-ui:2.12.9'

```java
@SpringBootApplication
@EnableZipkinServer
public class ZipkinServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ZipkinServerApplication.class, args);
    }
}
```

<br>
4. 추적 레벨 결정

슬루스는 모든 로그를 집킨으로 전송하지 않습니다.
> 그렇다는 것은 슬루스에서는 샘플링 목적으로 집킨을 사용하는 것 같기도 합니다.

로그 전송량은 application.yml 혹은 java config를 통해 정의할 수 있습니다.

값은 0-1 사이의 값으로 정의해야 하며 의미는 0~100%입니다.

```yml
spring:
  sleuth:
    sampler:
      percentage:  1 # 모든 로그 전송
```

```java
@Bean
public Sampler defaultSampler() {
    return Sampler.ALWAYS_SAMPLE;
}
```