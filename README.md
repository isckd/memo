# K6 부하테스트, Grafana (+ influx DB) 모니터링, Prometheus 인스턴스 메트릭 수집

부하 테스트 및 모니터링 도구 도입기



Spring Boot 기반의 MSA 아키텍처로 구성된 개발 환경에서는, 그동안 부하 테스트 및 모니터링에 적합한 도구를 제대로 사용하지 않고 있었습니다.

대부분 개발자들이 각자 로컬 환경에서 JMeter를 사용하여 부하 테스트를 진행했지만, JMeter는 일반 Thread를 사용하기 때문에 스레드 하나 당 메모리 1MB 이상을 소비하는 단점이 있었습니다.

이로 인해 부하 테스트 환경의 확장성과 효율성에 제한이 있었습니다. 개별적으로 구축한 Ngrinder 역시 비슷한 문제를 가지고 있었습니다.

이에 따라 보다 정형화되고 효율적인 부하 테스트 및 모니터링 도구를 도입하기로 결정했습니다.

먼저, Go 언어의 코루틴(고루틴) 기반으로 동작하여 메모리 효율이 뛰어난 (Java 의 일반 Thread 에 비해 1,000배 이상 메모리 효율이 좋은) K6를 부하 테스트 도구로 적용하였습니다.

K6는 경량화된 구조 덕분에 높은 부하를 생성하면서도 시스템 자원 사용을 최소화할 수 있었습니다.

K6 vs JMeter 부하테스트 도구 비교 : https://grafana.com/blog/2021/01/27/k6-vs-jmeter-comparison/
부하 테스트와 함께 시스템 전반의 성능을 모니터링하기 위해 시각화 도구인 Grafana를 도입했습니다.

Grafana는 K6와의 호환성이 뛰어나 부하 테스트 중 수집된 데이터를 실시간으로 시각화할 수 있어 테스트 결과를 직관적으로 분석할 수 있었습니다.

뿐만 아니라, 부하를 받는 Spring Boot 인스턴스의 성능을 모니터링하기 위해 Prometheus를 적용하였습니다.

Prometheus는 어플리케이션의 메트릭을 수집하고 이를 시계열 데이터로 저장하여 실시간으로 시스템의 상태를 파악하는 데 도움을 주었습니다.

그리고 이러한 메트릭을 저장하고, Grafana와 연동하여 시각화하기 위해 InfluxDB도 도입하였습니다.

InfluxDB는 고속의 데이터 쓰기와 읽기가 가능하여 모니터링 데이터를 빠르게 저장하고 활용하는 데 적합했습니다.



결과적으로, K6, Grafana, Prometheus, InfluxDB를 조합하여 부하 테스트와 모니터링의 통합된 환경을 구축할 수 있었습니다.

설치는 최대한 Docker 를 사용해 일관적인 관리와 재사용성을 높였습니다.

이를 통해 Spring Boot 기반의 MSA 아키텍처에 대한 신뢰성을 높이고, 성능 최적화를 위한 기반을 마련하였습니다.







---
---
---




1. Springboot 인스턴스 메트릭 수집을 위한 Prometheus 설정

```

dependencies {

    implementation("io.micrometer:micrometer-registry-prometheus")

}

```



2. Local PC (업무망) 에서 Docker image 다운로드 및 파일 이관

개발망은 방화벽 문제로 불가하므로 로컬에서 다운받아 이미지를 옮겨야 한다.

아래는 WSL (Windows Linux) 실행환경에서 진행한다.



```

# docker image download

docker pull prom/prometheus:v3.0.1
docker pull grafana/grafana:11.3.1
docker pull influxdb:2.7.10

```



```

# docker image to tar

docker save -o prometheus.tar prom/prometheus:v3.0.1
docker save -o grafana.tar grafana/grafana:11.3.1
docker save -o influxdb.tar influxdb:2.7.10

```



```

cd ;

explorer.exe .

# 이후 열린 파일 탐색기에서 Window 로 파일 이관 → Dev3 로 이관한다.

```



```

# Dev3 에서 아래 명령어로 tar 파일을 docker image 로 변환한다. .tar 파일을 저장한 위치를 지정해야 한다.

docker load -i /path/to/target/prometheus.tar

docker load -i /path/to/target/grafana.tar

docker load -i /path/to/target/influxdb.tar



# image 변환 확인

docker images

```



3 . docker-compose.yml 파일 작성

```

version: '3.7'



services:
  prometheus:
    image: prom/prometheus:v3.0.1
    container_name: prometheus
    ports:
      - "19090:9090" # Prometheus 웹 UI, 9090 포트 사용 중이라 외부 포트는 19090 으로 매핑
    volumes:
      -  ${docker 외부에서 마운트할 디렉토리}/prometheus.yml:/etc/prometheus/prometheus.yml
    restart: always

  grafana:
    image: grafana/grafana:11.3.1
    container_name: grafana
    ports:
      - "33000:3000" # Grafana 웹 UI, 3000 포트 사용 중이라 외부 포트는 33000 으로 매핑
    environment:
      - GF_SECURITY_ADMIN_USER=admin # Grafana 기본 사용자
      - GF_SECURITY_ADMIN_PASSWORD=admin # Grafana 기본 비밀번호
    volumes:
      - grafana-data:/var/lib/grafana
      - ${docker 외부에서 마운트할 디렉토리}/provisioning:/etc/grafana/provisioning # 프로비저닝 디렉토리
    depends_on:
      - prometheus
      - influxdb
    restart: always

  influxdb:
    image: influxdb:2.7.10
    container_name: influxdb
    ports:
      - "8086:8086" # InfluxDB API
    environment:
      - INFLUXDB_DB=metrics # 기본 데이터베이스 이름
      - INFLUXDB_ADMIN_USER=admin
      - INFLUXDB_ADMIN_PASSWORD=admin
      - INFLUXDB_HTTP_AUTH_ENABLED=true
    volumes:
      - influxdb-data:/var/lib/influxdb
    restart: always

volumes:
  grafana-data:
  influxdb-data:





```



4. ${docker 외부에서 마운트할 디렉토리}.prometheus.yml 파일 작성

아래 예시

```

global:
  scrape_interval: 5s # 메트릭 수집 주기

scrape_configs:
  - job_name: 'dapm-dev3'
    metrics_path: '/dapm/monitoring/prometheus'
    static_configs:
      - targets: ['10.30.210.153:10295']

  - job_name: 'prometheus'
    metrics_path: '/metrics'
    static_configs:
      - targets: ['localhost:9090'] # Prometheus 자체 메트릭

```



5. ${docker 외부에서 마운트할 디렉토리}/provisioning/datasources/datasource.yml 파일 작성

```

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



6. docker compose 기동 (docker-compose.yml 파일 존재하는 위치에서)

```

docker-compose up -d

```



7. 그라파나 대시보드 선택

커스텀 가능하지만, 다 만들어진 것을 활용하는게 편함.

https://grafana.com/grafana/dashboards/  에서 검색 가능

https://grafana.com/grafana/dashboards/11378-justai-system-monitor/   사내 springboot 버전을 고려해 springboot 2.1 버전용 대시보드 적용

![image](https://github.com/user-attachments/assets/33850913-bdc1-4fe9-ab8e-d156f7f6f639)


폐쇄망이라 직접적으로 대시보드 다운로드가 안되므로, json 으로 받아서 grafana dashboard 에 upload







---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



이제 k6 부하테스트 툴을 설치하고 테스트해보자.



```

docker pull grafana/k6:0.55.0

docker save -o k6.tar grafana/k6:0.55.0

cd ;

explorer.exe .

# 이후 아까와 같이 .tar 파일을 폐쇄망(dev3) 으로 이관한다.

# load 시 .tar 파일을 저장한 위치를 지정해야 한다.

docker load -i /path/to/target/k6.tar

docker images  #설치 확인

```



${docker 외부에서 마운트할 디렉토리}/load-test/ 에 test-script.js  파일을 작성한다.

예시)

```

import http from 'k6/http';
import { sleep } from 'k6';



export default function () {
    http.get('http://10.30.210.153:10295/dapm/health');
    sleep(1);
}

```



이후 컨테이너 실행은 아래와 같이 진행한다.

```

# /data/home/vcc/docker/load-test:/scripts 을 마운트해서, test-script.js 파일을 실행한다는 의미이다.

docker run --rm -i -v ${docker 외부에서 마운트할 디렉토리}/load-test:/scripts grafana/k6:0.55.0 run /scripts/test-script.js

```

![image](https://github.com/user-attachments/assets/1ad0ca61-90a3-4dce-8db1-408a2fa02f32)








---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------



이제 K6 부하테스트 결과와 Grafana 와 연동해보자.



우선 K6 result 를 dashboard 로 표현할 수 있는 템플릿을 다운받는다.

https://grafana.com/grafana/dashboards/18030-k6-prometheus-native-histograms/     # Grafana Labs 공식 K6 모니터링 템플릿

위와 마찬가지로 Json 으로 다운받아 폐쇄망의 DashBoards 에 import 한다.

