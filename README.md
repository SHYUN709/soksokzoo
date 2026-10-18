# 속마음 동물원

> 내 안에 사는 동물은 누구일까?
> 80개 문장에 답하면 나를 닮은 동물 캐릭터와 찰떡 궁합, 살짝 안 맞는 궁합을 알려주는 자기이해 테스트.

**테스트 하러 가기 → https://shyun709.github.io/soksokzoo/**

---

## 이런 걸 알려줘

- **내 캐릭터**: 동물 5마리 × 아이템 5개 = 25가지 캐릭터 중 하나 (예: "취향 확실한 비버")
- **이런 사람이야**: 성향 설명, 강점, 조심할 점
- **동물 궁합**: 찰떡 궁합 동물과 살짝 안 맞는 동물
- **기준 궁합**: 삶의 기준이 잘 맞는 사람과 엇갈리는 사람
- **상세 마음 지도**: 10개 축 점수를 오각형 차트 2장으로
- **인스타 스토리 카드**: 1080×1920 결과 이미지 저장·공유

## 캐릭터 구성

**나는 어떤 사람? (성향 1위 → 동물)**

| 동물 | 성향 | 한 줄 |
|---|---|---|
| 다람쥐 | 현실감 | 모아두고 대비하는 현실 만렙 |
| 고양이 | 감수성 | 분위기를 제일 먼저 읽는 섬세함 |
| 리트리버 | 친화력 | 어디서든 분위기 메이커 |
| 올빼미 | 이상추구 | 생각이 많은 밤의 몽상가 |
| 비버 | 추진력 | 맡으면 끝까지, 일 처리 장인 |

**나의 기준 (기준 1위 → 수식어 + 아이템)**

| 기준 | 수식어 | 아이템 |
|---|---|---|
| 관계 | 곁을 챙기는 | 빨간 목도리 |
| 신뢰 | 약속은 꼭 지키는 | 금빛 배지 |
| 체계 | 계획대로 움직이는 | 체크리스트 클립보드 |
| 주관 | 내 갈 길 가는 | 선글라스 |
| 취향 | 취향 확실한 | 베레모 |

## 점수는 이렇게 계산해

- 총 80문항 (성향 40 + 기준 40), 축마다 8문항, 1~5점 척도
- 축마다 **역문항 2개**를 넣어서 "그렇다"만 연달아 누르는 응답 편향을 줄였어 (역문항은 6−점수로 계산)
- 같은 축 문항이 연달아 나오지 않도록 섞어서 배치
- 축별 평균을 0~100점으로 환산하고, 성향 1위가 동물, 기준 1위가 수식어가 돼
- 성향 점수 차가 15점 미만이면 "상황마다 다른 동물이 나오는 타입"으로 따로 안내

## 기술

- HTML · CSS · JavaScript 파일 하나 (`index.html`), 빌드 과정 없음
- 캐릭터는 SVG로 직접 그림, 스토리 카드는 Canvas로 생성
- 응답은 브라우저(localStorage)에 저장해. AI 심층 인터뷰를 누를 때만 문항 응답과 인터뷰 답이 AI 서버로 가
- AI: Cloudflare Worker 프록시(`worker/`) → Google Gemini API (아래 참고)
- 배포: GitHub Pages

## 로컬에서 실행

```bash
git clone https://github.com/SHYUN709/soksokzoo.git
cd soksokzoo
# index.html을 브라우저로 열면 끝
```

## 앞으로 할 것

- [x] AI 심층 인터뷰 (애매한 축을 AI가 꼬리 질문으로 보정)
- [ ] 친구랑 궁합 보기 (두 사람 결과 비교)
- [ ] 캐릭터 일러스트 고도화
- [ ] 방문자·완주율 통계

## AI 심층 인터뷰가 동작하는 방식

| 열린 곳 | AI 호출 방법 |
|---|---|
| claude.ai 아티팩트 | `window.claude.use("sample")` (그대로) |
| GitHub Pages 등 그 밖 | `index.html`의 `AI_PROXY` 주소(Cloudflare Worker)로 `fetch` |
| `AI_PROXY`가 비어 있음 | AI 패널을 숨기고 "프롬프트 복사"만 남김 |

프록시 엔드포인트 (`worker/src/index.js`):

- `POST /questions` — `{answers: number[80]}` → `{questions:[{axes, q}]}` (질문 3개, JSON)
- `POST /interpret` — `{answers, interview:[{axes, q, a}]}` → SSE 스트림
  - `event: text` `{t}` 글 조각 / `event: done` `{truncated, refused}` / `event: error` `{code}`

프롬프트는 서버가 만든다. 브라우저는 문항 응답(1~5)과 인터뷰 답만 보내고, 점수도 서버가 다시 계산하므로 프록시를 아무 질문에나 쓰는 범용 AI로 돌려쓸 수 없어.

### 남용 방지

- **CORS / Origin 검사**: `ALLOWED_ORIGINS`에 있는 주소에서 온 요청만 받는다(그 밖은 403). curl처럼 Origin을 위조하는 요청은 이것만으로 못 막아서 아래 횟수 제한이 함께 필요해.
- **하루 호출 제한**: IP당 `DAILY_LIMIT_PER_IP`회(기본 10, 검사 1회 = 질문 1 + 해석 1 = 2회), 전체 `DAILY_LIMIT_GLOBAL`회(기본 300). 날짜는 UTC 기준이고, 카운터는 Workers KV에 48시간 저장돼.
- **토큰 상한**: 질문 `maxOutputTokens` 4000, 해석 8000 (`LIMITS`에서 조정). 요청 본문은 16KB, 질문 300자, 답변 800자까지만 받는다.

## 배포 (Cloudflare Workers)

준비물: Node.js 18+, Cloudflare 계정(무료 플랜 가능, 이메일 인증 필요), Gemini API 키([Google AI Studio](https://aistudio.google.com/apikey)에서 무료 발급).

> Windows PowerShell에서 `npx`가 "스크립트를 실행할 수 없다"며 막히면 `npx` 대신 `npx.cmd`를 쓰면 돼.

```bash
cd worker
npm install
npx wrangler login
```

**1. 레이트리밋용 KV 만들기** — 출력된 `id`를 `worker/wrangler.toml`의 `[[kv_namespaces]] id`에 넣어.

```bash
npx wrangler kv namespace create SOKSOKZOO_RATE_LIMIT
```

**2. 허용 도메인 확인** — `worker/wrangler.toml`의 `ALLOWED_ORIGINS`는 이미 `https://shyun709.github.io`로 돼 있어. Origin에는 경로(`/soksokzoo/`)가 안 붙으니 그대로 두면 돼. 커스텀 도메인을 쓰게 되면 그 주소를 쉼표로 추가해.

**3. API 키 넣기** — 프롬프트가 나오면 키를 붙여넣어. 키는 Cloudflare에만 암호화돼 저장되고 저장소에는 남지 않아.

```bash
npx wrangler secret put GEMINI_API_KEY
```

**4. 배포**

```bash
npx wrangler deploy
```

출력되는 `https://soksokzoo-ai-proxy.<계정>.workers.dev` 주소를 복사해.

**5. 페이지에 연결** — `index.html`에서 아래 줄을 찾아 주소를 넣고 커밋·푸시해. (이 저장소는 이미 `https://soksokzoo-ai-proxy.wpi-ai-proxy.workers.dev`로 연결돼 있어.)

```js
const AI_PROXY = "https://soksokzoo-ai-proxy.<계정>.workers.dev";
```

### 확인

```bash
curl -i -X POST https://soksokzoo-ai-proxy.<계정>.workers.dev/questions -H "Origin: https://evil.example" -H "Content-Type: application/json" -d "{}"
```

`403 forbidden_origin`이 나오면 정상이야. 실제 동작은 GitHub Pages에서 검사를 끝내고 "질문 받기"를 눌러 봐. 로그는 `npx wrangler tail`로 볼 수 있어.

### 로컬에서 테스트

```bash
cd worker
cp .dev.vars.example .dev.vars
npx wrangler dev
```

`.dev.vars`에 키를 넣고, 다른 터미널에서 `python -m http.server 8080`으로 페이지를 띄운 뒤 `AI_PROXY`를 `http://127.0.0.1:8787`로 잠깐 바꿔서 테스트해. (커밋하기 전에 되돌리기)

### 설정 바꾸기

- 모델: `wrangler.toml`의 `MODEL` (기본 `gemini-3.8-flash`). 이 모델이 붐비거나(503) 한도(429)에 걸리면 한 번 더 시도한 뒤 `FALLBACK_MODEL`(기본 `gemini-3.5-flash-lite`)로 넘어가.
- 횟수: `DAILY_LIMIT_PER_IP`, `DAILY_LIMIT_GLOBAL`. 바꾼 뒤 `npx wrangler deploy`.
- 무료 등급에는 분당·하루 요청 한도가 있어서, 넘으면 페이지에 "요청이 너무 많아" 안내가 떠. 유료 등급으로 바꾸면 Google Cloud 결제 계정에서 예산 알림을 걸어 둬.

### 개인정보 안내

Gemini API 무료 등급은 Google 요금 안내상 입력 내용이 "Google 제품 개선에 사용"될 수 있어. 이 검사는 문항 응답과 인터뷰 답변을 보내므로, 민감한 내용을 쓰지 않도록 안내하거나 유료 등급으로 바꾸는 걸 고려해.

## 비밀값 관리

- API 키는 `wrangler secret`으로만 넣는다. `wrangler.toml`, `index.html`, 커밋 메시지 어디에도 쓰지 마.
- 로컬용 `.dev.vars`는 `.gitignore`에 들어 있어. 커밋 전에 `git status`로 한 번 더 확인해.
- 키가 노출됐다면 Google AI Studio에서 바로 삭제하고 새 키로 `wrangler secret put`을 다시 실행해.

## 프롬프트 수정할 때

질문·해석 프롬프트는 두 군데에 있어. 바꿀 땐 둘 다 맞춰 줘.

- `worker/src/index.js` — `questionsPrompt`, `interpretPrompt` (GitHub Pages용)
- `index.html` — `claudeBackend` (claude.ai 아티팩트용)

문항(`POOL`)이나 축 설명(`AXES`)을 바꾸면 `worker/src/index.js`의 사본도 같이 바꿔야 점수가 맞아.

## 안내

재미로 보는 자기이해 테스트예요. 심리 진단이나 전문 검사를 대신하지 않아요.
문항, 캐릭터, 해석은 모두 직접 만들었어요.

---

Made by [@SHYUN709](https://github.com/SHYUN709)
