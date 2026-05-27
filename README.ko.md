# zzang-claude-session

[English](README.md) · [한국어](README.ko.md)

Claude Code의 크로스 세션 컨텍스트 저장소 — [zzang-claude-skills](https://github.com/Kimseungzzang/kimseungzzang-claude-skills)의 일부.

세션과 머신을 넘나드며 대화 컨텍스트를 영속화합니다. 실시간 툴 사용 로그가 포함되어 있어 토큰 소진, 앱 크래시 등 중간 중단 상황에서도 복구 가능합니다.

## 설정

**1. 스킬 설치** (모든 것을 자동으로 처리):

```bash
npx zzang-claude-skills
```

인스톨러가:
- `/session-save`와 `/session-load` 스킬 설치
- `PostToolUse` 훅 (`task-log.sh`) 설치 및 등록
- private 세션 레포 URL 요청 (없으면 생성 안내)

**2. 사용:**

```
# 세션 종료 전 (또는 작업 마일스톤마다)
/session-save

# 새 세션 시작 시 제일 먼저
/session-load
```

> 새 머신: `npx zzang-claude-skills` → `/session-load` — 세션 레포를 자동으로 clone합니다.

---

## 작동 방식

### 전체 흐름

```
매 툴 사용  →  PostToolUse 훅  →  task-log.md  (로컬 전용)
                                          │
                                 /session-save
                                          │
                      ┌───────────────────┼───────────────────┐
                      ▼                   ▼                   ▼
                task-log 읽기     스냅샷 작성            CURRENT.ctx 업데이트
                (흡수 후 삭제)    (타임스탬프)           (누적 상태)
                      │
                      └──── git commit & push ──→  GitHub (private 레포)
                                                           │
                                                   /session-load
                                                           │
                                          git pull → CURRENT.ctx 읽기
                                                   + task-log 확인
                                                           │
                                                   이어서 작업 안내
```

### `/session-save` 플로우

```
[1] ~/.claude/zzang-ctx가 git 레포인지 확인 (없으면 clone)
         │
[2] git pull --rebase
         │
[3] 이번 세션에서 작업한 프로젝트 감지
    └── git rev-parse --show-toplevel | xargs basename
    └── 대화에서 다른 프로젝트 경로 스캔
    └── 목록 확인
         │
[4] 각 프로젝트별:
    ├─► task-log.md 읽기  ← DONE/CHANGED 채우기, 마지막 타임스탬프를 TASK-LOG-ID로 기록
    └─► CURRENT.ctx 읽기  ← 누적 히스토리 로드
         │
[5] 타임스탬프 스냅샷 작성
    ~/.claude/zzang-ctx/{project}/YYYY-MM-DDTHH-MM
         │
[6] CURRENT.ctx에 병합  (아래 병합 규칙 참고)
         │
[7] task-log.md 삭제  (흡수 완료 — 다음 작업은 새 로그 시작)
         │
[8] git add . && git commit && git push
         │
[9] 완료 보고 ✅
```

### `/session-load` 플로우

```
[1] ~/.claude/zzang-ctx가 git 레포인지 확인 (없으면 clone)
         │
[2] git pull --rebase
         │
[3] 프로젝트 이름 감지
         │
[4] CURRENT.ctx 읽기
         │
[5] task-log vs TASK-LOG-ID 비교
         │
    ┌────┴──────────────────────────────────────────────────────┐
    │                                                           │
Case 1: ID 일치                         Case 4: task-log 없음
→ 정상 상태                             → 다른 머신
→ 정상적으로 진행                       → CURRENT.ctx만 사용, 사용자에게 안내
    │                                                           │
Case 2: task-log가 SAVED_ID보다 최신    Case 3: task-log 헤더 > SESSION
→ 미저장 작업 존재                      → 새 작업 진행 중 (중단 아님)
→ 미저장 항목 표시, 재개 여부 확인     → task-log를 현재 작업 컨텍스트로 표시
    │                                          │
    └──────────────────┬───────────────────────┘
                       │
[6] 간략한 안내 출력 (150단어 이내)
```

### PostToolUse 훅 — `task-log.sh`

매 툴 사용 후 자동 실행. `task-log.md`에 한 줄씩 추가:

```
## 2026-05-28T14:00 | my-app          ← 세션 첫 항목에 생성
[14:01] Write      src/api/webhooks/stripe.ts
[14:03] Bash       npm test
[14:05] Edit       src/middleware/idempotency.ts
```

- **로컬 전용** — GitHub에 직접 push되지 않음
- **크래시에도 기록됨** — Claude가 응답하기 전에 기록되므로 토큰 소진 시에도 생존
- **`/session-save`가 흡수** — CURRENT.ctx에 병합 후 삭제
- **`/session-load`가 읽음** — `TASK-LOG-ID`와 비교해서 중단 지점 감지

---

## 저장소 레이아웃

```
~/.claude/
├── commands/
│   ├── session-save.md       ← 스킬 정의
│   └── session-load.md
├── scripts/
│   └── task-log.sh           ← PostToolUse 훅
├── settings.json             ← 훅 등록
├── zzang-ctx-remote          ← GitHub 레포 URL
└── zzang-ctx/                ← git 레포 (GitHub 연동)
    ├── my-app/
    │   ├── CURRENT.ctx       ← 누적 상태  (GitHub에 push됨)
    │   ├── task-log.md       ← 실시간 로그 (로컬 전용, push 안 됨)
    │   └── 2026-05-28T14-00  ← 타임스탬프 스냅샷 (push됨)
    └── other-project/
        └── CURRENT.ctx
```

---

## CURRENT.ctx 형식

150–300 토큰. 전체 대화를 다시 보지 않고도 빠르게 컨텍스트를 복원합니다.

```
SESSION 2026-05-28T14:00 | /Users/kim/my-app | feat/payments
TASK-LOG-ID: 14:05
STACK: Next.js TypeScript PostgreSQL Stripe Redis
DONE: Stripe 웹훅 핸들러; 멱등성 미들웨어; 주문 상태 폴링
CHANGED: src/api/webhooks/stripe.ts(new); src/middleware/idempotency.ts(new); src/lib/order.ts
TRIED: 웹소켓으로 주문 상태(모바일 백그라운드 시 연결 끊김); 엣지 런타임에서 Stripe 웹훅(crypto 사용 불가)
DECIDED: 웹소켓 대신 폴링: 더 단순하고 모든 클라이언트에서 동작; Redis에 멱등성 키: TTL이 네이티브, DB 스키마 변경 불필요
TODO: 분쟁 웹훅 처리 | 웹훅 재시도 UI 추가 | 멱등성 동시 요청 부하 테스트
OPEN: 고액 주문에 Stripe Radar vs 커스텀 사기 탐지 규칙
CTX: STRIPE_WEBHOOK_SECRET 수동 설정 필요 (.env.example에 없음); DB 스키마 Q3까지 동결; Redis TTL Stripe 권장 24h
```

**병합 규칙:**

| 필드 | 동작 |
|------|------|
| `SESSION` | 항상 최신 타임스탬프로 업데이트 |
| `TASK-LOG-ID` | 마지막 흡수된 task-log 라인 시간으로 교체 |
| `TRIED` | **누적** — 같은 실수를 반복하지 않도록 |
| `DECIDED` | **누적** — 결정의 근거를 보존 |
| `CTX`, `OPEN` | **누적** |
| `DONE`, `CHANGED` | 매 세션 교체 (히스토리는 스냅샷 파일에 유지) |
| `TODO` | 완료된 항목 자동 제거; 새 항목 추가 |

> `TRIED`와 `DECIDED`는 **절대 압축하지 않습니다** — 연속성에 가장 중요한 필드입니다.

---

## 멀티머신 워크플로우

```
머신 A                             머신 B
──────                             ──────
/session-save                      npx zzang-claude-skills
  → task-log 흡수                    → zzang-ctx-remote에서 clone
  → CURRENT.ctx 업데이트           /session-load
  → GitHub에 push                    → GitHub에서 CURRENT.ctx pull
                                     → task-log 없음 (Case 4 — 정상)
                                     → 저장된 상태에서 재개
```

---

## 여러 프로젝트를 한 세션에서 작업한 경우

`/session-save`가 세션 중 작업한 모든 프로젝트를 감지하고, 각 프로젝트 폴더에 **필터링된 컨텍스트**를 별도로 저장합니다 — 프로젝트 간 컨텍스트 혼재 없음.

---

## ⚠️ 주의사항

**세션 레포는 반드시 private으로 설정하세요.**
CURRENT.ctx에는 파일 경로, 설계 결정, API 키 이름, 내부 아키텍처 세부사항이 담깁니다. 절대 public 레포를 사용하지 마세요.

**`~/.claude/zzang-ctx/`를 수동으로 삭제하지 마세요.**
`task-log.md`가 여기에 있으며 로컬 전용입니다. `/session-save` 전에 폴더를 지우면 흡수되지 않은 항목이 영구적으로 사라집니다.

**CURRENT.ctx를 직접 편집하지 마세요.**
병합 로직이 특정 형식을 기대합니다. 수동 편집 시 필드 누적이 조용히 깨질 수 있습니다.

**`/session-save` 도중 중단되면 다시 실행하세요.**
스냅샷은 멱등성이 있어 덮어써도 문제없습니다.

**task-log가 로컬 전용인 것은 의도적 설계입니다.**
GitHub에 커밋되지 않습니다. 다른 머신에서 Case 4로 분류되는 건 에러가 아닌 정상 동작입니다.

---

## 💡 권장사항

**세션 끝뿐만 아니라 마일스톤마다 저장하세요.**
기능 완성 후, 다음 작업 시작 전에 중간 저장하면 복구 지점이 세밀해집니다.

**새 세션을 시작하면 제일 먼저 `/session-load`를 실행하세요.**
로드 전에 작업을 시작하면 Claude가 누적된 컨텍스트 없이 동작해서 동일한 결정을 반복하거나 알려진 제약사항을 놓칠 수 있습니다.

**머신을 바꾸기 전에 반드시 `/session-save`를 먼저 실행하세요.**
다른 머신은 GitHub에서 pull하므로 push하지 않으면 오래된 컨텍스트를 보게 됩니다.

**세션 전용 레포를 따로 만드세요.**
기존 레포를 재사용하지 마세요. 세션 레포는 프로젝트별 CURRENT.ctx가 시간이 지남에 따라 쌓이므로 전용 레포가 관리하기 깔끔합니다.

---

## 데모

실제 스냅샷과 CURRENT.ctx 예시는 [`demo/my-app/`](./demo/my-app/)을 참고하세요.
