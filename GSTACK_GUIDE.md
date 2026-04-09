# GStack 사용 가이드

> **GStack** (Garry's Stack) — Claude Code용 AI 엔지니어링 워크플로우 스킬 모음  
> 버전: 0.15.15.1 | 제작: Garry Tan (YC CEO)

Claude Code 대화에서 `/스킬이름`으로 실행합니다.

---

## 목차

1. [QA & 테스팅](#1-qa--테스팅)
2. [배포 워크플로우](#2-배포-워크플로우)
3. [디버깅 & 코드 품질](#3-디버깅--코드-품질)
4. [디자인](#4-디자인)
5. [플랜 리뷰](#5-플랜-리뷰)
6. [보안](#6-보안)
7. [문서 & 유틸리티](#7-문서--유틸리티)
8. [팀 & 회고](#8-팀--회고)

---

## 1. QA & 테스팅

### `/browse` — 헤드리스 브라우저

사이트를 탐색하고 스크린샷을 찍고 요소와 상호작용합니다.

```
# 기본 탐색
/browse http://localhost:3000

# 스크린샷 촬영
/browse http://localhost:3000 screenshot

# 요소 클릭
/browse http://localhost:3000 click #login-button

# 폼 입력
/browse http://localhost:3000 fill #email test@example.com

# 반응형 테스트
/browse http://localhost:3000 --viewport 375x812
```

**AllCrew 예시:**
```
사용자: /browse http://localhost:3000
→ 대시보드 로그인 페이지 열림, 스크린샷 캡처
→ "로그인 폼이 보입니다. 이메일과 비밀번호 필드, 카카오 로그인 버튼이 있습니다."
```

---

### `/qa` — QA 테스트 + 자동 수정

사이트를 체계적으로 테스트하고 발견된 버그를 소스코드에서 수정합니다.

```
# 전체 QA (기본)
/qa http://localhost:3000

# 빠른 QA (심각한 버그만)
/qa http://localhost:3000 --quick

# 철저한 QA (미세한 이슈까지)
/qa http://localhost:3000 --exhaustive

# 특정 페이지만
/qa http://localhost:3000 --scope dashboard
```

**AllCrew 예시:**
```
사용자: /qa http://localhost:3000
→ 로그인, 대시보드, 스태프 목록, 프로젝트 CRUD 순회
→ 발견: "스태프 목록 페이지에서 빈 상태일 때 에러 표시 안 됨"
→ 자동 수정: staff/page.tsx에 empty state 컴포넌트 추가
→ 커밋: "fix: add empty state for staff list page"
→ 재검증 스크린샷 촬영
```

**결과 예시:**
```
QA Report — localhost:3000
━━━━━━━━━━━━━━━━━━━━━━
Health Score: 7.2/10 → 8.8/10 (after fixes)

Fixed (3):
  ✅ Empty state missing on /staff
  ✅ Mobile nav overflow on /projects
  ✅ Console error on /dashboard chart

Remaining (1):
  ⚠️ Low: favicon missing (cosmetic)

Ship-ready: YES
```

---

### `/qa-only` — QA 리포트만 (수정 없음)

```
/qa-only http://localhost:3000
```

수정하지 않고 리포트와 스크린샷만 생성합니다. PM이나 디자이너에게 공유할 때 유용.

---

### `/benchmark` — 성능 벤치마크

```
# 기준선 캡처 (변경 전)
/benchmark http://localhost:3000 --baseline

# 변경 후 비교
/benchmark http://localhost:3000

# 빠른 체크
/benchmark http://localhost:3000 --quick

# 특정 페이지만
/benchmark http://localhost:3000 --pages /,/dashboard,/staff
```

**측정 항목:** TTFB, FCP, LCP, DOM 완료 시간, 번들 크기

**회귀 기준:**
- 타이밍 50%↑ 또는 500ms↑ = REGRESSION
- 번들 25%↑ = REGRESSION

---

### `/canary` — 배포 후 모니터링

```
# 배포 후 5분간 모니터링
/canary https://allcrew.example.com --duration 5m

# 30초 간격 체크
/canary https://allcrew.example.com --interval 30s

# 특정 페이지만
/canary https://allcrew.example.com --pages /,/dashboard
```

콘솔 에러, 성능 저하, 페이지 장애를 감시합니다.

---

## 2. 배포 워크플로우

### `/ship` — 테스트→리뷰→PR 생성

전체 배포 파이프라인을 한 커맨드로 실행합니다.

```
# 전체 워크플로우
/ship

# 테스트 스킵
/ship --skip-tests

# 버전 명시
/ship --bump minor
```

**워크플로우:**
1. 베이스 브랜치 감지 & 머지
2. 테스트 실행
3. 디프 리뷰
4. VERSION 파일 범프
5. CHANGELOG 업데이트
6. 커밋 & 푸시
7. PR 생성

**AllCrew 예시:**
```
사용자: (feature/staff-filter 브랜치에서)
/ship

→ main 브랜치 머지 확인
→ npm run test 실행 (12 passed)
→ 디프 리뷰: "staff 필터 기능 추가, 3파일 변경"
→ VERSION: 1.2.0 → 1.3.0
→ CHANGELOG 업데이트
→ PR 생성: "feat: add staff list filtering"
→ PR URL: https://github.com/org/allcrew/pull/42
```

---

### `/review` — PR 코드 리뷰

```
# 현재 브랜치 리뷰
/review

# 특정 브랜치 리뷰
/review feature/payroll
```

**검토 항목:**
- SQL 안전성 (인젝션, N+1, 트랜잭션)
- LLM 신뢰 경계 위반
- 조건부 사이드 이펙트
- 프로덕션 안전성
- 성능 우려사항

---

### `/land-and-deploy` — PR 머지→배포→검증

```
/land-and-deploy
```

`/ship`으로 PR을 만든 후 사용합니다.

**워크플로우:**
1. PR 머지
2. CI 대기
3. 배포 대기
4. 카나리 체크
5. 프로덕션 건강 확인

---

### `/setup-deploy` — 배포 설정

```
/setup-deploy
```

배포 플랫폼(Fly.io, Vercel, Netlify 등)을 자동 감지하고 설정합니다.

---

## 3. 디버깅 & 코드 품질

### `/investigate` — 체계적 버그 조사

근본 원인을 찾을 때까지 수정하지 않습니다 (Iron Law).

```
사용자: 대시보드 차트가 안 나와요
/investigate
```

**4단계 프로세스:**
1. **Investigate** — 증거 수집, 재현
2. **Analyze** — 로그, 스택 트레이스 분석
3. **Hypothesize** — 근본 원인 가설
4. **Implement** — 수정 & 검증

**AllCrew 예시:**
```
사용자: /attendance 페이지에서 500 에러가 나요
/investigate

→ Phase 1: 브라우저에서 /attendance 접속, 500 확인
→ Phase 2: backend 로그 확인 — Prisma query 에러
→ Phase 3: 가설 — attendance 테이블에 새 컬럼 마이그레이션 누락
→ Phase 4: prisma migrate 실행, 서버 재시작, 200 확인
```

---

### `/health` — 코드 품질 대시보드

```
# 전체 건강 체크
/health

# 요약만
/health --quick

# 트렌드 보기
/health --trend
```

**측정 항목 (가중 점수 0-10):**
- TypeScript 타입 체크
- ESLint 린트
- 테스트 커버리지
- 데드 코드
- 쉘 스크립트 품질

**AllCrew 예시:**
```
사용자: /health

Code Health Dashboard — allcrew
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Overall: 7.4/10

  TypeScript   8/10  ██████████░░  3 errors
  ESLint       9/10  ████████████  clean
  Tests        5/10  ██████░░░░░░  42% coverage
  Dead Code    8/10  ██████████░░  2 unused exports
```

---

### `/simplify` — 코드 개선

변경된 코드를 리뷰하고 재사용성, 품질, 효율성 문제를 수정합니다.

```
사용자: (코드 수정 후)
/simplify
```

---

## 4. 디자인

### `/design-consultation` — 디자인 시스템 컨설팅

프로젝트에 맞는 디자인 시스템을 제안하고 DESIGN.md를 생성합니다.

```
/design-consultation
```

**생성물:**
- DESIGN.md (디자인 시스템 문서)
- 폰트 프리뷰 페이지
- 컬러 팔레트 프리뷰
- 타이포그래피 가이드라인

---

### `/design-shotgun` — 디자인 변형 비교

여러 디자인 변형을 생성하고 비교합니다.

```
# 기본 (3개 변형)
/design-shotgun

# 5개 변형 생성
/design-shotgun --count 5
```

**AllCrew 예시:**
```
사용자: 대시보드 레이아웃 옵션 보여줘
/design-shotgun

→ Variant A: 카드 그리드 (2x3)
→ Variant B: 사이드바 + 메인 차트
→ Variant C: 풀와이드 타임라인
→ 비교 보드 생성, 피드백 수집
```

---

### `/design-html` — 프로덕션급 HTML/CSS 생성

```
# 설명으로 생성
/design-html 스태프 프로필 카드 컴포넌트

# 기존 디자인 기반
/design-html   (이전 /design-shotgun 승인 후)
```

30KB 오버헤드, 외부 의존성 없는 순수 HTML/CSS를 생성합니다.

---

### `/design-review` — 시각적 QA + 자동 수정

```
/design-review http://localhost:3000

# 고우선순위만
/design-review http://localhost:3000 --quick
```

**검출 항목:**
- 시각적 불일치
- 간격/정렬 문제
- 계층 구조 문제
- AI 슬롭 패턴 (AI가 만든 뻔한 디자인)
- 느린 인터랙션

---

### `/frontend-design` — 프론트엔드 UI 제작

프로덕션급 프론트엔드 인터페이스를 생성합니다.

```
사용자: 랜딩 페이지 만들어줘
/frontend-design
```

---

## 5. 플랜 리뷰

구현 전 계획을 다각도로 검토합니다.

### `/plan-ceo-review` — CEO/창업자 관점

```
/plan-ceo-review
```

**모드:**
- **SCOPE EXPANSION** — 더 크게 생각
- **SELECTIVE EXPANSION** — 범위 유지 + 선별 확장
- **HOLD SCOPE** — 최대 엄격함
- **SCOPE REDUCTION** — 핵심만 남김

**AllCrew 예시:**
```
사용자: (급여 정산 자동화 계획 작성 후)
/plan-ceo-review

→ "급여 정산만이 아니라, 계약→근무→정산→정산서 발급까지
   엔드투엔드 자동화가 10-star 경험입니다."
→ 모드 선택: SELECTIVE EXPANSION
→ 계약서 자동 연동만 추가, 나머지 범위 유지
```

---

### `/plan-eng-review` — 엔지니어링 관점

```
/plan-eng-review
```

아키텍처, 데이터 플로우, 엣지 케이스, 테스트 커버리지, 성능을 검토합니다.

---

### `/plan-design-review` — 디자인 관점

```
/plan-design-review
```

각 디자인 차원을 0-10으로 평가하고, 10점이 되려면 무엇이 필요한지 설명합니다.

---

### `/plan-devex-review` — 개발자 경험 관점

```
/plan-devex-review
```

개발자 페르소나, 경쟁사 벤치마크, 마법 같은 순간 설계를 검토합니다.

---

### `/autoplan` — 전체 자동 리뷰

```
/autoplan
```

CEO → 디자인 → 엔지니어링 → DX 리뷰를 순차 실행합니다. 판단이 필요한 결정만 최종 승인 게이트에서 표시합니다.

---

## 6. 보안

### `/cso` — 보안 감사 (Chief Security Officer)

```
/cso
```

**검사 항목:**
- 시크릿 아키올로지 (하드코딩된 키, .env 누출)
- 의존성 공급망 보안
- CI/CD 파이프라인 보안
- OWASP Top 10
- STRIDE 위협 모델링
- LLM/AI 보안

**AllCrew 예시:**
```
사용자: /cso

→ ⚠️ HIGH: backend/.env.example에 실제 JWT_SECRET 값 노출
→ ⚠️ MED: axios 4.1.2에 CVE-2024-XXXX 취약점
→ ✅ PASS: SQL 인젝션 — Prisma 파라미터 바인딩 사용 중
→ ✅ PASS: XSS — React 자동 이스케이프
```

---

### `/careful` — 위험 명령 경고

```
/careful
```

`rm -rf`, `DROP TABLE`, `git push --force`, `git reset --hard` 등 실행 전 경고합니다.

---

### `/guard` — 최대 안전 모드

```
/guard
```

`/careful` + `/freeze`를 결합합니다. 프로덕션 작업 시 권장.

---

### `/freeze` & `/unfreeze` — 편집 범위 제한

```
# backend 디렉토리만 편집 허용
/freeze backend/

# 제한 해제
/unfreeze
```

디버깅 시 관련 없는 코드를 실수로 수정하는 것을 방지합니다.

---

## 7. 문서 & 유틸리티

### `/document-release` — 배포 후 문서 업데이트

```
/document-release
```

README, ARCHITECTURE, CONTRIBUTING, CLAUDE.md, CHANGELOG을 배포 내용에 맞게 업데이트합니다.

---

### `/mermaid` — 다이어그램 생성

```
사용자: AllCrew 백엔드 모듈 관계를 다이어그램으로 그려줘
/mermaid
```

플로우차트, 시퀀스, 클래스, ER, 간트 등 18+ 다이어그램 타입을 지원합니다.

---

### `/pdf`, `/pptx`, `/docx` — 오피스 파일

```
# PDF 읽기/생성/병합
/pdf

# 프레젠테이션 생성
/pptx

# Word 문서 생성
/docx
```

---

### `/codex` — OpenAI Codex 세컨드 오피니언

```
# 코드 리뷰
/codex review

# 적대적 테스트
/codex challenge

# 질문
/codex 이 아키텍처의 약점은?
```

---

### `/checkpoint` — 작업 상태 저장

```
# 현재 상태 저장
/checkpoint

# 이전 상태 복원
/checkpoint --resume
```

브랜치 전환이나 컨텍스트 스위칭 시 유용합니다.

---

## 8. 팀 & 회고

### `/retro` — 주간 회고

```
# 주간 회고
/retro

# 팀원별 분석 포함
/retro --team
```

커밋 히스토리, 작업 패턴, 코드 품질 메트릭을 분석합니다.

---

### `/learn` — 학습 내용 관리

```
# 전체 보기
/learn

# 검색
/learn --search prisma

# 오래된 것 정리
/learn --prune
```

---

### `/office-hours` — YC 오피스아워

```
/office-hours
```

**Startup 모드:** 수요 현실, 현상 유지, 가장 좁은 쐐기 등 6가지 질문
**Builder 모드:** 사이드 프로젝트/해커톤용 디자인 씽킹

---

## 워크플로우 조합 예시

### 기능 개발 전체 사이클

```
1. /office-hours          # 아이디어 검증
2. (계획 작성)
3. /autoplan              # CEO+디자인+엔지니어링 리뷰
4. (구현)
5. /health                # 코드 품질 확인
6. /qa http://localhost:3000  # QA + 자동 수정
7. /ship                  # PR 생성
8. /land-and-deploy       # 머지 + 배포
9. /canary https://prod   # 배포 후 모니터링
10. /document-release     # 문서 업데이트
```

### 버그 수정 사이클

```
1. /investigate           # 근본 원인 분석
2. (수정)
3. /simplify              # 코드 개선
4. /qa http://localhost:3000  # 회귀 테스트
5. /ship                  # PR 생성
```

### 보안 점검

```
1. /cso                   # 전체 보안 감사
2. /guard                 # 안전 모드 활성화
3. (수정)
4. /review                # 코드 리뷰
```

---

## 설정

gstack 설정은 `~/.gstack/config.yaml`에 있습니다:

```yaml
skill_prefix: false     # /qa (true면 /gstack-qa)
proactive: true         # 컨텍스트 기반 자동 제안
telemetry: anonymous    # anonymous | community | off
auto_upgrade: false     # 자동 업그레이드
```

업그레이드:
```
/gstack-upgrade
```
