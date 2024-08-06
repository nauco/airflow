# Chapter 12. 

### 12.1 Airflow 아키텍처 
![diagram_basic_airflow_architecture.png](..%2F..%2F..%2FDownloads%2Fdiagram_basic_airflow_architecture.png)

- Web Server : web UI 제공
- Scheduler
  - DAG 파일 구분 분석
  - 실행할 task 를 결정하고 task 를 대기열에 배치
- Database 
    - web server 와 scheduler 는 airflow process 인 반면, database 는 airflow에 따로 제공하는 서비스. 
  
- airflow 는 Executor 의 종류에 따라서 구성이 달라지고 설치 방식도 달라진다. 

