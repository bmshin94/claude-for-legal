# claude-for-legal 분석 정리 (한국어)

> 이 문서는 `claude-for-legal` 레포지토리를 직접 열어보며 분석한 내용과,
> 활용 방안 · 수익화 아이디어를 정리한 기록입니다.

## 📌 레포지토리 정보

| 항목 | 내용 |
|---|---|
| **이 레포 (fork)** | https://github.com/bmshin94/claude-for-legal |
| **원본 (upstream)** | https://github.com/anthropics/claude-for-legal |
| **소유자** | Anthropic (marketplace.json `owner.name`) |
| **라이선스** | Apache License 2.0 (상업적 이용·수정·재배포 가능) |
| **저작권 표기** | `Copyright 2026 Anthropic PBC` |
| **작성 브랜치** | `claude/relaxed-wozniak-nt18m6` |

---

## 1. 이게 뭐하는 건가?

### 한 줄 정의

**변호사·법무팀을 위한 Claude 플러그인 마켓플레이스.**
애플리케이션이나 라이브러리가 아니라, **Claude에게 "법무 전문가 역할"을 가르치는
프롬프트/설정 묶음**이다. 실제 코드 비율은 매우 낮고 대부분이 마크다운이다.

### 실측 규모

| 항목 | 개수 |
|---|---|
| 플러그인 (마켓플레이스 등록) | 13개 (자체 12 + 벤더 1) |
| `SKILL.md` 스킬 파일 | 151개 |
| 서브에이전트 정의 (`agents/*.md`) | 10개 |
| 매니지드 에이전트 쿡북 | 5개 |
| MCP 커넥터 | 19종 |

### 디렉터리 구조

```
.claude-plugin/marketplace.json   # 마켓플레이스 카탈로그 (13개 플러그인)
commercial-legal/                 # 상업계약 (스킬 12)
privacy-legal/                    # 개인정보 (9)
product-legal/                    # 프로덕트 법무 (7)
corporate-legal/                  # M&A·기업법무 (13)
employment-legal/                 # 노동법무 (20)
regulatory-legal/                 # 규제대응 (9)
ai-governance-legal/              # AI 거버넌스 (10)
litigation-legal/                 # 소송 (19)
ip-legal/                         # 지식재산 (12)
legal-clinic/                     # 로스쿨 클리닉 (16)
law-student/                      # 로스쿨 학생 (13)
legal-builder-hub/                # 커뮤니티 스킬 탐색/설치 (10)
external_plugins/cocounsel-legal/ # Thomson Reuters (Westlaw) 벤더 플러그인
managed-agent-cookbooks/          # API 배포용 스케줄 에이전트 5종
scripts/                          # validate.py, lint-tool-scope.py, orchestrate.py,
                                  # deploy-managed-agent.sh, test-cookbooks.sh
references/                       # company-profile / dashboard 공용 템플릿
```

### 플러그인 하나의 내부 (예: `commercial-legal`)

```
commercial-legal/
├── .claude-plugin/plugin.json   # 이름, 버전(1.0.2), 설명, 저자
├── .mcp.json                    # Ironclad, DocuSign, iManage, TopCounsel,
│                                #   Definely, Slack, Google Drive
├── CLAUDE.md                    # 업무 프로필 "템플릿"
├── skills/                      # 12개 (nda-review, review, renewal-tracker,
│                                #   amendment-history, escalation-flagger ...)
├── agents/                      # renewal-watcher, deal-debrief, playbook-monitor
└── hooks/hooks.json             # 현재 `{"hooks": {}}` 빈 스텁
```

---

## 2. 핵심 설계 사상

### ① Cold-start Interview (콜드스타트 인터뷰) — 이 레포의 심장

설치 후 `/<plugin>:cold-start-interview`를 실행하면 Claude가 10~15분간 인터뷰를 진행하고,
그 결과를 **버전 독립 경로**에 저장한다.

```
~/.claude/plugins/config/claude-for-legal/<plugin>/CLAUDE.md
```

이후 **151개 스킬 전부가 이 파일을 먼저 읽고** 동작한다.
즉 "일반적인 법률 AI"가 아니라 **"우리 팀 플레이북대로 판단하는 AI"** 가 된다.

### ② 설정이 없으면 작동 거부

`<plugin>/CLAUDE.md` 실제 규칙:

> 설정 파일이 없거나 `[PLACEHOLDER]`가 남아 있으면 **실질적 작업 전에 STOP**.
> 기본값으로 진행하지 말 것.

`nda-review` 스킬의 가드:

> **GREEN(서명 진행)은 변호사가 검토한 플레이북 없이 발급 불가.**
> 기본값으로 GREEN을 주는 것은, 비변호사가 정한 기준을 다음 비변호사가
> 신뢰하게 만드는 것이다.

### ③ 법률 도메인 특유의 3중 안전장치

1. **Destination check** — 출력 전 "특권(privilege) 범위 안인가?" 확인.
   공개 채널·상대방 변호사에게 보내면 특권이 깨지므로 특권본/정제본/양쪽 중 선택 유도.
2. **Scope check** — "NDA"라는 이름이지만 스탠드스틸·라이선스 부여·경업금지·IP 양도가
   들어있으면 **무조건 YELLOW 강제**.
3. **Matter workspace** — `Cross-matter context`가 off면 **다른 사건 파일 접근 금지**.
   정보 차단벽(ethical wall)을 규칙으로 구현.

### ④ 하나의 소스, 두 가지 배포

```
        같은 시스템 프롬프트 · 같은 스킬
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
  Claude Code / Cowork      Managed Agents API
  (로컬, 대화형)             (서버, 스케줄 자동실행)
```

`managed-agent-cookbooks/docket-watcher/agent.yaml`:

```yaml
system:
  file: ../../litigation-legal/agents/docket-watcher.md   # 플러그인 프롬프트 재사용
  append: |
    You are running headless behind the platform team's workflow engine.
    Produce files in ./out/; do not assume an interactive session.
```

### ⑤ 최소 권한 원칙 (CI에서 강제)

`scripts/lint-tool-scope.py`가 검증하는 규칙:

- **오케스트레이터** = `read`, `grep`, `glob`만 (쓰기 권한 없음)
- **MCP·쓰기 권한** = 특정 서브에이전트 리프 하나만

```yaml
tools:
  - type: agent_toolset_20260401
    default_config: { enabled: false }   # 전부 기본 OFF
    configs:
      - { name: read, enabled: true }    # 필요한 것만 ON
      - { name: grep, enabled: true }
      - { name: glob, enabled: true }

callable_agents:
  - { manifest: ./subagents/tracker-writer.yaml }   # only leaf with Write
```

### ⑥ 프롬프트 인젝션 방어 5계층

`scripts/orchestrate.py` docstring에 신뢰도 순으로 명시:

| 순위 | 방어 | 신뢰도 |
|---|---|---|
| 1 | 닫힌 스키마 인텐트 (enum) — 자유 텍스트 통과 금지 | **주 방어** |
| 2 | 타겟 에이전트 화이트리스트 | **주 방어** |
| 3 | `<agent-handoff>` 데이터 프레임 래핑 | 보조 |
| 4 | 명령어 문구 denylist | **낮음 — "믿지 말 것"** |
| 5 | `handoff-audit.jsonl` 감사 로그 | 사후 검증 |

> 원문: "denylists for prompt injection are trivially bypassed. Do not rely on it."
> 자기 방어책의 한계를 스스로 문서화한 점이 인상적.

---

## 3. 언제 쓰는가?

| 대상 | 활용 |
|---|---|
| **사내 법무팀** | 벤더 계약 리뷰, NDA 신호등 분류, 갱신·해지 기한 추적, DSAR, PIA, AI 사용사례 심사 |
| **로펌** | M&A 실사 그리드(셀마다 인용), 클레임 차트, 연표, 증언 준비, 특권 로그 1차 검토 |
| **로스쿨** | 소크라테스 문답, IRAC 채점, 케이스 브리프, 아웃라인, 콜드콜 대비, 변시 준비 |
| **자동화** | 규제 다이제스트, 법원 docket 감시, 계약 갱신 알림, 출시 레이더 |

---

## 4. Q&A 정리

### Q1. 설치 및 사용법

```bash
git clone https://github.com/bmshin94/claude-for-legal.git
cd claude-for-legal
claude
```

Claude Code 안에서:

```
/plugin marketplace add /경로/claude-for-legal
/plugin install commercial-legal@claude-for-legal
  ⚠️ 여기서 user scope 선택
  ⚠️ Claude Code 완전히 재시작 (필수)
/commercial-legal:cold-start-interview
/commercial-legal:review 계약서.pdf
```

**자주 걸리는 함정 (QUICKSTART.md 기준)**

| 증상 | 원인 | 해결 |
|---|---|---|
| Command not found | 재시작 안 함 | Claude Code 재시작 |
| "I can't read [파일]" | project scope 설치 | user scope로 재설치 |
| 인용에 `[verify]` | 리서치 MCP 미연결 | CourtListener 등 연결 |

### Q2. 플러그인? 스킬? MCP?

**전부 맞다. 플러그인이 스킬과 MCP를 담는 그릇이다.**

| | 스킬 | MCP | 플러그인 |
|---|---|---|---|
| 정체 | 마크다운 지시서 | 서버 프로토콜(HTTPS) | 배포 패키지 |
| 역할 | 어떻게 일하나 | 어디서 데이터를 가져오나 | 무엇을 배포하나 |
| 이 레포 | 151개 | 19종 | 13개 |

스킬 파일은 **코드가 없는 순수 마크다운**이다. 그래서 비개발자(변호사)도 직접 수정 가능.
`user-invocable: false`가 붙은 9개는 내부 전용 스킬로 명령어 목록에서 숨겨진다.

### Q3. API 토큰이 필요한가?

| 방식 | Anthropic API 키 | 인증 | 비용 |
|---|---|---|---|
| Claude Code | ❌ 불필요 | 구독 로그인 | 정액 구독 |
| Claude Cowork | ❌ 불필요 | 구독 로그인 | 정액 구독 |
| Managed Agents API | ✅ **필수** | `ANTHROPIC_API_KEY` | 토큰 종량제 |

근거 (`scripts/deploy-managed-agent.sh`):

```bash
API="${ANTHROPIC_API_BASE:-https://api.anthropic.com}"
[[ $DRY_RUN -eq 1 ]] || : "${ANTHROPIC_API_KEY:?ANTHROPIC_API_KEY must be set}"
curl -sS -H "x-api-key: $ANTHROPIC_API_KEY" ...
```

**MCP 커넥터는 별도 인증.** `.mcp.json`에는 URL만 있고 키는 없다(최초 사용 시 OAuth).
Ironclad·DocuSign·iManage·Everlaw 등은 해당 서비스 유료 계정이 필요하다.
커넥터 없이도 동작하지만 인용에 `[verify]` 표시가 붙는다.

### Q4. 왜 GitHub에서 유명한가?

1. **Anthropic 1st-party 레포** — Claude를 만든 회사의 모범 답안
2. **플러그인 시스템 레퍼런스 구현** — 불변식 I1~I11 문서화
   (I1 알파벳 정렬, I3 설명 10~2000자, I9 경로에 `..`·쉘 메타문자 금지,
   I10 숨은 유니코드 금지, I11 name 정규식 `^[a-z0-9][a-z0-9-]{1,63}$`)
3. **151개 프로덕션 프롬프트 전면 공개** — 규모가 희귀함
4. **LegalTech 시장 규모** — 연 300억 달러+, Harvey AI 밸류 30억 달러
5. **업계 벤더 결집** — Thomson Reuters(Westlaw), Ironclad, DocuSign, iManage,
   Everlaw, Trellis, Definely 등
6. **AI 안전 설계 교과서** — 최소권한 + 인젝션 5계층 + 자기 한계 명시

### Q5. 로컬 에이전트 구축에 도움이 되는가 → 매우 도움됨

바로 이식 가능한 패턴 8가지:

| # | 패턴 | 핵심 |
|---|---|---|
| 1 | **설정 영속화** | 인터뷰 1회 → 버전 독립 경로 저장 → 모든 스킬이 먼저 읽음 |
| 2 | **계층형 설정 상속** | company-profile → plugin CLAUDE.md → matter.md (CSS cascade 구조) |
| 3 | **최소 권한** | 기본 OFF + 필요한 것만 ON, 쓰기는 리프 하나만, CI 린터로 강제 |
| 4 | **명시적 게이트** | 돌이킬 수 없는 행동 앞에 승인 게이트 |
| 5 | **모르면 묻고 저장** | 물어본 답을 설정에 기록 → 다음부터 일관 적용 |
| 6 | **인젝션 방어 5계층** | 닫힌 enum + 화이트리스트가 주 방어, denylist는 보조 |
| 7 | **한 소스 두 배포** | 같은 프롬프트 + 환경별 `append` |
| 8 | **스킬 가시성 제어** | `user-invocable: false`로 내부 스킬 숨김 |

**그대로 이식 가능한 도메인**

| 도메인 | 대응 플러그인 |
|---|---|
| 세무/회계 | commercial-legal |
| 의료 행정 | privacy-legal (HIPAA→의료법) |
| HR/인사 | employment-legal |
| 부동산 | corporate-legal |
| 보험 손해사정 | litigation-legal |
| 건설 감리 | regulatory-legal |
| DevOps SRE | 쿡북 5종 |

### Q6. React / PHP로 만들 수 있는가?

**레포 자체를 포팅하는 것은 무의미하다** (실체가 마크다운이라 옮길 코드가 없음).
대신 **이를 감싸는 웹 애플리케이션**을 만드는 것이 정답이고, 그것이 수익화의 핵심이다.

```
React / Next.js (프론트)
  - 문서 업로더, 신호등 대시보드, 플레이북 설정 위저드, 리뷰 이력
        │ REST / SSE
PHP Laravel · Node · FastAPI (백엔드)
  - 인증/멀티테넌시/과금
  - 플레이북 DB → CLAUDE.md 포맷 렌더
  - 스킬 마크다운 로드 → 프롬프트 조립
  - Anthropic API 호출 (x-api-key)
  - 감사 로그, 스케줄러
        │ HTTPS
Anthropic Messages / Managed Agents API
```

```php
// 개념 코드 (Laravel)
$skillMd   = file_get_contents(base_path("skills/{$skill}/SKILL.md"));
$profileMd = $playbook->renderAsMarkdown();

Http::withHeaders([
    'x-api-key'         => config('services.anthropic.key'),
    'anthropic-version' => '2023-06-01',
])->post('https://api.anthropic.com/v1/messages', [
    'model'      => 'claude-opus-5',
    'max_tokens' => 8000,
    'system'     => $skillMd . "\n\n---\n\n" . $profileMd,
    'messages'   => [['role' => 'user', 'content' => $doc]],
]);
```

| 스택 | 적합도 | 비고 |
|---|---|---|
| React / Next.js | ⭐⭐⭐⭐⭐ | 스트리밍 UI·대시보드 |
| Node / TS | ⭐⭐⭐⭐⭐ | 공식 SDK, 프론트와 타입 공유 |
| Python FastAPI | ⭐⭐⭐⭐⭐ | 공식 SDK, 레포 스크립트도 Python |
| PHP Laravel | ⭐⭐⭐⭐ | 공식 SDK 없지만 REST 호출로 충분 |

**웹앱화 시 필수 주의사항**

1. **API 키는 절대 프론트엔드에 두지 말 것** — 반드시 백엔드 프록시
2. **면책 문구를 UI에 강제 노출** — 레포 원칙 "loud is correct"
3. **인젝션 방어는 서버에서** — 업로드 문서는 항상 `user` 턴 + 데이터 래핑
4. **감사 로그 필수** — 누가 언제 무엇을 승인했는지
5. **Apache 2.0 준수** — 저작권 고지 + 라이선스 사본 + 변경사항 표시
6. **테넌트 격리** — `Cross-matter context: off`를 DB row-level security로 구현

---

## 5. 수익화 아이디어

### 대전제

1. **프롬프트 자체는 상품이 아니다** (Apache 2.0, 누구나 무료 획득 가능)
2. **팔리는 것은 ① 로컬라이즈 ② 도메인 이식 ③ UI/자동화/통합 ④ 지식 전달**
3. **한국 법률은 규제업** — 변호사법 §34(동업·알선 금지), §109(비변호사 법률사무 금지).
   **"변호사 대체" 포지셔닝 금지, "변호사 업무 보조 도구" 포지셔닝만.**

### 아이디어 1 — 다른 도메인 이식 ⭐⭐⭐⭐⭐

| 이식 대상 | 원본 | 타겟 |
|---|---|---|
| claude-for-tax | commercial-legal | 세무사 사무소 |
| claude-for-hr | employment-legal | 중소기업 인사팀 |
| claude-for-realty | corporate-legal | 부동산 중개·시행 |
| claude-for-construction | regulatory-legal | 건설 감리·안전 |
| claude-for-devops | 쿡북 5종 | 개발팀 |
| claude-for-clinic | privacy-legal | 병·의원 |
| claude-for-import | regulatory-legal | 수출입 업체 |

수익 모델: 무료 플러그인(마케팅) → SaaS 월 9.9만~99만 → 온프레미스 연 2,000만~
→ 도입 컨설팅 500만~3,000만 → 커스텀 MCP 개발 1,000만~

### 아이디어 2 — 한국 법률 로컬라이즈 ⭐⭐⭐⭐ (가치 최고, 리스크 최고)

현재 레포는 100% 미국법 기준(FMLA, CFRA, FRE 408, CCPA, NPRM, Westlaw, PACER).

| 미국 원본 | 한국 대응 |
|---|---|
| FMLA/CFRA/PFL | 근로기준법, 남녀고용평등법 |
| CCPA/GDPR | 개인정보보호법(PIPA), 정보통신망법 |
| FRE 408 | 민사소송법, 형사소송법 |
| NPRM | 입법예고 (국민참여입법센터) |
| CourtListener | 종합법률정보, 케이스노트, 엘박스 |
| PACER | 전자소송 |

**진짜 해자(moat) = 한국 법률 MCP 서버.** 프롬프트는 복제되지만 커넥터는 복제 어렵다.

1. 국가법령정보 MCP (법제처 Open API — 무료)
2. 종합법률정보 MCP (대법원 판례)
3. 입법예고 MCP (규제 감시용)
4. 전자소송 MCP (docket-watcher 한국판)
5. **DART MCP** (금감원 전자공시 — M&A 실사용, 공개 API 우수)
6. **KIPRIS MCP** (특허청 — ip-legal용, 공개 API 우수)
7. 등기부·건축물대장 MCP

⚠️ 안전 포지셔닝:
- ❌ "AI 변호사", "법률 상담", "승소 가능성"
- ✅ "법무팀 업무 보조 도구", "초안 생성 소프트웨어", "검토 시간 단축"
- 판매 대상은 **로펌 · 사내 법무팀 · 법무 대행사 · 로스쿨** (일반 소비자 X)

### 아이디어 3 — 로스쿨/변시 준비 ⭐⭐⭐⭐⭐ (최우선 추천)

- ✅ 변호사법 규제 밖 (교육 서비스)
- ✅ B2C 가능 → 결제 전환 빠름
- ✅ `law-student` 13개 + `legal-clinic` 16개 스킬이 이미 완성품

| 원본 스킬 | 한국판 |
|---|---|
| socratic-drill | 소크라테스식 문답 (민·형·헌) |
| irac-practice | 사례형 답안 채점 |
| case-brief | 판례 요약 |
| outline-builder | 서브노트 자동 생성 |
| flashcards | Leitner 방식 조문·판례 암기 |
| bar-prep-questions | 변시 기록형·선택형 생성 |
| exam-forecast | 교수 기출 분석 → 출제 경향 |
| cold-call-prep | 수업 발표 대비 |
| study-plan | 변시 D-day 역산 플랜 |
| legal-writing | 답안 구조 첨삭 (대필 X) |

가격 설계: 무료(1일 3문제) / 월 19,900 / 월 39,900 / 시즌권 299,000 /
로스쿨 B2B 연 1,000만~3,000만

보수적 시뮬레이션: 로스쿨생 6,000명 × 전환 8% = 480명 × 월 25,000원
= 월 1,200만원(연 1.44억) + B2B 5곳 7,500만 ≈ **연 2.2억**

### 아이디어 4 — 도구·인프라 판매 (B2D) ⭐⭐⭐⭐

| 제품 | 기반 | 가격 |
|---|---|---|
| Skill Studio (스킬 작성 GUI + I1~I11 검증) | marketplace 스키마 | 월 $29~99 |
| Agent Guard (권한 과다 자동 검출 + CI) | lint-tool-scope.py | 월 $99~499 |
| Prompt Registry (버전관리·A/B·롤백) | 스킬 구조 | 월 $199~ |
| 한국 공공 API MCP 팩 (DART/KIPRIS/법제처) | .mcp.json | 라이선스·종량제 |
| Cookbook Deploy (원클릭 배포·모니터링) | deploy-managed-agent.sh | 월 $149~ |
| Injection Test Suite | orchestrate.py 5계층 | 건당 $2,000~ |

### 아이디어 5 — 교육·컨설팅 ⭐⭐⭐⭐ (즉시 현금화)

| 상품 | 가격 |
|---|---|
| 온라인 강의 "Claude 플러그인 완전정복" | 99,000원/강의 |
| 기업 워크샵 (사내 AI 에이전트 구축) | 300~800만원/회 |
| 법무팀 도입 컨설팅 | 1,000~3,000만원 |
| 기술 블로그/뉴스레터 스폰서십 | 월 100~500만원 |
| 템플릿 팩 (Gumroad) | $49~199 |

즉시 쓸 콘텐츠 주제:
1. "Anthropic이 공개한 151개 프롬프트를 분석했다"
2. "AI 에이전트에 권한을 주는 법 — Least Privilege 실전"
3. "프롬프트 인젝션 5계층 방어 — Anthropic 코드 읽기"
4. "설정 영속화 패턴 — 에이전트를 기억하게 만들기"
5. "151개 프롬프트로 배우는 도메인 특화 AI 설계"

### 종합 비교

| 아이디어 | 난이도 | 초기투자 | 잠재력 | 규제리스크 | 회수기간 | 추천 |
|---|---|---|---|---|---|---|
| 1. 도메인 이식 | 중 | 낮음 | 높음 | 낮음 | 3~6개월 | ⭐⭐⭐⭐⭐ |
| 2. 한국 법률화 | 높음 | 높음 | 최고 | **높음** | 12개월+ | ⭐⭐⭐⭐ |
| 3. 로스쿨/변시 | 낮음 | 낮음 | 중~높음 | 거의 없음 | 2~4개월 | ⭐⭐⭐⭐⭐ |
| 4. 도구/인프라 | 중 | 중간 | 중~높음 | 없음 | 6개월 | ⭐⭐⭐⭐ |
| 5. 교육/컨설팅 | 낮음 | 거의 0 | 중간 | 없음 | 즉시 | ⭐⭐⭐⭐ |

### 권장 3단 로켓 전략

```
1단 (0~2개월)   아이디어 5: 교육/콘텐츠
                → 투자 0원, 즉시 현금흐름, 인지도 확보
                → 동시에 아이디어 1·3 시장 검증

2단 (2~6개월)   아이디어 3: 로스쿨/변시 SaaS
                → law-student 스킬 한국화 (이미 완성품)
                → React + Node/Laravel MVP → B2C 구독 MRR 확보
                → 규제 리스크 거의 없음

3단 (6~18개월)  아이디어 1 또는 4로 확장
                → 1단 인바운드 리드 중 반응 좋은 도메인 선택
                → 한국 MCP 서버(DART/KIPRIS/법제처) 오픈소스 공개 → 커뮤니티 확보
                → 필요 시 변호사·노무사 파트너십 후 아이디어 2 진입
```

---

## 6. 반드시 기억할 원칙

> **모든 출력물은 변호사 검토용 초안이다.**
> 법률 자문도, 법적 결론도, 변호사 대체품도 아니다.
> 이 플러그인들은 검토를 **더 빠르게** 만들 뿐, 검토를 **대체하지 않는다.**
>
> — README.md (원문 의역)

이 원칙이 151개 파일 전체에 일관되게 구현되어 있으며,
어떤 형태로 재활용하든 이 구조는 반드시 유지해야 한다.

---

## 참고 링크

- 이 레포: https://github.com/bmshin94/claude-for-legal
- 원본 레포: https://github.com/anthropics/claude-for-legal
- 빠른 시작: [QUICKSTART.md](QUICKSTART.md)
- 전체 레퍼런스: [README.md](README.md)
- 커넥터 추가 가이드: [CONNECTORS.md](CONNECTORS.md)
- 기여 가이드: [CONTRIBUTING.md](CONTRIBUTING.md)
- 매니지드 에이전트 쿡북: [managed-agent-cookbooks/](managed-agent-cookbooks/)
