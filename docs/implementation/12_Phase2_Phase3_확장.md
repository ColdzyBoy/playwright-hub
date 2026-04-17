# 12. Phase 2 / Phase 3 확장 계획

- 최종 수정일: 2026-04-17
- 관련 스펙: `../specs/05_인프라_배포명세서.md` §6~§8, `../specs/02_요구사항명세서.md` NFR-02(확장성)
- 목적: Phase 1(MVP) 이후 중규모·대규모 팀 대응을 위한 **확장 경로**를 사전에 설계하여, 핵심 Phase 1 코드의 추상화 여부를 판단.

## 1. Phase 2 — 다중 워커 서버 + NFS (2~3주)

### 1.1 목표
- 15~30명 팀 동시 사용 대응
- 코드 변경 **최소화** (주로 인프라/환경변수 조정)
- 워커 장애 복구 메커니즘 확립

### 1.2 구조

```
[Main Server]
  ├── API :3001 + SSE(fastify-sse-v2)
  ├── PostgreSQL :5432
  └── Redis :6379

[NFS / GlusterFS]
  ├── projects/stable/
  ├── projects/working/
  └── reports/

[Worker Server A] (concurrency 4)
[Worker Server B] (concurrency 4)
[Worker Server C] (concurrency 4)
```

### 1.3 구현 체크리스트

#### 1.3.1 NFS/공유 스토리지
- [ ] 메인 서버에 NFS 서버 설치 또는 외부 NFS(AWS EFS 등) 연결
- [ ] 각 워커 서버에 `/mnt/shared` 마운트
- [ ] 각 경로 동기화: `projects/stable`, `projects/working`, `reports`
- [ ] `apps/api/src/lib/storage.ts` — 스토리지 어댑터 인터페이스 (Phase 3 S3 대비)
  - `interface Storage { read, write, copy, remove, stat }`
  - `class LocalStorage`, `class NfsStorage` (Phase 2는 로컬 경로 재사용 가능하므로 별도 구현 없음도 가능)

#### 1.3.2 워커 서버 분리 배포
- [ ] `deploy/worker-compose.yml`
  ```yaml
  services:
    worker:
      image: playwright-hub-api:latest
      command: ["node","dist/workers/playwright.worker.js"]
      environment:
        - DATABASE_URL=postgresql://user:pass@main-server:5432/db
        - REDIS_URL=redis://main-server:6379
        - PROJECTS_STABLE_PATH=/mnt/shared/projects/stable
        - PROJECTS_WORKING_PATH=/mnt/shared/projects/working
        - REPORTS_PATH=/mnt/shared/reports
        - WORKER_CONCURRENCY=4
      volumes:
        - /mnt/shared/projects/stable:/mnt/shared/projects/stable:ro
        - /mnt/shared/projects/working:/mnt/shared/projects/working
        - /mnt/shared/reports:/mnt/shared/reports
        - /var/run/docker.sock:/var/run/docker.sock
  ```
- [ ] 워커 서버 프로비저닝 스크립트 (`deploy/scripts/provision-worker.sh`)
- [ ] 추가/제거 절차 문서화 (`docs/operations.md`)

#### 1.3.3 워커 장애 복구
- [ ] BullMQ `stalledInterval: 30000`, `maxStalledCount: 1` 설정
- [ ] Docker 컨테이너 dangling 감지 — 워커 기동 시 `docker ps --filter label=playwright-hub=true` → 오래된 컨테이너 정리
- [ ] 컨테이너에 label 추가: `--label playwright-hub=true --label run-id=${runId}`
- [ ] cleanup job: Run.status == RUNNING 인데 in-flight worker가 없으면(`worker.getActive()` 비어있고) 30분 경과 시 FAILED 전환 + errorMessage "Worker lost"
- [ ] graceful shutdown 재확인 (SIGTERM → current job 완료 대기 5분)

#### 1.3.4 읽기-쓰기 락 업그레이드
- [ ] `lock.service`를 Redlock으로 업그레이드 (다중 노드 안전성)
  - `npm i redlock`
  - 현재 단순 SET NX → `new Redlock([redis], { retryCount, retryDelay })`
- [ ] 공유(read) 락과 배타(write) 락 구분 고려 (Phase 1은 단순 뮤텍스, Phase 2는 테스트 여러 개 동시 실행 + sync 하나 식 구조가 적합하므로 Lua 기반 counter 방식)

#### 1.3.5 SSE 다중 인스턴스 준비
- [ ] Phase 2는 API 서버 1대 유지. Phase 3에서 API 수평 확장 예정.
- SSE는 Socket.IO와 달리 별도 adapter가 필요 없다. 각 API 인스턴스가 각자의 Redis 클라이언트로 `run:*:events` / `edit:*:events` 채널을 직접 구독하면, 클라이언트 연결이 어느 인스턴스에 붙어 있든 해당 인스턴스가 이벤트를 수신해 SSE 스트림으로 재전송하므로 sticky session 없이도 동작한다.
- Phase 2 단계에서는 API가 1대이므로 추가 설정 없음.

### 1.4 검증 기준

```bash
# 1) 워커 서버 2대 실행
# 메인 서버에서
docker compose up -d

# 워커 서버 A, B에서 각각
docker compose -f deploy/worker-compose.yml up -d

# 2) 동시 8개 실행 요청
for i in {1..8}; do
  curl -s -X POST http://main-server:3001/api/v1/projects/$PID/runs \
    -H "Authorization: Bearer $TOKEN" -d '{}' &
done

# 각 워커 서버에서 docker ps → 4개씩 동시 실행 확인

# 3) 워커 A 강제 종료
docker -H worker-a:2375 kill <worker-container>
# 진행 중 작업이 stalled → B에서 재분배 or FAILED 전환 (attempts=1이면 FAILED)
# 정책에 따라 결정

# 4) NFS 장애 시뮬레이션
# NFS 서버 일시 정지 → 진행 작업 FAILED 전환 + 관리자 알림
```

### 1.5 리스크
- NFS 지연 (큰 리포트 쓰기): async 쓰기 + rsync 전략 or Phase 3 S3 조기 도입
- 워커 서버 간 시계 불일치: NTP 필수
- 공유 스토리지 단일 장애점: Phase 3 S3로 대체

## 2. Phase 3 — 다중 Redis + Queue Router 고도화 + S3 (4~6주)

### 2.1 목표
- 30명 이상 대규모 팀 + 지역별 격리
- API 서버 로드밸런싱
- S3 호환 스토리지로 무한 확장
- Queue Router의 3가지 전략 실전 적용

### 2.2 구조

```
[Load Balancer / ALB]
    ↓
[API Server A]  [API Server B]  (각 인스턴스가 Redis Pub/Sub 직접 구독, sticky session 불필요)
    ↓
[PostgreSQL] + [Redis Master(A, B, C)] + [Queue Router]
    ├── Queue A (projects 1~10) → Worker Group A
    ├── Queue B (projects 11~20) → Worker Group B
    └── Queue C (projects 21~)   → Worker Group C

[S3 호환 스토리지 (MinIO 또는 AWS S3 + CloudFront)]
    ├── projects/
    └── reports/
```

### 2.3 구현 체크리스트

#### 2.3.1 Queue Router 고도화
- [ ] `REDIS_URLS=redis://r1:6379,redis://r2:6379,redis://r3:6379` 파싱
- [ ] `queue-router.service.ts`의 3 전략 실전 구현 (Phase 1에서 스켈레톤만 있었음)
  - `least-busy`: `Queue.getJobCounts('waiting','active')` 합산 최소 큐 선택
  - `round-robin`: 메모리 counter 원자적 증가
  - `project-based`: `prisma.project.workerGroup` 조회 → 매핑 테이블
- [ ] `Project.workerGroup` 필드가 Phase 1부터 존재(M1에서 준비) → 관리자가 수동 할당 가능 UI (settings/org 하위)
- [ ] 전략 전환: 재기동 필요 (hot reload는 Phase 4)
- [ ] 각 Redis 연결 실패 시 다른 큐로 폴백

#### 2.3.2 S3 스토리지
- [ ] `apps/api/src/lib/storage/s3.ts`
  - AWS SDK v3 또는 MinIO SDK
  - `class S3Storage implements Storage`
  - presigned URL 발급 기능
- [ ] 리포트 처리 변경:
  - 워커가 `reports/{runId}` 생성 후 tar+gzip → S3 업로드
  - API가 리포트 요청 시 S3 presigned URL 발급 → 프론트가 직접 fetch
- [ ] 프로젝트 stable/working:
  - 워커가 작업 시작 시 S3에서 pull → local ephemeral → 종료 시 cleanup
  - 또는 NFS(Phase 2)와 하이브리드 유지

#### 2.3.3 API 서버 로드밸런싱
- [ ] AWS ALB 또는 Nginx (HTTP/1.1 → 각 SSE 연결은 장시간 유지되는 단일 TCP 연결)
- [ ] SSE는 Socket.IO와 달리 sticky session **불필요** — 어느 API 인스턴스든 Redis Pub/Sub을 구독해 이벤트를 수신하기 때문.
- [ ] 단, 장시간 연결 유지를 위해 LB/프록시의 idle timeout을 SSE 예상 수명 이상으로 설정(예: 1시간).
- [ ] HTTP/2 활용 시 다중 연결 하나로 통합 가능(`--http2` + Fastify HTTP/2 지원).
- [ ] API 서버 상태 무상태화: 모든 세션 정보는 JWT 또는 DB에서 조회

#### 2.3.4 조직 격리 강화
- [ ] 대규모에서 조직별 큐 할당:
  - `organization.workerGroup` 필드 추가 (신규)
  - `queueRouter.getQueue({ orgId })` 오버로드
- [ ] 관리자 UI에서 조직 → 워커 그룹 매핑 제공 (Phase 4)

#### 2.3.5 리포트 CDN
- [ ] S3 + CloudFront 구성 (또는 MinIO + Nginx 캐싱)
- [ ] 첨부 파일 presigned URL 만료 10분 설정

### 2.4 검증 기준

```bash
# 1) Redis 2대 + 전략
REDIS_URLS=redis://r1:6379,redis://r2:6379 QUEUE_ROUTING_STRATEGY=least-busy
# API 서버 재기동

# 2) project-based 테스트
# DB: project.workerGroup='A' 또는 'B'
# POST /runs → DB의 workerGroup에 따라 R1 또는 R2 큐 등록

# 3) S3 업로드
curl -X POST http://.../runs/...   # 테스트 실행
aws s3 ls s3://playwright-hub-reports/runs/$RUN_ID/
# 또는 MinIO web console

# 4) 리포트 조회 presigned URL
curl http://.../runs/$RUN_ID/report/files/xxx.png
# → 302 Redirect to S3 presigned URL

# 5) API 2대 + LB
# 동시 여러 클라이언트 연결 → 분산 확인 (lb access log)
# SSE 재연결: 클라이언트 A가 API1 → API1 재시작 → EventSource가 자동 재연결 → API2로 라우팅 → Last-Event-ID 헤더로 미수신 이벤트 복원
```

### 2.5 리스크
- S3 GET 요청 비용 (리포트 빈번 조회): ZIP 압축 + 단일 요청 전략, 또는 첨부를 정적 배포
- Redis 파티셔닝 재배치 (프로젝트 이동) 시 큐 마이그레이션 도구 필요
- Redis Pub/Sub 채널 병목: 이벤트 분할 (`run:*`, `edit:*` 별도 채널)
- 다중 Redis 장애 연쇄: Redis Sentinel/Cluster로 HA 구성 (Phase 3+)

## 3. Phase 4 — 포스트 MVP (지속)

우선순위 순:

### 3.1 OAuth 인증
- Google, GitHub, SAML
- `next-auth` 또는 자체 구현
- 기존 이메일/비밀번호 병행

### 3.2 리포트 비교 (FR-04-07)
- 두 Run 선택 → 테스트 케이스 diff + 플레이키 검출
- UI: 좌우 분할 뷰

### 3.3 트렌드 분석 고도화
- 플레이키 테스트 자동 탐지 (최근 N회 실행 중 불일치 비율)
- 실행 시간 회귀 감지

### 3.4 알림 시스템
- Slack/Discord/email webhook
- 트리거: Run FAILED, Edit APPROVED, 비용 한도 접근 등

### 3.5 RBAC 세분화
- 현재 Admin/Member 2단계
- 추가: ProjectAdmin (특정 프로젝트만 관리), Viewer (읽기 전용)
- 프로젝트별 멤버 할당

### 3.6 Claude SDK 고도화
- Bash 도구 허용 + seccomp 프로파일
- 다중 에이전트 오케스트레이션 (테스트 작성 → 검토 → 수정)
- 사용자 지정 시스템 프롬프트 조직별 저장

### 3.7 보안 강화
- rootless docker 또는 podman
- 환경변수 KMS 암호화 (AWS KMS, HashiCorp Vault)
- 감사 로그 (audit log) 별도 테이블
- 2FA, SSO

### 3.8 모니터링·관찰성
- OpenTelemetry 도입
- Grafana 대시보드 (SLO: P95 latency, 성공률)
- Sentry 연동 (프론트/백)
- PagerDuty 연동

### 3.9 Git 고급 기능
- Branch 전환/비교
- PR 생성 자동화 (Edit 승인 시 GitHub PR 오픈)
- Webhook 수신 (push 시 자동 테스트)

### 3.10 UI/UX 개선
- 키보드 숏컷 (cmd+K 명령 팔레트)
- 드래그 앤 드롭 (테스트 재정렬)
- 모바일 반응형 (리포트 조회 중심)

## 4. Phase별 전환 의사결정 기준

### Phase 1 → Phase 2
트리거 중 하나라도 해당 시:
- 동시 활성 사용자 > 10명
- 큐 대기 시간 > 2분 (P95)
- 단일 Worker 서버 CPU 지속 > 80%
- 팀 요청 증가율 > 주 20%

### Phase 2 → Phase 3
- 조직 수 > 5
- 동시 실행 요청 > 20
- 프로젝트별 실행 패턴 명확(조직 격리 필요)
- 지역별 지연 문제 발생

### Phase 3 → Phase 4
- MVP 안정성 확보
- 주요 사용자 피드백 반영 필요
- 기능 확장 요청 누적

## 5. 핵심 설계 원칙 (확장성 관점)

1. **Phase 1부터 Queue Router 인터페이스 존재**: Phase 3 전환 시 코드 변경 최소
2. **Worker → API는 Redis pub/sub만 사용**: 다중 워커/다중 API 자연스럽게 확장
3. **스토리지 어댑터 인터페이스**: local/NFS/S3 전환 용이
4. **Project.workerGroup 필드 Phase 1부터 존재**: 마이그레이션 불필요
5. **SSE는 sticky session 불필요**: 각 API 인스턴스가 Redis를 직접 구독하므로 다중 API 서버 전환이 자연스럽다
6. **환경변수 기반 설정**: 재빌드 없이 배포 변경

## 6. 다음 단계

본 구현 계획 문서 집합은 Phase 4까지의 여정을 개괄한다. 실제 작업은 Phase 0 → M1부터 순차 착수한다. 각 마일스톤 종료 시 `11_리스크_불확실성.md`를 갱신하고, Phase 2 진입 전 본 문서의 1.2절 구조를 다시 검토한다.
