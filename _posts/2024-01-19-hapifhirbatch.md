---
title: "[HAPI FHIR] Batch Processing"
excerpt: "HAPI FHIR의 Batch Processing 개요"

categories:
  - FHIR
tags:
    - HAPI FHIR

toc: true
toc_sticky: true

date: 2024-01-19
---

# HAPI FHIR Batch Processing
HAPI FHIR 5.1.0은 Spring Batch framework를 사용한 batch 프로세스를 도입

- 문제점
  - k8s 같은 컨테이너화된 배포 기술로 전환됨에 따라 점점 더 일반적인 서버 종료로 인해 Spring Batch Job이 제대로 복구되지 않음
- 해결
  - Spring Batch -> “batch2”(커스텀 batch framework) 대체하기 시작(HAPI FHIR 6.0.0부터)
## 디자인
### 정의
HAPI FHIR batch job 정의는 job name, version, parameter json input type, job step의 chain으로 구성됨.
Chain의 각 단계는 생성하는 json output type을 선언하며, 이는 다음 단계의 input type이 됨. chain의 마지막 단계는 일반적으로 resources reindex, disk에 데이터 내보내기 등 job의 작업을 수행하므로 output type을 선언하지 않음.
![Job Definition Steps](https://hapifhir.io/hapi-fhir/docs/images/job-definition.svg)

### Job 제출
Job 정의 후JobInstanceStartRequest에 job name, job parameters json을 채운 다음 해당 요청을 Batch Job Coordinator에 제출하여 해당 job의 인스턴스를 일괄처리(batch processing)하도록 제출할 수 있음
그러면 Batch Job Coordinator가 database에 2개의 레코드를 저장
- QUEUED 상태의 Job Instance
  - 해당 job과 관련된 모든 데이터의 상위 레코드
- QUEUED 상태의 Batch Work Chunk
  - 해당 job에 필요한 첫 번째 작업 “chunk”를 설명. 첫 번째 Batch Work Chunk에는 데이터가 포함되어 있지 않음
    마지막으로 Batch Job Coordinator가 Batch Notification Message Channel(batch2-work-notification이라는 이름의)에 메시지를 게시하여 작업 스레드에 이 첫 번째 작업 chunk가 이제 처리할 준비가 되었음을 알림

### Job Processing - First Step
HAPI FHIR Batch Jobs은 작업 알림 메시지를 기반으로 실행됨
첫 번째 작업 chunk로 프로세스는 시작됨.
이 알람 메시지가 도착하면 message handler는 job 정의에 정의된 첫 번째 단계를 한 번 호출하여 job parameters를 입력으로 전달함.

그 후 handler는 다음 단계를 수행
1. 작업 chunk 상태를 QUEUED에서 IN_PROGRESS 2로 변경. Job Instance 상태를 QUEUED에서 IN_PROGRESS 3으로 변경. Job Instance가 취소되면 상태를 CANCELLED로 변경하고 처리를 중단함
2. Job 정의의 첫 번째 단계는 job parameters 5를 사용하여 실행됨. 이 단계에서는 새로운 작업 chunk가 생성됨. 생성된 각 작업 chunk에 대해 Json은 작업 chunk를 직렬화하고 이를 database에 저장한 다음 Batch Notification Message Channel에 새 메시지를 게시하여 처리 대기 중인 작업 chunk가 있음을 작업 스레드에 알림
3. 단계가 성공하면, 작업 chunk 상태가 IN_PROGRESS에서 COMPLETED로 변경되고 포함된 데이터가 삭제됨
4. 단계가 실행하면, 작업 chunk 상태는 오류 심각도에 따라 IN_PROGRESS에서 ERROR나 FAILED로 변경됨

### Job Processing - Middle steps
Job 정의의 중간 단계는 Job Parameters만 입력으로 사용하는 대신 이전 단계에서 생성된 Job Parameters와 Work Chunk 데이터를 모두 사용한다는 점을 제외하면 동일한 방식으로 실행됨

### Job Processing - Final step
마지막 단계는 새로운 작업 chunk를 생성하지 않는다는 점을 제외하면 중간 단계와 동일한 방식으로 작동

### Gated Execution
Job 정의가 Gated Execution을 갖도록 설정된 경우 다음 단계의 작업 chunk가 시작되기 전에 한 단계의 모든 작업 chunk가 완료되어야 함.

### Job Instance Completion
Batch Job Maintenance Service는 1분마다 실행되어 모든 Job Instance의 상태를 모니터링하고 Job Instance는 해당하는 job instance에 대한 모든 미해결 작업 chunk에 따라 COMPLETED, ERRORED, FAILED로 전환됨.
Job instance가 여전히 IN_PROGRESS인 경우 이 maintenance 서비스는 작업을 완료하는 데 남은 시간도 예측함.


## 결론 
Spring Batch에 대해서 기반 지식이 필요함

참고 
- [HAPI FHIR Batch Processing](https://hapifhir.io/hapi-fhir/docs/server_jpa_batch/introduction.html)
