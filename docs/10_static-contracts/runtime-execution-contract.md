---
Status: DRAFT
Phase: 0
Owner: @seoyhaein
Last-Updated: 2026-02-10
Depends-On:
  - ./paths-and-mount-policy.md"
---

# 런타임 실행 계약 (Runtime Execution Contract)

## 1. 범위/용어

이 문서는 파이프라인의 각 노드가 실행되는 컨테이너/Pod/Job 내부에서,
**동일한 방식으로 실행을 시작하고 종료하며 결과를 산출**하기 위한 “불변 계약(Contract)”을 정의한다.

- 목적
  - 분석 이미지(툴/파이프라인)가 달라도 **동일한 런타임 인터페이스**로 실행 가능
  - Spawner/Watcher/Repair/옵션 Sidecar-Lite가 **실행 모델을 바꾸지 않고** 관측/수렴을 수행 가능
- 용어
  - runtime: 컨테이너 내부 실행 환경(파일시스템, entrypoint, env, signals)
  - executor.sh: 컨테이너 내부에서 사용자 스크립트/툴 실행을 표준화하는 래퍼(예: entrypoint)
  - stage-in: 실행 전 필요한 설정/매니페스트/작은 파일을 `/in/artifacts`로 주입하는 단계
  - result artifact: `/out` 아래에 남기는 결과/메타/로그 파일
  - CAS: content-addressable storage (sha256 키 기반 저장소)

## 2. 전체 아키텍처 흐름

- 입력 준비/물리화 → `/in/**`에 RO로 제공
- 런타임 실행은 `/work/**`에서 중간 산출, `/out/**`에 최종 산출
- (외부) Watcher/Repair가 `/out` 결과를 CAS 업로드하고 Provenance에 연결
- 기본 전달: `node_out -> CAS -> next_node_in`
- (옵션) Sidecar/mini-agent가 붙어도 **메인 실행 계약은 동일** (관측만 추가)

## 3. 구성요소

- Spawner: 이 계약을 만족하도록 PodSpec/JobSpec를 구성(경로, env, entrypoint)
- Runner(Actor): 노드 단위 실행 상태를 DB에 커밋하고, 제출/관측을 수렴
- Watcher/Repair: 실행 결과를 관측하여 DB 상태 전이 및 CAS/Provenance 갱신
- (옵션) Sidecar-Lite / mini-agent: 관측/업로드를 보조하되 실행 로직은 변경하지 않음

## 4. 헬스 설정 API

- (N/A: 계약 문서)

## 5. 헬스 상태 읽기 API

- (N/A: 계약 문서)

## 6. 테스트

최소 테스트는 “계약 위반이 즉시 드러나도록” 한다.

- 경로 계약 테스트
  - `/in` 쓰기 시 실패(RO)
  - `/work`, `/out` 쓰기 성공
- 종료코드/결과 테스트
  - 성공 종료코드(0) 시, `/out`에 최소 결과 파일이 존재
  - 실패 종료코드(비0) 시, `/out`에 실패 메타/로그가 남아 있음
- 신호 처리 테스트(권장)
  - SIGTERM 수신 시, 정리(가능한 범위) 후 종료

## 7. 실행 샘플

컨테이너 내부에서의 “표준 실행 흐름” 예시:

1. `/in/**`에서 입력을 읽는다(쓰기 금지)  
2. 중간 파일은 `/work/**`에 쓴다  
3. 최종 결과는 `/out/**`에 정리한다  
4. 종료코드로 성공/실패를 명확히 표현한다

## 8. 구성/경로 정책

경로 정책은 `paths-and-mount-policy.md`를 따른다.

- `/in` : RO (입력/참조/스테이지인)
- `/work` : RW (중간 산출)
- `/out` : RW (최종 산출)

권장 서브 경로:

- `/in/sample` : 샘플 입력
- `/in/ref` : 참조 번들(RO)
- `/in/artifacts` : stage-in(RO)
- `/out` : 결과/로그/메타

## 9. 오류/이벤트 기록

런타임은 실패 시 “원인 파악”이 가능하도록 최소 정보를 남긴다.

- 필수(권장) 기록
  - 실행 시작/종료 시각(또는 duration)
  - 실행 명령/버전(가능하면)
  - 종료코드
  - 표준 출력/에러 로그(또는 그 경로)
- 계약 위반(/in write 등)은 명시적인 에러 로그로 남긴다.

## 10. 운영 체크리스트

- 모든 실행 Pod/Job이 동일한 entrypoint(또는 동일한 표준 래퍼)를 사용한다.
- `/in`은 RO로 강제된다.
- `/out` 결과가 Watcher에 의해 CAS/Provenance로 연결된다.
- 실패 시에도 `/out`에 디버깅 가능한 메타/로그가 남는다.

## 11. 한계 및 로드맵

- 실행 계약이 너무 세밀해지면 이미지/툴 다양성을 제한할 수 있음
  - 따라서 Phase 0에서는 “불변 최소”만 고정하고, 나머지는 권장/가이드로 둔다.
- (옵션) Sidecar-Lite, mini-agent 도입 시에도 이 계약을 유지

## 12. 빠른 시작

- 컨테이너는 `/in`에서 읽고, `/work`와 `/out`에만 쓴다.
- 성공/실패는 종료코드로 표현하고, 결과/로그는 `/out`에 남긴다.

## 13. 부록

### A. 최소 결과 산출물(권장안)

- `/out/_meta/`
  - `run.json` (선택): attemptKey/nodeId/image digest 등 메타
  - `result.json` (선택): 성공/실패/요약 지표
- `/out/_logs/`
  - `stdout.log` / `stderr.log` (또는 단일 `executor.log`)

### B. 환경변수(초안)

(필요 시 표준화)

- `ATTEMPT_KEY`
- `NODE_ID`
- `SPAWN_ID`
- `ARTIFACTS_DIR=/in/artifacts`
- `SAMPLE_DIR=/in/sample`
- `REF_DIR=/in/ref`
- `WORK_DIR=/work`
- `OUT_DIR=/out`
