# Chapter 12. 

### Airflow 아키텍처 

<img src="./img/diagram_basic_airflow_architecture.png">

- Web Server : web UI 제공
- Scheduler
  - DAG 파일 구분 분석
  - 실행할 task 를 결정하고 task 를 대기열에 배치
- Database 
    - web server 와 scheduler 는 airflow process 인 반면, database 는 airflow에 따로 제공하는 서비스. 
  
- airflow 는 Executor 의 종류에 따라서 구성이 달라지고 설치 방식도 달라진다. 

### Executor 종류

#### 1. SequentialExecutor
- 시연 / 테스트 환경에 적합 
- 가장 단순하게 구성 가능 
- Task 를 하나씩 순차적으로 실행시킴 
- 작업 처리 속도가 상대적으로 느리고 단일 호스트 환경에서만 작동함. 

#### 2. LocalExecutor
- 단일 호스트 환경에서 작동함. 
- 여러 Task 를 병렬로 실행할 수 있음
- Executor 내부적으로 worker process가 FIFO 방식을 통해 대기열에 실행할 Task 등록함.   
  - Q. Local Executor 내부적으로 Task 를 등록하는 방식에 대해 설명해보세요. 
- 최대 32개 병렬 프로세스 실행 가능. 

#### 3. CeleryExecutor
<img src="./img/celery.png">  

- Celery 란 : 
  - Python Application 에서 Task들을 나눠서 처리하도록 해주는 분산 시스템. 실시간 처리 및 작업 스케줄링을 제공하는 Queue 매커니즘. 
  - Web UI 로 Celery Flower 를 사용하여 worker들을 모니터링 할 수 있다. 

<img src="./img/run_task_on_celery_executor.png">  
  Celery queue 는 2가지 컴포넌트로 구성되어 있다.    

    * Broker : 실행할 명령들을 저장함. 
    * Result Backend : 완료된 결과 상태를 저장함. 

- Celery Executor 작동 방식 
  - Celery 를 이용하여 실행할 Task 들을 대기열에 등록시킴. 
  - Worker가 대기열에 등록된 Task 를 읽어와서 개별적으로 처리함. 
  
#### 4. KubernetesExecutor