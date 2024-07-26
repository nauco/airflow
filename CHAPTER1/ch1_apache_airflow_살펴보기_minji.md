
# Chapter 1. Apache Airflow 살펴보기

## 1.1 데이터 파이프라인 

- Task 간의 의존성을 확인하는 방법 중 하나가 pipeline 을 Graph 자료구조로 그리는 것. 
- DAG (Directed Acyclic Graph) : 방향성 비순환 그래프. 반복 및 순환 (cycle)을 허용하지 않음.   

### 1.1.2 파이프라인 그래프 실행 
- DAG 구성을 이용하여 정해진 순서로 Task 를 실행함. 

### 1.1.3 그래프 파이프라인과 절차적 스크립트 파이프라인 비교 
- 각 Task 를 Node 로 생성하고, Task 간의 데이터 의존성을 화살표 끝점으로 연결하여 표현함. 
- 1.1.2 예시와 다른 점은 파이프라인 첫번째 단계가 독립적인 두개의 태스크로 구성되어 있으며, 병렬로 실행할 수 있다는 점이다. 
- Task를 순차적으로 실행하는 것보다 전체 실행 시간을 줄일 수 있으며, 리소스를 효율적으로 사용할 수 있다. 
  

## 1.2 Airflow 소개 
### 1.2.2  파이프라인 스케줄링 및 실행 
Airflow 구성 요소 
- Scheduler : DAG를 분석하고, 현재 시점에서 스케줄이 지난 경우 Worker에 DAG 태스크를 예약함. 
- Worker : 예약된 Task 를 선택하고 실행함. 
- Web Server: DAG를 시각화하고 실행과 결과를 확인할 수 있는 주요 인터페이스를 제공.  
  
- 이해를 돕기위해 docker compose 파일을 첨부하였습니다.
```yaml
# docker-compose.yaml 
version: '3.7'
services: 
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: airflow
      POSTGRES_PASSWORD: airflow
      POSTGRES_DB: airflow
    volumes:
      - resumait-postgres-db-volume:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "airflow"]
      interval: 30s
      retries: 5
      start_period: 5s
    restart: always

  webserver:
    build: .
    restart: always
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CORE__LOAD_EXAMPLES: 'False'
      AIRFLOW__WEBSERVER__DEFAULT_UI_TIMEZONE: 'KST'
      _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
      _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
      AIRFLOW__CORE__FERNET_KEY: ''
      AIRFLOW__WEBSERVER__SECRET_KEY: 'hex(16)값 넣기'
    depends_on:
      - postgres
    ports:
      - "8080:8080"
    volumes:
      - /home/ubuntu/resumait-airflow/dags:/opt/airflow/dags
      - /home/ubuntu/logs:/opt/airflow/logs
    command: >
      bash -c "
        airflow db init &&
        if [ ! -e /usr/local/airflow/airflow-webserver.pid ]; then
          airflow users create --username $${_AIRFLOW_WWW_USER_USERNAME} --password $${_AIRFLOW_WWW_USER_PASSWORD} --firstname Admin --lastname User --role Admin --email mjwoo001@gmail.com;
        fi &&
        airflow webserver"
    healthcheck:
      test: ["CMD-SHELL", "[ -f /usr/local/airflow/airflow-webserver.pid ]"]
      interval: 30s
      timeout: 30s
      retries: 3

  scheduler: 
    build: .
    restart: always
    environment:
      AIRFLOW__CORE__EXECUTOR: LocalExecutor
      AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
      AIRFLOW__CORE__LOAD_EXAMPLES: 'False'
      AIRFLOW__WEBSERVER__DEFAULT_UI_TIMEZONE: 'KST'
      AIRFLOW__CORE__FERNET_KEY: ''
      AIRFLOW__WEBSERVER__SECRET_KEY: 'hex(16)값 넣기'
    depends_on:
      - webserver
    volumes:
      - /home/ubuntu/resumait-airflow/dags:/opt/airflow/dags
      - /home/ubuntu/logs:/opt/airflow/logs
    command: >
      bash -c "
        airflow db init &&
        airflow scheduler"
volumes:
 resumait-postgres-db-volume:


```

- 스케줄러 : 파이프라인이 실행되는 시기와 방법을 결정한다. 
    - 스케줄러의 작업 단계 
      1. DAG 파일을 분석하고, DAG Task, 의존성, 예약 주기를 확인한다. 
      2. 마지막 DAG 까지 내용을 확인 후, 예약 주기가 경과 했는지 확인한다. 예약 주기가 현재 시간 이전이라면 실행되도록 예약한다. 
      3. Scheduler는 해당 Task 의 의존성을 확인한다. 의존성 태스크가 완료되지 않았다면 실행 대기열에 추가한다. 
      4. 1단계로 돌아가서 새로운 루프를 대기한다. 
    - Task 가 실행 대기열 (queue) 에 추가되면 Airflow 워커 pool에 있는 워커가 Task를 선택하고 실행한다. 
    - 실행은 병렬로 수행되고, 결과는 지속적으로 추적된다. 
    - 과정의 결과 (log) 들은 airflow metastore (Database) 로 전달된다. 

### 1.2.4 점진적 로딩 및 백필 
- 정의된 특정 시점에 trigger 할 수 있음. 
- 최종 시점과 예상되는 다음 스케줄 주기를 상세하게 알려준다. 
- 과거 특정 기간에 대해 DAG 를 실행해 새로운 데이터를 back fill 할 수 있다. 

### 1.3 언제 Airflow 를 사용해야 할까 
- Airflow 가 적합한 경우 
  - Batch 지향 파이프라인 구현 
  - Python 에서 구현할 수 있는 대부분으 방법으로 커스텀 파이프라인 개발 가능. 
  - 쉽게 확장 가능, 다양한 시스템과 통합 가능 
  - Web UI 가 훌륭한 편 
  (클라우드 업계 종사자 답계 우리야 좋은 SaaS/PaaS 서비스들에 익숙하지만 오픈소스에서 이정도 인터페이스를 제공하면 훌륭한 편이라고 한다)
    

- Airflow 가 적합하지 않은 경우 
  - 스트리밍 (실시간 데이터 처리) 워크플로 및 처리에 적합하지 않을 수 있다. Airflow 자체가 배치 task & 반복적인 task 에 초점이 맞춰져 있음. 
  - 추가 및 삭제 task 가 빈번한 동적 파이프라인의 경우 적합하지 않을 수 있다. WebUI에서는 가장 최근 실행 버전에 대한 정의만 표현함. 