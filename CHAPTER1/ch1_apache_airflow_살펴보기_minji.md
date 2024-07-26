
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


### 코드 뜯어보기 -  Scheduler 
```python 
"""

File : airflow/airflow/jobs/scheduler_job_runner.py

"""

    @retry_db_transaction
    def _schedule_all_dag_runs(
        self,
        guard: CommitProhibitorGuard,
        dag_runs: Iterable[DagRun],
        session: Session,
    ) -> list[tuple[DagRun, DagCallbackRequest | None]]:
        """Make scheduling decisions for all `dag_runs`."""
        callback_tuples = [(run, self._schedule_dag_run(run, session=session)) for run in dag_runs]
        guard.commit()
        return callback_tuples

    def _schedule_dag_run(
        self,
        dag_run: DagRun,
        session: Session,
    ) -> DagCallbackRequest | None:
        """
        Make scheduling decisions about an individual dag run.

        :param dag_run: The DagRun to schedule
        :return: Callback that needs to be executed
        """
        trace_id = int(trace_utils.gen_trace_id(dag_run=dag_run, as_int=True))
        span_id = int(trace_utils.gen_dag_span_id(dag_run=dag_run, as_int=True))
        links = [{"trace_id": trace_id, "span_id": span_id}]

        with Trace.start_span(
            span_name="_schedule_dag_run", component="SchedulerJobRunner", links=links
        ) as span:
            span.set_attribute("dag_id", dag_run.dag_id)
            span.set_attribute("run_id", dag_run.run_id)
            span.set_attribute("run_type", dag_run.run_type)
            callback: DagCallbackRequest | None = None

            dag = dag_run.dag = self.dagbag.get_dag(dag_run.dag_id, session=session)
            dag_model = DM.get_dagmodel(dag_run.dag_id, session)

            if not dag or not dag_model:
                self.log.error("Couldn't find DAG %s in DAG bag or database!", dag_run.dag_id)
                return callback
            
            # start_date 를 기준으로 dag 를 실행하는 부분 
            
            if (
                dag_run.start_date
                and dag.dagrun_timeout
                and dag_run.start_date < timezone.utcnow() - dag.dagrun_timeout
            ):
                dag_run.set_state(DagRunState.FAILED)
                unfinished_task_instances = session.scalars(
                    select(TI)
                    .where(TI.dag_id == dag_run.dag_id)
                    .where(TI.run_id == dag_run.run_id)
                    .where(TI.state.in_(State.unfinished))
                )
                for task_instance in unfinished_task_instances:
                    task_instance.state = TaskInstanceState.SKIPPED
                    session.merge(task_instance)
                session.flush()
                self.log.info("Run %s of %s has timed-out", dag_run.run_id, dag_run.dag_id)

                if self._should_update_dag_next_dagruns(
                    dag, dag_model, last_dag_run=dag_run, session=session
                ):
                    dag_model.calculate_dagrun_date_fields(dag, dag.get_run_data_interval(dag_run))

                callback_to_execute = DagCallbackRequest(
                    full_filepath=dag.fileloc,
                    dag_id=dag.dag_id,
                    run_id=dag_run.run_id,
                    is_failure_callback=True,
                    processor_subdir=dag_model.processor_subdir,
                    msg="timed_out",
                )

                dag_run.notify_dagrun_state_changed()
                duration = dag_run.end_date - dag_run.start_date
                Stats.timing(f"dagrun.duration.failed.{dag_run.dag_id}", duration)
                Stats.timing("dagrun.duration.failed", duration, tags={"dag_id": dag_run.dag_id})
                span.set_attribute("error", True)
                if span.is_recording():
                    span.add_event(
                        name="error",
                        attributes={
                            "message": f"Run {dag_run.run_id} of {dag_run.dag_id} has timed-out",
                            "duration": str(duration),
                        },
                    )
                return callback_to_execute

            if dag_run.execution_date > timezone.utcnow() and not dag.allow_future_exec_dates:
                self.log.error("Execution date is in future: %s", dag_run.execution_date)
                return callback

            if not self._verify_integrity_if_dag_changed(dag_run=dag_run, session=session):
                self.log.warning(
                    "The DAG disappeared before verifying integrity: %s. Skipping.", dag_run.dag_id
                )
                return callback
            # TODO[HA]: Rename update_state -> schedule_dag_run, ?? something else?
            schedulable_tis, callback_to_run = dag_run.update_state(session=session, execute_callbacks=False)

            if self._should_update_dag_next_dagruns(dag, dag_model, last_dag_run=dag_run, session=session):
                dag_model.calculate_dagrun_date_fields(dag, dag.get_run_data_interval(dag_run))
            # This will do one query per dag run. We "could" build up a complex
            # query to update all the TIs across all the execution dates and dag
            # IDs in a single query, but it turns out that can be _very very slow_
            # see #11147/commit ee90807ac for more details
            if span.is_recording():
                span.add_event(
                    name="schedule_tis",
                    attributes={
                        "message": "dag_run scheduling its tis",
                        "schedulable_tis": [_ti.task_id for _ti in schedulable_tis],
                    },
                )
            dag_run.schedule_tis(schedulable_tis, session, max_tis_per_query=self.job.max_tis_per_query)

            return callback_to_run
```