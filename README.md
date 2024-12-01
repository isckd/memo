# K6 부하테스트, Grafana (+ influx DB) 모니터링, Prometheus 인스턴스 메트릭 수집

위 기술에 대해 검색해보면 대부분 로컬 환경에서 혼자 실행하는 것을 목표로 하기 때문에 따로 블로그를 작성한다.
조직에서, 특히 폐쇄망에서 사내 구성원들이 하나의 환경에서 사용하고, 커스텀한 DashBoard 구축을 목표로 진행한다.

<br>
<br>

---

<br>
<br>

## 도입 이유

사내 Spring Boot 기반의 MSA 아키텍처로 구성된 개발 환경에서는, 그동안 부하 테스트 및 모니터링에 적합한 도구를 제대로 사용하지 않고 있었다.

대부분 개발자들이 각자 로컬 환경에서 JMeter를 사용하여 부하 테스트를 진행했지만, JMeter는 일반 Thread를 사용하기 때문에 스레드 하나 당 메모리 1MB 이상을 소비하는 단점이 있었다.

이로 인해 부하 테스트 환경의 확장성과 효율성에 제한이 있었다. 개별적으로 구축한 Ngrinder 역시 비슷한 문제를 가지고 있었다.

이에 따라 보다 중앙화해 관리할 수 있고, 정형화되고 효율적인 부하 테스트 및 모니터링 도구를 도입하기로 결정했다.

> 먼저, Go 언어의 코루틴(고루틴) 기반으로 동작하여 메모리 효율이 뛰어난 (Java 의 일반 Thread 에 비해 1,000배 이상 메모리 효율이 좋은) K6를 부하 테스트 도구로 적용하였다. <br>
K6는 경량화된 구조 덕분에 높은 부하를 생성하면서도 시스템 자원 사용을 최소화할 수 있었다.
K6 vs JMeter 부하테스트 도구 비교 : https://grafana.com/blog/2021/01/27/k6-vs-jmeter-comparison/ 

부하 테스트와 함께 시스템 전반의 성능을 모니터링하기 위해 시각화 도구인 Grafana를 도입했다.

Grafana는 K6와의 호환성이 뛰어나 부하 테스트 중 수집된 데이터를 실시간으로 시각화할 수 있어 테스트 결과를 직관적으로 분석할 수 있었다.

뿐만 아니라, 부하를 받는 Spring Boot 인스턴스의 성능을 모니터링하기 위해 Prometheus를 적용하였다.

Prometheus는 어플리케이션의 메트릭을 수집하고 이를 시계열 데이터로 저장하여 실시간으로 시스템의 상태를 파악하는 데 도움을 준다.

그리고 K6 부하테스트의 결과를 저장하고, Grafana와 연동하여 시각화하기 위해 InfluxDB도 도입하였다.

InfluxDB는 고속의 데이터 쓰기와 읽기가 가능하여 모니터링 데이터를 빠르게 저장하고 활용하는 데 적합했다.

결과적으로, K6, Grafana, Prometheus, InfluxDB를 조합하여 부하 테스트와 모니터링의 통합된 환경을 구축할 수 있었다.

설치는 최대한 Docker 를 사용해 일관적인 관리와 재사용성을 높였다.

이를 통해 Spring Boot 기반의 MSA 아키텍처에 대한 신뢰성을 높이고, 성능 최적화를 위한 기반을 마련해보자.

<br>
<br>

---

<br>
<br>


## 구성도

![](https://velog.velcdn.com/images/mud_cookie/post/5dd7fe1a-658f-433c-acca-c82549b45ca6/image.png)


1. Spring Boot 인스턴스들의 실시간 메트릭 정보들을 요청하고 저장하기 위해 Prometheus 를 도입했다. 각 Spring Boot 인스턴스들은 /actuator/prometheus 엔드포인트를 활성화해야 하고, Prometheus 에서 어느 인스턴스를 몇초마다 Polling 할 건지 지정할 수 있다.

2. Grafana 에서 Spring Boot 인스턴트들을 시각화할 DashBoard 를 만들고, Prometheus 를 Polling 하여 시계열 메트릭 정보를 시각화한다.

3. K6 부하테스트를 진행하고, 각 진행상황 및 결과들을 InfluxDB 에 저장한다.

4. Grafana 에서 부하테스트 결과들을 시각화할 DashBoard 를 만들고, K6 에서 진행한 부하테스트 시계열 데이터를 Polling 하여 시각화한다.

<br>
<br>

---

<br>
<br>


## 구성 특이사항

아래 소개할 설치과정에서 조직 공통으로 사용하기 위해 설정한 특이사항들을 소개한다.

- 2024/11 기준 최신 버전인 
	prometheus:v3.0.1
    grafana:11.3.1
 을 기준으로 진행하고, InfluxDB 는 2.x 버전에서 아직 K6 와의 호환성이 떨어지므로 1.x 버전 중 최신인 1.11.8 으로 진행한다. 대신 InfluxDB 1.x 버전은 웹 UI 를 지원하지 않는다.
- docker-compose 로 prometheus, grafana, influxDB 를 하나로 관리한다.
- K6 는 각 로컬 환경에서 실행하는 것이 일반적이나, 테스트 스크립트 중앙화 및 조직 내 편의성을 위해 폐쇄망에서 Docker 로 설치한다.
다만 Docker K6 는 컨테이너를 일회성으로 띄우는 방식이므로, 위 docker-compose 로 같이 관리하지 않고 docker run 명령어로 실행시킬 수 있게 한다.
- 다수의 구성원들이 작성한 테스트 결과들이 중첩되는 것을 방지하기 위해, Grafana 에서 K6 모니터링 DashBoard 를 조금 커스텀했다.

<br>
<br>

---

<br>
<br>

## 설치 과정

### 1. Spring Boot 에 Prometheus 메트릭 수집 활성화

```
# build.gradle.kts

dependencies {
    implementation("io.micrometer:micrometer-registry-prometheus")
}

# actuator 의 prometheus endpoint 노출하도록 해야 한다.
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: "*"
```

아래는 Postman 으로 /actuator/prometheus 을 호출한 예시이다.
json 형태가 아니라 일반 text 로 보냄에 참고하자.

![](https://velog.velcdn.com/images/mud_cookie/post/34565d44-b682-4cdf-827e-65dc6c442e36/image.png)

아래 정보들이 포함됐음을 확인 가능하다.

1. 애플리케이션 상태
- `application_ready_time_seconds`: 애플리케이션이 요청을 처리할 준비가 되기까지 걸린 시간.
- `application_started_time_seconds`: 애플리케이션이 시작되기까지 걸린 시간.

2. 디스크 사용량
- `disk_free_bytes`: 사용 가능한 디스크 공간 (바이트).
- `disk_total_bytes`: 전체 디스크 용량 (바이트).

3. 쓰레드 풀 관련 메트릭 (Executor)
- `executor_active_threads`: 현재 활성 상태인 쓰레드 개수.
- `executor_completed_tasks_total`: 완료된 작업의 총 개수.
- `executor_pool_core_threads`: 풀의 핵심 쓰레드 수.
- `executor_pool_max_threads`: 풀의 최대 쓰레드 수.
- `executor_pool_size_threads`: 현재 풀의 쓰레드 수.
- `executor_queue_remaining_tasks`: 큐에서 수용 가능한 작업의 남은 공간.
- `executor_queued_tasks`: 큐에 대기 중인 작업 수.

4. HikariCP (JDBC Connection Pool)
- `hikaricp_connections`: 전체 커넥션 수.
- `hikaricp_connections_acquire_seconds`: 커넥션 획득 시간 통계.
- `hikaricp_connections_active`: 활성 상태 커넥션 수.
- `hikaricp_connections_idle`: 유휴 상태 커넥션 수.
- `hikaricp_connections_max`: 최대 커넥션 수.
- `hikaricp_connections_min`: 최소 커넥션 수.
- `hikaricp_connections_pending`: 대기 중인 스레드 수.
- `hikaricp_connections_timeout_total`: 타임아웃 발생 횟수.
- `hikaricp_connections_usage_seconds`: 커넥션 사용 시간 통계.

5. HTTP 요청 메트릭
- `http_server_requests_seconds`: HTTP 요청 처리 시간 통계.
- `http_server_requests_active_seconds`: 활성 요청 처리 시간 통계.
- `http_server_requests_seconds_max`: 요청 처리 시간의 최대값.

6. JDBC 커넥션 메트릭
- `jdbc_connections_active`: 활성 JDBC 커넥션 수.
- `jdbc_connections_idle`: 유휴 JDBC 커넥션 수.
- `jdbc_connections_max`: 최대 JDBC 커넥션 수.
- `jdbc_connections_min`: 최소 JDBC 커넥션 수.

7. JVM 메모리 메트릭
- `jvm_memory_committed_bytes`: JVM이 커밋한 메모리.
- `jvm_memory_max_bytes`: JVM이 사용할 수 있는 최대 메모리.
- `jvm_memory_used_bytes`: JVM이 사용 중인 메모리.
- `jvm_memory_usage_after_gc`: GC 이후 사용 중인 메모리 비율.

8. JVM 쓰레드 메트릭
- `jvm_threads_live_threads`: 현재 활성 쓰레드 수.
- `jvm_threads_daemon_threads`: 현재 활성 데몬 쓰레드 수.
- `jvm_threads_peak_threads`: JVM 시작 이후 최고 쓰레드 수.
- `jvm_threads_states_threads`: 각 상태별 쓰레드 수 (Runnable, Waiting 등).

9. JVM 클래스 로딩 메트릭
- `jvm_classes_loaded_classes`: 현재 JVM에 로드된 클래스 수.
- `jvm_classes_unloaded_classes_total`: JVM 시작 이후 언로드된 클래스 수.

10. JVM GC (Garbage Collection) 메트릭
- `jvm_gc_pause_seconds`: GC로 인한 일시 중단 시간 통계.
- `jvm_gc_memory_promoted_bytes_total`: 힙의 old generation으로 승격된 메모리 총량.
- `jvm_gc_memory_allocated_bytes_total`: GC 후 힙에 할당된 메모리 총량.

11. JVM CPU 및 프로세스 메트릭
- `process_cpu_usage`: JVM 프로세스의 CPU 사용량.
- `process_cpu_time_ns_total`: JVM 프로세스의 CPU 사용 시간 (나노초).
- `process_start_time_seconds`: JVM 프로세스 시작 시간.
- `process_uptime_seconds`: JVM 프로세스 실행 시간.

12. 시스템 메트릭
- `system_cpu_count`: CPU 코어 수.
- `system_cpu_usage`: 시스템 CPU 사용률.

13. 로깅 메트릭
- `logback_events_total`: 로그 레벨별 발생 이벤트 수.

14. Tomcat 세션 메트릭
- `tomcat_sessions_active_current_sessions`: 현재 활성 세션 수.
- `tomcat_sessions_active_max_sessions`: 최대 활성 세션 수.
- `tomcat_sessions_created_sessions_total`: 생성된 세션 총 수.
- `tomcat_sessions_expired_sessions_total`: 만료된 세션 총 수.
- `tomcat_sessions_rejected_sessions_total`: 거부된 세션 총 수.

15. JVM 정보
- `jvm_info`: JVM의 버전 및 런타임 정보.

<br>
<br>



### 2. Prometheus, Grafana, InfluxDB 설치 및 설정

사내 조직은 폐쇄망이라, Windows docker 에서 이미지를 받은 후, tar 파일로 변환 후 폐쇄망으로 이관해 다시 이미지로 변환하는 과정을 거친다.

폐쇄망에서 바로 이미지를 pull 받을 수 있는 경우에는 그럴 필요가 없으니 
바로 docker-compose.yml 파일로 이동하면 된다.

준비 환경
- Local PC : Windows, WSL2, Docker
- 폐쇄망 : Linux, Docker


#### 2-1. Local PC

```
# docker image download

docker pull prom/prometheus:v3.0.1
docker pull grafana/grafana:11.3.1
docker pull influxdb:1.1.18

# docker image to tar
docker save -o prometheus.tar prom/prometheus:v3.0.1
docker save -o grafana.tar grafana/grafana:11.3.1
docker save -o influxdb.tar influxdb:1.1.18

cd ;
explorer.exe .
# 이후 열린 WSL 파일 탐색기에서 Window 로 파일 이관 → 폐쇄망으로 이관한다. 또는 바로 SSH 로 파일 업로드를 해도 된다.
```

<br>


#### 2-2. 폐쇄망

.tar 파일이 이관됐으면 이제부터는 폐쇄망에서 작업한다.

```
# 폐쇄망에서 아래 명령어로 tar 파일을 docker image 로 변환한다. .tar 파일을 저장한 위치를 지정해야 한다.

docker load -i /path/to/target/prometheus.tar
docker load -i /path/to/target/grafana.tar
docker load -i /path/to/target/influxdb.tar

# image 변환 확인
docker images
```

<br>

#### 2-3. docker-compose.yml

관리 편의성을 위해서 ${docker 외부에서 마운트할 디렉토리} 는 모니터링 관련 디렉토리를 따로 만들어서 마운트하자.


```
version: '3.7'

services:
  prometheus:
    image: prom/prometheus:v3.0.1
    container_name: prometheus
    ports:
      - "9090:9090" # Prometheus 웹 UI
    volumes:
      -  ${docker 외부에서 마운트할 디렉토리}/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - monitoring_network	# 모니터링 전용 network 이름을 지정
    restart: always

  grafana:
    image: grafana/grafana:11.3.1
    container_name: grafana
    ports:
      - "3000:3000" # Grafana 웹 UI
    environment:
      - GF_SECURITY_ADMIN_USER=admin # Grafana 기본 사용자
      - GF_SECURITY_ADMIN_PASSWORD=admin # Grafana 기본 비밀번호
    volumes:
      - grafana-data:/var/lib/grafana
      - ${docker 외부에서 마운트할 디렉토리}/provisioning:/etc/grafana/provisioning # 프로비저닝 디렉토리
    depends_on:
      - prometheus
      - influxdb
    networks:
      - monitoring_network	# 모니터링 전용 network 이름을 지정
    restart: always

  influxdb:
    image: influxdb:1.11.8	# influxdb 2.x 버전은 k6 와 호환성이 떨어져 1.x 버전 중 최신으로 진행
    container_name: influxdb
    ports:
      - "8086:8086" # InfluxDB API
    environment:
      - INFLUXDB_DB=metrics # 기본 데이터베이스 이름
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=admin
      - INFLUXDB_HTTP_AUTH_ENABLED=false	# Grafana DashBoard 에서 바로 접근 가능하도록 HTTP Auth 인증을 false 로 지정
    volumes:
      - influxdb-data:/var/lib/influxdb
    networks:
      - monitoring_network	# 모니터링 전용 network 이름을 지정
    restart: always

volumes:
  grafana-data:
  influxdb-data:

networks:				# 모니터링 전용 network 이름을 지정
  monitoring_network:
    driver: bridge
    name: monitoring_network
```

<br>

#### 2-4. ${docker 외부에서 마운트할 디렉토리}/prometheus.yml 파일 작성

아래 scrape_configs 내부에 Spring Boot 인스턴스 별로 job 을 추가하자.
prometheus 자체 메트릭은 기본값으로 추가해두자.

```
# prometheus.yml
global:
  scrape_interval: 5s # 메트릭 수집 주기

scrape_configs:
  - job_name: 'test'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['localhost:8080']  # 

  - job_name: 'prometheus'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['localhost:9090'] # Prometheus 자체 메트릭
```

<br>

#### 2-5. ${docker 외부에서 마운트할 디렉토리}/provisioning/datasources/datasource.yml 파일 작성


```
# datasource.yml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090 # Prometheus 컨테이너 이름 사용
    isDefault: true

  - name: InfluxDB
    type: influxdb
    access: proxy
    url: http://influxdb:8086 # InfluxDB 컨테이너 이름 사용
    database: metrics
    user: admin
    password: admin
    jsonData:
      httpMode: POST   
```

```
docker compose up -d	# docker-compose.yml 이 존재하는 위치에서 실행
docker ps -a	# Grafana, Prometheus, InfluxDB 컨테이너 기동 확인
```

<br>
<br>

### 3. Spring Boot Grafana DashBoard 적용

Grafana, Prometheus, InfluxDB 컨테이너들이 모두 기동이 완료되었다면, Grafana 에서 DashBoard 로 시각화해보자.

우선 Prometheus 에 metric 정보들이 잘 수집이 되는지 확인한다.
http://폐쇄망Host:3000/targets 으로 Prometheus 웹 UI 에 진입하자.

![](https://velog.velcdn.com/images/mud_cookie/post/9cac647f-5592-477b-a994-0bf060ffcb5d/image.png)

기본 prometheus metric 과, 별도로 Spring Boot 를 모니터링하기 위한 test job 이 5초 주기로 잘 수집되는 것을 확인할 수 있다.

이후 Grafana 에 접속한다.
초기 username/pw 는 docker-compose.yml 에 설정했던 값으로 진행한다. (admin/admin)

![](https://velog.velcdn.com/images/mud_cookie/post/8d6bfcfa-2620-4d4b-ae75-eb2e12862d74/image.png)

이후 pw 변경까지 완료하면 아래와 같은 좌측 사이드 탭이 뜬다.
먼저 `Data sources` 탭에서 Promethues 와 InfluxDB 와의 Connection 이 잘 되는지 확인한다.

![](https://velog.velcdn.com/images/mud_cookie/post/be5f8346-085a-40a5-86cc-39f5e4046b1b/image.png)

![](https://velog.velcdn.com/images/mud_cookie/post/ba802856-1c4a-4b10-a551-9d5348988ff9/image.png)

이후 DashBoards 탭으로 진입해 시각화 템플릿을 import 하자.

![](https://velog.velcdn.com/images/mud_cookie/post/551bba55-3898-4810-aaa0-273f1f599bf0/image.png)

![](https://velog.velcdn.com/images/mud_cookie/post/8fa4b725-d2bf-47da-b9ac-0ab3bdcffe21/image.png)

여기서 DashBoard Id 나 Json 은 https://grafana.com/grafana/dashboards/
에서 검색해서 가져온다.
Grafana 의 장점이 사용자가 많다보니 이런 오픈소스 템플릿이 잘 구비되어있다는 점이다.

사내에서는 Spring Boot 2.1 버전이 가장 많이 쓰이고 있으므로, 
https://grafana.com/grafana/dashboards/11378-justai-system-monitor/
위 Grafana Labs solution 에서 직접 제공하는 Spring Boot 2.1 버전용 DashBoard template 을 사용한다.

![](https://velog.velcdn.com/images/mud_cookie/post/c79bb579-1903-49d0-a163-65ff9e227f1d/image.png)

외부망과 연결이 되는 상태라면 Copy ID 로 설치한 Grafana 에서 ID 만 넣으면 되고, 연결이 되지 않는 상태라면 .json 을 다운받아 코드를 복사해 붙여넣고 Load 하면 된다.

이후 아래 화면에서 연결할 data source 를 prometheus 로 지정하고 Import 한다.

![](https://velog.velcdn.com/images/mud_cookie/post/dff8e07b-bbfd-45b2-b2e5-0a922c3899f2/image.png)

그러면 아래와 같이 시각화되는 모습을 볼 수 있다.
Instance 탭에는 prometheus.yml 파일에서 지정했던 targets 을 선택해서 원하는 Spring Boot 인스턴스의 메트릭 정보를 확인하면 된다.

![](https://velog.velcdn.com/images/mud_cookie/post/202a3c59-825f-48dc-bcdf-59db741e4860/image.png)


<br>
<br>


### 4. K6 부하테스트 설치 및 Grafana DashBoard 적용

앞서 언급했다시피, Docker K6 는 일회성으로 컨테이너가 생성되다보니, docker-compose.yml 로 통합시키지 않고 Docker run 으로 실행시키고자 한다.

폐쇠망의 경우, 역시 위 Windows 에서 진행했던 것과 같이 image 를 pull 하고 tar 파일로 변환해 폐쇄망으로 이관 후 다시 image 로 변환하는 과정을 거치자.

#### 4.1 k6 image pull 후 변환 해 이관, 역변환

```
docker pull grafana/k6:0.55.0
docker save -o k6.tar grafana/k6:0.55.0

cd ;
explorer.exe .

# 이후 아까와 같이 .tar 파일을 폐쇄망으로 이관한다.
# load 시 .tar 파일을 저장한 위치를 지정해야 한다.

docker load -i /path/to/target/k6.tar
docker images  #설치 확인
```


#### 4.2 ${docker 외부에서 마운트할 디렉토리}/load-test/test-script.js 작성

k6 테스트를 위한 script 를 작성한다.
k6 는 기본적으로 Javascript 로 구동되는데, 크게 어려울 것은 없고 자세한 스크립트 작성법은 
[K6 부하테스트 스크립트 작성법](https://velog.io/@mud_cookie/K6-%EB%B6%80%ED%95%98%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%8A%A4%ED%81%AC%EB%A6%BD%ED%8A%B8-%EC%9E%91%EC%84%B1%EB%B2%95)
에서 설명한다.

```
// #test-Script.js
import http from 'k6/http';
import { sleep } from 'k6';

export let options = {
	// InfluxDB 저장 시에 이러한 테스트 스크립트로 실행되었다 라는 것을 명시하기 위해 작성한 커스텀 필드를 작성한다.
    // Grafana 에서 시각화 시 여러 개가 중첩되어 보이는 것을 방지하기 위함이고, DashBoard 역시 커스텀해야 한다. 
    // 이 커스텀 필드는 K6 Grafana DashBoard 에서 추가로 설명한다.
    tags: { test_name: "test-script-1" }, // 태그 추가
};

export default function () {
	// 테스트할 API 를 지정한다.
    // 만약 Linux 에서 Spring Boot 인스턴스가 docker 외부의 localhost 에 존재한다면, localhost -> 172.17.0.1 으로 대체한다.
    // Windows 또는 Mac 환경이라면 host.docker.internal 로 대체.
    const res = http.get('http://localhost:8080/test');
    sleep(1);
}
```

<br>

#### 4.3 K6 test-script.js 실행

이제 K6 스크립트를 Docker 로 실행해보자.

```
docker run --rm --network monitoring_network \
  -v ${docker 외부에서 마운트할 디렉토리}/load-test:/scripts grafana/k6:0.55.0 run \
  --out influxdb=http://influxdb:8086/metrics \
  /scripts/test-script.js
```

명령어를 하나하나 살펴보면 이렇다.
- --rm : K6 컨테이너는 일회성이라 실행이 끝나면 컨테이너가 자동으로 종료되지만, 삭제되지는 않아 디스크 남용을 방지하기 위해 삭제를 명시한다.\
- --network monitoring_network : 이전 docker-compose.yml 에 명시했던 docker network 에서 influxDB 와의 connection 을 위함
- -v : 스크립트를 docker 외부에서 설정하고 끌어오기 때문에 마운트 설정
- --out influxdb=http://influxdb:8086/metrics : 테스트 결과를 InfluxDB 로 저장
- /scripts/test-script.js : 마운트한 디렉토리에서 (${docker 외부에서 마운트할 디렉토리}/load-test) test-script.js 를 실행함을 알린다.

각 개발자는 ${docker 외부에서 마운트할 디렉토리}/load-test 디렉토리에서 스크립트를 작성하고, 위 명령어에서 test-script.js 대신 본인이 작성한 스크립트 명을 넣기만 하면 된다.

터미널에서 실행한 K6 스크립트 결과는 아래 예시와 같이 출력된다.
참고로, K6 는 진행상황과 결과를 InfluxDB 에 1초마다 저장한다.

![](https://velog.velcdn.com/images/mud_cookie/post/ca24afb4-b638-44c0-b0fe-d3dd27cb1b02/image.png)

<br>

#### 4.4 Influx DB 저장 확인

Grafana 로 시각화 전에 Influx DB 에 정상적으로 저장됐는지 확인해보자.

```
# docker influxdb 컨테이너 내부로 진입, influx 명령어 사용
docker exec -it influxdb influx

# DATABASES 목록 확인
SHOW DATABASES

# 결과 예시
# docker-compose.yml 의 influxdb 에서 INFLUXDB_DB=metrics 을 설정했음을 기억하자.
# name: databases
# name
# ----
# metrics
# _internal


# metrics DATABASE 사용
USE metrics


# MEASUREMENT 목록 확인. 수집된 컬럼들이 존재해야 한다.
SHOW MEASUREMENTS

# 결과 예시
# name: measurements
# name
# ----
# data_received
# data_sent
# http_req_blocked
# http_req_connecting
# http_req_duration
# http_req_failed
# http_req_receiving
# http_req_sending
# http_req_tls_handshaking
# http_req_waiting
# http_reqs
# iteration_duration
# iterations
# vus
# vus_max


# test-script.js 에서 설정한 test_name 이라는 tag 값이 잘 저장되었는지 확인
SHOW TAG VALUES WITH KEY = "test_name"


# 저장된 값 중 상위 10개 확인 예시 (MEASUREMENT 목록 중 하나를 선택)
SELECT * FROM http_req_connecting LIMIT 10

# 결과 예시
# name: http_req_connecting
# time                expected_response method name                        proto    scenario status test_name     tls_version url                         value
# ----                ----------------- ------ ----                        -----    -------- ------ ---------     ----------- ---                         -----
# 1732982789752285597 true              GET    http://httpbin.test.k6.io   HTTP/1.1 default  308    test-script-1             http://httpbin.test.k6.io   1.085468
# 1732982790340330238 true              GET    https://httpbin.test.k6.io/ HTTP/1.1 default  200    test-script-1 tls1.3      https://httpbin.test.k6.io/ 1.010407
```

<br>

#### 4.5 Grafana K6 DashBoard 적용

기본적으로는 https://grafana.com/grafana/dashboards/2587-k6-load-testing-results/
템플릿을 사용하려 했으나, 테스트 결과가 중첩되는 문제가 발생해 템플릿을 조금 커스텀했다.

DashBoard 의 variabels 에 test_name 을 추가하고,
SHOW TAG VALUES WITH KEY = "test_name" 값을 넣었다.
이후 DashBoard 의 각 그래프에서 test_name 변수 값을 기준으로 아래와 같은 WHERE 조건문을 넣었다.
`WEHRE "test_name" =~ /^$test_name$/`

그래서 완성된 json 파일은 아래 Github 에 넣어두었다.
`k6 Load Testing Results-with-test_name.json` 파일을 download 받으면 된다.
https://github.com/isckd/memo/blob/main/k6%20Load%20Testing%20Results-with-test_name.json

json 파일을 기준으로 DashBoard 를 import 하는 것은 위에서 이미 설명했으므로 생략한다.
import 가 완료되었다면 아래와 같은 화면이 출력된다.

> 내가 커스텀한 것은 test_name 이라는 변수 값으로, 강조한 박스 안에서 원하는 test_name 태그를 선택하면 해당 결과만 출력할 수 있다.

![](https://velog.velcdn.com/images/mud_cookie/post/c213d003-283b-4bb8-a71d-739f692fa12d/image.png)

DashBoard 를 어떻게 커스텀했는지는 아래에 작성한다.

<br>
<br>

---

<br>
<br>


## Grafana DashBoard 커스텀 방법

Grafana DashBoard 커스텀 방법을 알아보자.
크게는 두 가지로 나뉜다.
- UI 에서 변경하는 방법
- Json 코드를 변경하는 방법

UI 에서 변경하면, 자동으로 Json 코드도 변경된다.
단순 반복적인 InfluxDB 쿼리 변경이라고 하면, UI 에서 필요없이 Json 코드에서 변경해도 무방하다.

내가 커스텀한 내용을 기반으로 진행해보자.
필요한 것은 K6 테스트 스크립트 별로 유니크한 태그 값이 필요한 상황이므로, 
K6 테스트 스크립트 안에 tag 값을 집어넣는다.

```
// #test-Script.js
import ...

export let options = {
    tags: { test_name: "test-script-1" }, // 태그 추가
};

export default function () {
    ...
}
```

이 test_name 이라는 InfluxDB 값이 저장되었으므로, Grafana 에서 불러와야 한다.
K6 Grafana DashBoard 에 진입해 우측 상단의 Edit -> Settings 에 진입한다.

![](https://velog.velcdn.com/images/mud_cookie/post/23b290a8-c73e-4241-9645-4c018b276fcd/image.png)

![](https://velog.velcdn.com/images/mud_cookie/post/75dc8f3c-d705-4c03-b7d8-363526d00057/image.png)

이후 Variables 탭 -> New variable 으로 진입한다.

![](https://velog.velcdn.com/images/mud_cookie/post/01964b5d-39b5-4ca2-9da9-8cb6aec94e9d/image.png)

아래 번호에 맞게 진행한다.
1. InfluxDB 에서 Query 로 가져올 것이므로 Query 를 선택한다.
2. 변수의 명을 지정한다.
3. Data source 를 InfluxDB 로 지정한다.
4. 변수들을 가져올 쿼리명을 지정한다. 이번에는 
`SHOW TAG VALUES WITH KEY = "test_name"` 와 같이 TAG 를 가져온다.
5. DashBoard 상단의 변수 선택에서 정렬을 어떻게 할 건지를 지정한다. 입맛에 맞게 진행한다.
6. Multi-value : 다중 선택이 가능한지를 묻는다.
Include All option : All(전체 선택) 옵션이 가능한지를 묻는다.
7. 현재 DashBoard 에 변수로 보여질 값들이 노출된다. 6. 번에서 All 옵션을 선택했으므로 All 변수도 추가된다.

![](https://velog.velcdn.com/images/mud_cookie/post/b042d7b2-06eb-4ffc-93c2-4ef1b1185c24/image.png)

다시 DashBoard 탭으로 돌아와서, 아직 Save dashboard 로 따로 저장하지 않은 상태임에도 Grafana에서 저장 전 실시간 DashBoard 업데이트한 화면을 보여준다. 
아래 화면과 같이 test_name 이라는 변수들이 잘 노출됨을 보여준다.

![](https://velog.velcdn.com/images/mud_cookie/post/4624b47f-6b0d-47fc-8d62-98f825db7bd4/image.png)

아직 끝이 아니다. 각 그래프들에 변수 WHERE 조건을 추가해주어야 한다.
각 그래프들도 결국 InfluxDB 에서 값을 조회해서 노출해주는 것일 뿐이다.
먼저 그래프 하나를 선택해 쿼리를 지정하는 방법을 알아보자.

### 1. UI 에서 그래프별로 커스텀하는 방법

그래프 노드에 마우스를 올리면 메뉴 바가 노출되고, 그것을 클릭해 Edit 탭으로 진입한다.

![](https://velog.velcdn.com/images/mud_cookie/post/5eaa9488-738c-4298-b516-267e0a05c5d8/image.png)

이후 쿼리 수정 버튼을 눌러 쿼리를 수정하자.

![](https://velog.velcdn.com/images/mud_cookie/post/e826660e-c159-4d91-89fe-7ae93f592e45/image.png)

기존에는 `SELECT mean("value") FROM "vus" WHERE $timeFilter GROUP BY time($__interval) fill(none)` 처럼 되어 있었지만, 
여기서 WHERE 절 뒤에`test_name =~ /^$test_name$/ AND` 절을 추가하자.
결론적으로 
`SELECT mean("value") FROM "vus" WHERE test_name =~ /^$test_name$/ AND $timeFilter GROUP BY time($__interval) fill(none)`
와 같이 수정하면 된다.

이후 상단의 test_name 변수값을 조정하며 정상적으로 노출되는지 확인한다.

![](https://velog.velcdn.com/images/mud_cookie/post/64db4c7b-5590-4375-85d4-5693c118689b/image.png)

<br>

### 2. Json 에서 일괄 적용하는 방법

다시 Settings 탭으로 돌아와 JSON Model 탭에서 Json 코드를 수정해보자.

![](https://velog.velcdn.com/images/mud_cookie/post/e8bfeda7-a215-4fc6-b18c-a5182a52c513/image.png)

단순 작업이므로 JSON 코드에서 `WHEHE ` 이라는 문자열을
`WHERE test_name =~ /^$test_name$/ AND ` 으로 일괄 변경하고 저장하자.
저장은 우측상단의 Save dashboard 로 저장 가능하다.

이후 저장 후 DashBoard 화면으로 돌아오면 아래와 같이 적용됐음을 확인 가능하다.

![](https://velog.velcdn.com/images/mud_cookie/post/7220e4aa-589d-416f-a933-e9798809e2e7/image.png)

---

<br>
<br>
