# 문서 운영 규칙 (Spec-first)

## Core rules
1) docs/는 제품 코드처럼 관리한다.
- docs/ 변경은 PR 리뷰 대상이다.

2) 모든 문서는 상단에 상태 라벨을 가진다.
- Status: DRAFT | REVIEW | STABLE
- Phase 0(contracts)는 가능한 빨리 STABLE로 고정한다.

3) Phase 0(Contracts)는 헌법이다.
- contracts가 STABLE이 되면, 이후 문서는 contracts를 **링크로만 참조**한다(중복 서술 금지).

4) “왜?”는 ADR로 남긴다.
- docs/adr/0001-*.md 형태로 번호를 붙인다.
- 결정을 바꾸면 ADR을 추가하거나, superseded를 명시한다.

5) docs/README.md는 공식 인덱스다.
- 신규 문서는 docs/README.md에 링크를 추가한다.

## TODO (코드 repo 생긴 뒤)
- [ ] 구현 PR에 설계 영향이 있으면 docs도 함께 수정 (코드 repo에서 강제)
- [ ] 코드 repo가 spec 버전을 pin 하고 CI에서 계약 검증 수행
