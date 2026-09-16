# 📚 Temporal TypeScript SDK 학습 정리 (한국어)

> 이 문서는 `temporalio/sdk-typescript` 저장소를 처음 접한 뒤,
> 코드를 직접 뜯어보며 정리한 **한국어 학습 노트 + 활용 전략** 문서입니다.
>
> 작성일: 2026-09-16

---

## 🔗 관련 링크 모음

### 이 저장소

| 항목                                | 주소                                             |
| ----------------------------------- | ------------------------------------------------ |
| 📦 **원본 저장소 (upstream)**       | https://github.com/temporalio/sdk-typescript     |
| 🍴 **내 포크 (this repo)**          | https://github.com/bmshin94/sdk-typescript       |
| 🦀 **Core SDK (Rust, 서브모듈)**    | https://github.com/temporalio/sdk-rust           |
| 🖥️ **Temporal 서버 본체 (별 23k+)** | https://github.com/temporalio/temporal           |
| 💡 **공식 예제 모음**               | https://github.com/temporalio/samples-typescript |

### 문서

| 항목                          | 주소                                        |
| ----------------------------- | ------------------------------------------- |
| 📖 공식 문서                  | https://docs.temporal.io                    |
| 🇰🇷 **한국어 문서 (커뮤니티)** | https://docs.temporal.kr/                   |
| 📘 TypeScript 개발 가이드     | https://docs.temporal.io/develop/typescript |
| 🔍 API 레퍼런스               | https://typescript.temporal.io/             |
| ⌨️ Temporal CLI               | https://docs.temporal.io/cli                |
| 💰 Temporal Cloud 가격        | https://docs.temporal.io/cloud/pricing      |

### 다른 언어 SDK (총 8종)

`.NET` · `Go` · `Java` · **`PHP`** · `Python` · `Ruby` · `Rust` · `TypeScript`
→ https://docs.temporal.io/encyclopedia/temporal-sdks

---

## 1. 🚀 Temporal이 뭐야?

### 한 줄 요약

> **"요리하다가 정전돼도, 전기 들어오면 하던 데서 그대로 이어서 요리해주는 주방장"**

정식 용어로는 **Durable Execution(내구성 있는 실행)** 플랫폼입니다.

### 어떤 문제를 해결하나?

쇼핑몰 주문 처리를 예로 들면:

```
①재고 확인 → ②카드 결제 → ③포인트 적립 → ④배송 접수 → ⑤문자 발송
```

**③번에서 서버가 죽으면?**

- 돈은 빠져나갔는데 포인트는 안 쌓임
- 배송 접수 안 됨 → 고객 불만 → 새벽 호출 😱

기존 방식은 이걸 막으려고 **상태 기록 + 재시도 + 보상 트랜잭션 코드**를 전부 직접 짜야 했습니다.

### Temporal을 쓰면?

```ts
export async function 주문처리(주문: Order) {
  await 재고확인(주문);
  await 카드결제(주문);
  await 포인트적립(주문);
  await 배송요청(주문);
  await 알림전송(주문);
}
```

평범한 `async/await` 코드인데:

| Temporal이 자동으로 해주는 것                         |
| ----------------------------------------------------- |
| ✅ 어디까지 실행했는지 자동 기록                      |
| ✅ 서버 재시작 시 **③번부터 정확히 이어서** 실행      |
| ✅ 실패 시 자동 재시도 (횟수·간격 설정 가능)          |
| ✅ `await sleep('30 days')` — 30일 대기도 실제로 동작 |
| ✅ 웹 UI에서 실행 이력 전부 확인 가능                 |

---

## 2. 🏗️ 아키텍처 — 부품 3개

```
[ ① Temporal 서버 ]  ← 기억을 담당하는 뇌 (별도 실행 필요!)
        ↕
[ ② Worker ]         ← 실제로 일하는 일꾼 (내가 만드는 Node 앱)
        ↕
[ ③ Client ]         ← 일을 시키는 리모컨 (웹서버에 탑재)
```

**②, ③번이 이 SDK가 제공하는 부분.** ①번은 따로 띄워야 합니다.

### 핵심 패키지 4인방

| 패키지              | 역할                             | 비유          |
| ------------------- | -------------------------------- | ------------- |
| `packages/workflow` | 실행 순서(시나리오) 정의         | 📜 **대본**   |
| `packages/activity` | 실제 작업 함수 (DB, API 호출 등) | 🎭 **연기**   |
| `packages/worker`   | 대본대로 실행시키는 실행기       | 🎬 **감독**   |
| `packages/client`   | 워크플로우 시작·조회·취소        | 🎮 **리모컨** |

### 지원 패키지

| 패키지              | 설명                                                             |
| ------------------- | ---------------------------------------------------------------- |
| `common`            | 공통 로직 (직렬화, 에러 타입 등)                                 |
| `proto`             | gRPC 통신용 protobuf 정의 (자동 생성)                            |
| `core-bridge`       | **Rust 코어 엔진 ↔ Node 연결 브릿지**                            |
| `testing`           | 테스트 환경 (⏩ **시간 스킵 가능!** 30일 대기 로직도 1초 테스트) |
| `create-project`    | `npx @temporalio/create` 프로젝트 생성기                         |
| `envconfig`         | TOML 프로필 기반 접속 설정 관리 (AWS CLI 프로필과 유사)          |
| `nexus`             | 서비스 간 연결                                                   |
| `plugin`            | Temporal 자체 플러그인 시스템                                    |
| `lambda-worker`     | AWS Lambda에서 Worker 실행                                       |
| `cloud`             | Temporal Cloud 클라이언트                                        |
| `nyc-test-coverage` | 코드 커버리지 연동                                               |

### `contrib/` — AI 통합 패키지 (요즘 제일 핫한 부분 🔥)

| 폴더                              | 연동 대상                       |
| --------------------------------- | ------------------------------- |
| `ai-sdk`                          | **Vercel AI SDK** (가장 범용적) |
| `openai-agents`                   | OpenAI Agents SDK               |
| `google-adk-agents`               | Google Agent Development Kit    |
| `strands`                         | AWS Strands Agents              |
| `langsmith`                       | LangSmith (추적/디버깅)         |
| `workflow-streams`                | 스트리밍 응답 처리              |
| `interceptors-opentelemetry(-v2)` | OpenTelemetry 관측              |
| `external-storage-s3 / gcs`       | 대용량 페이로드 외부 저장       |

---

## 3. 🛠️ 설치 및 사용법

### STEP 1. Temporal 서버 실행 (제일 중요!)

```bash
# macOS
brew install temporal

# Linux / Windows
curl -sSf https://temporal.download/cli.sh | sh

# 개발용 서버 실행 (설정 불필요)
temporal server start-dev
```

👉 실행 후 **http://localhost:8233** 접속하면 웹 UI에서 워크플로우 실행 상태를 눈으로 확인할 수 있습니다.

### STEP 2. 프로젝트 생성

```bash
npx @temporalio/create@latest my-app
cd my-app
```

### STEP 3. 실행

```bash
npm run start.watch   # 터미널 1: Worker 실행
npm run workflow      # 터미널 2: 워크플로우 실행
```

### 기존 프로젝트에 추가하는 경우

```bash
# 클라이언트 기능만 (워크플로우 시작/조회)
npm i @temporalio/client @temporalio/common

# 워커 기능까지 (워크플로우/액티비티 실행)
npm i @temporalio/worker @temporalio/workflow @temporalio/activity @temporalio/common
```

> ⚠️ **`@temporalio/*` 패키지는 버전 번호를 전부 동일하게 맞춰야 합니다.**
> (peer dependency로 강제되지만 복잡한 모노레포에선 주의 필요)

### 실제 코드 4개 파일

```ts
// activities.ts — 실제 작업 (평범한 함수)
export async function sendEmail(to: string, msg: string) {
  await fetch('...');
}
```

```ts
// workflows.ts — 실행 순서 정의
import { proxyActivities } from '@temporalio/workflow';
import type * as activities from './activities';

const { sendEmail } = proxyActivities<typeof activities>({
  startToCloseTimeout: '1 minute', // 1분 초과 시 재시도
});

export async function 주문완료() {
  await sendEmail('a@b.com', '주문 완료!');
}
```

```ts
// worker.ts — 일꾼 실행
import { NativeConnection, Worker } from '@temporalio/worker';
import * as activities from './activities';

const connection = await NativeConnection.connect({ address: 'localhost:7233' });
const worker = await Worker.create({
  connection,
  namespace: 'default',
  taskQueue: 'my-queue',
  workflowsPath: require.resolve('./workflows'),
  activities,
});
await worker.run();
```

```ts
// client.ts — 워크플로우 시작
import { Client } from '@temporalio/client';

const client = new Client();
await client.workflow.start(주문완료, {
  taskQueue: 'my-queue',
  workflowId: 'order-123',
});
```

### 이 저장소 자체를 빌드하려면 (SDK 기여자용)

```bash
pnpm install
pnpm run build   # ⚠️ Rust 컴파일 포함, 10~20분 소요 가능
pnpm test        # ⚠️ 로컬 Temporal 서버 실행 필요
pnpm lint
```

**요구 사항:** Node 20/22/24 · pnpm ≥ 10.27 · Rust ≥ 1.53 · Protocol Buffers

---

## 4. ❓ 플러그인? 스킬? MCP?

### 결론: **셋 다 아닙니다. 평범한 npm 라이브러리(SDK)입니다.**

| 구분                       | 설명                           | Temporal은?  |
| -------------------------- | ------------------------------ | ------------ |
| 🔌 Claude 플러그인/스킬    | Claude Code에 기능 추가        | ❌ 아님      |
| 🔗 MCP 서버                | AI가 외부 툴을 쓰게 하는 규격  | ❌ 아님      |
| 📦 **npm 라이브러리(SDK)** | `npm install` 해서 코드에 사용 | ✅ **이것!** |

### 헷갈릴 만한 부분 2가지

**① `packages/plugin` 폴더**
→ **Temporal 자체의 플러그인 시스템**입니다. Claude 플러그인과 무관.
(Worker에 기능을 끼워넣는 용도. `SimplePlugin` 등을 export)

**② `contrib/ai-sdk/src/mcp.ts` 파일**
→ Temporal이 **MCP 클라이언트 역할**을 하는 코드입니다:

```ts
export class TemporalMCPClient {
  async tools(): Promise<ToolSet> { ... }
}
```

**MCP 툴 호출을 Temporal Activity로 감싸서 실패해도 자동 재시도되게** 만들어줍니다.
`google-adk-agents/src/mcp.ts`, `strands/src/temporal-mcp-client.ts` 에도 동일한 패턴이 존재합니다.

> 🎯 **정리: Temporal은 MCP 서버가 아니라, MCP를 "튼튼하게 써주는" 라이브러리입니다.**

---

## 5. 🔑 API 토큰이 필요한가?

### 결론: **로컬 개발은 토큰 불필요, 완전 무료**

| 상황                                       | 토큰                       | 비용              |
| ------------------------------------------ | -------------------------- | ----------------- |
| 🏠 로컬 개발 (`temporal server start-dev`) | ❌ 불필요                  | **무료**          |
| 🖥️ 셀프호스팅                              | ❌ 불필요 (직접 인증 설정) | 무료 (인프라비만) |
| ☁️ Temporal Cloud                          | ✅ 필요                    | 유료              |

### Temporal Cloud 인증 방식 2가지

`packages/client/src/connection.ts:120` 에서 확인:

```ts
apiKey?: string | (() => string);   // 방법 ①: API 키
// 방법 ②: mTLS 인증서
```

- `apiKey`와 `Authorization` 헤더는 **동시 사용 불가** (상호 배타적)
- `Connection.setApiKey()` 로 런타임 중 갱신 가능
- `packages/envconfig` 가 **TOML 파일로 프로필별 접속 설정**을 관리 (개발/운영 분리)

### ⚠️ AI 통합 사용 시

`contrib/ai-sdk`, `openai-agents` 등을 쓰려면 **OpenAI / Anthropic API 키가 별도로** 필요합니다.
Temporal은 AI를 대신 호출하는 게 아니라, 내 호출을 **튼튼하게 감싸주는** 역할입니다.

---

## 6. ⭐ 왜 GitHub에서 유명한가?

### 별 개수 현황

| 저장소                                  | 별                                            |
| --------------------------------------- | --------------------------------------------- |
| `temporalio/temporal` (서버 본체)       | **약 23,000개** (GitHub 전체 랭킹 약 1,800위) |
| `temporalio/sdk-typescript` (이 저장소) | 약 900개                                      |

### 유명한 이유 5가지

**① 🚕 Uber에서 태어남**
Uber가 만든 **Cadence** 프로젝트 개발자들이 독립해 2019년 창업. 대규모 프로덕션에서 검증된 기술.

**② 💰 회사가 풀타임으로 개발**

- CI / 릴리즈 / 나이틀리 / **스트레스 테스트**까지 자동화 (`.github/workflows/`)
- CHANGELOG 꼼꼼히 관리 — **v1.24.0이 2026-09-14 릴리즈**

**③ 🌍 8개 언어 SDK 지원**
백엔드는 Go, AI는 Python, 웹은 TypeScript — **섞어 써도 하나의 워크플로우로 연결됨**

**④ ⚙️ 핵심 엔진이 Rust**

```
[submodule "sdk-core"]
	path = packages/core-bridge/sdk-core
	url = https://github.com/temporalio/sdk-rust.git
```

→ 모든 언어 SDK가 **동일한 Rust 코어**를 공유 (빠르고 일관성 있음)

**⑤ 🤖 AI 붐에 정확히 대응**
`contrib/` 에 주요 AI 프레임워크 연동을 전부 갖춤

> 📌 공개 자료에 따르면 Stripe, Netflix, Coinbase 등이 프로덕션에서 사용 중이라고 알려져 있습니다.

---

## 7. 🤖 로컬 AI 에이전트 구축에 도움이 되는가?

### 결론: **장시간 실행 에이전트라면 매우 유용, 단순 챗봇이면 오버킬**

### ✅ 왜 유용한가

```
❌ 그냥 만들면:
   툴1 → 툴2 → 툴3 → 💥 API 타임아웃 → 처음부터 다시 → 토큰 비용 낭비

✅ Temporal 사용 시:
   툴1 ✅ → 툴2 ✅ → 툴3 💥 → 자동 재시도 → 툴3 ✅ → 계속 진행
```

추가 이점:

- 🕵️ 모든 LLM 호출 / 툴 호출이 **웹 UI에 기록** → 디버깅 용이
- 🙋 **사람 승인 대기** 가능 (3일 뒤 승인해도 워크플로우 유지)
- ⏳ 몇 시간~며칠 걸리는 에이전트 구현 가능

### `contrib/strands/src/` 파일 구성 (참고)

```
temporal-agent.ts        temporal-model.ts
temporal-mcp-client.ts   temporal-mcp-tool.ts
temporal-activity-tool.ts  heartbeat.ts
```

→ 에이전트 · 모델 · MCP 툴을 전부 Temporal로 감쌀 수 있게 설계됨

### ⚠️ 단점

| 단점         | 설명                                                                                         |
| ------------ | -------------------------------------------------------------------------------------------- |
| 🏋️ 무겁다    | 서버를 별도로 띄워야 함                                                                      |
| 📚 학습 곡선 | **결정론적 코드** 개념 필수 — 워크플로우 안에서 `Math.random()`, `Date.now()` 직접 사용 불가 |
| 🐌 오버킬    | 툴 2~3개짜리 챗봇엔 과함                                                                     |

### 판단 기준

> **에이전트가 5분 이상 돌고, 실패하면 아까운 작업** → 사용 추천
> **질문하면 바로 답하는 챗봇** → 불필요

---

## 8. ⚛️🐘 React / PHP로 만들 수 있는가?

### ⚛️ React — 직접 실행은 ❌, 백엔드 경유는 ✅

**직접 안 되는 이유:** Worker는 Node 전용 기능에 의존합니다.

```
worker_threads · vm · AsyncLocalStorage · async_hooks · Node-API 네이티브 모듈
```

→ 브라우저에 존재하지 않음.

**올바른 구조:**

```
[React 화면] → fetch → [백엔드 API (@temporalio/client)] → [Temporal 서버] ↔ [Worker]
```

**Next.js 예시:**

```ts
// app/api/order/route.ts (서버에서 실행)
import { Client } from '@temporalio/client';

export async function POST(req: Request) {
  const client = new Client();
  await client.workflow.start(주문처리, {
    taskQueue: 'orders',
    workflowId: `order-${Date.now()}`,
  });
  return Response.json({ ok: true });
}
```

> 📌 README에 따르면 `@temporalio/client`는 **Bun, Deno, Cloudflare Workers** 에서도 대체로 동작하지만 **공식 지원은 아님** (정기 테스트를 하지 않음).

### 🐘 PHP — 공식 SDK 존재 ✅

```bash
composer require temporal/sdk
```

- 저장소: https://github.com/temporalio/sdk-php
- ⚠️ **RoadRunner** 애플리케이션 서버가 추가로 필요 (PHP는 요청마다 프로세스가 종료되어 장기 실행 워커를 못 돌리기 때문)

### 🌟 최대 강점 — 언어 혼합 사용

```
[Temporal 서버]
   ↓          ↓            ↓
[TS Worker]  [PHP Worker]  [Python Worker]
 알림 발송     레거시 결제    AI/ML 처리
```

**하나의 워크플로우 안에서 언어별로 나눠 처리 가능** → 레거시를 두고 **점진적 마이그레이션**이 가능합니다.

---

## 9. 💰 수익화 전략

### 📊 시장 현황 (2026년 기준)

| 선택지            | 비용                                                             |
| ----------------- | ---------------------------------------------------------------- |
| 🏠 로컬 개발      | 무료 (운영엔 사용 불가)                                          |
| 🖥️ 셀프호스팅     | 인프라 **$2,500~4,500/월** + 운영 인력                           |
| ☁️ Temporal Cloud | 최소 **$100/월**, 보통 **$200~2,000/월**, 대규모 **$50,000/월+** |

**Actions 과금:** 100만 건당 **$25~50**
**경쟁 플랫폼:** Trigger.dev(Apache 2.0, 셀프호스팅 무료) · Inngest · Restate · DBOS · Hatchet

### 🕳️ 발견한 시장 틈새 3개

| #   | 틈새                                   | 설명                                                                                         |
| --- | -------------------------------------- | -------------------------------------------------------------------------------------------- |
| ①   | **작은 팀의 계곡**                     | 무료(로컬)와 $200~2,000/월(Cloud) 사이가 비어있음                                            |
| ②   | **AI 에이전트 비용 폭탄**              | 툴 호출마다 Action 누적 → 비용 예측 불가, 추적 도구 없음                                     |
| ③   | **한국어 = 문서는 있으나 사람이 없음** | [docs.temporal.kr](https://docs.temporal.kr/)은 존재하나 **강의·영상·실전 사례·전문가 부재** |

---

### 🥇 TIER A — 자본 0원, 3개월 내 수익 가능

#### A-1. 컨설팅 / 구축 대행 ⭐⭐⭐⭐⭐

**타겟:** 결제·정산 시스템 보유 커머스/핀테크 · 크론잡 지옥에 빠진 회사 · AI 에이전트 안정성 문제를 겪는 팀

| 상품              | 가격대              | 기간    |
| ----------------- | ------------------- | ------- |
| 진단/설계 컨설팅  | 500만 ~ 1,500만원   | 2~4주   |
| PoC 구축          | 1,500만 ~ 3,000만원 | 1~2개월 |
| 전체 마이그레이션 | 3,000만 ~ 1억원     | 3~6개월 |
| 유지보수 리테이너 | 월 200만 ~ 500만원  | 지속    |

- ✅ 초기 비용 0원, **국내 경쟁자 거의 없음**
- ⚠️ 레퍼런스 없으면 첫 계약이 어려움 (닭-달걀 문제), 시간 팔이라 확장 제한

**첫 3단계**

1. 오픈소스 데모 프로젝트 1개 공개 (예: "Next.js 커머스 결제 + Temporal")
2. 제작 과정을 블로그/영상으로 기록 → 포트폴리오화
3. 링크드인/커뮤니티에서 타겟 CTO에게 접촉

#### A-2. 교육 콘텐츠 ⭐⭐⭐⭐

> ⚠️ **기본 튜토리얼은 이미 [docs.temporal.kr](https://docs.temporal.kr/)에 존재.** "Temporal이란?" 같은 주제는 차별화 불가.

**빈 주제를 노릴 것:**

| 주제                                | 이유                               |
| ----------------------------------- | ---------------------------------- |
| 🤖 **Temporal + AI 에이전트 실전**  | 전 세계적으로도 자료 부족          |
| 💸 비용 최적화 실전                 | Actions 절감 노하우 = 돈 되는 정보 |
| 🔄 레거시 → Temporal 마이그레이션   | 실전 삽질기 부재                   |
| 🧪 테스트 전략 (`packages/testing`) | 시간 스킵 테스트 등 고급 주제      |
| 🇰🇷 한국 기업 도입 사례              | 무주공산                           |

**진짜 목적은 컨설팅 유입:**

```
콘텐츠(신뢰) → "이 사람 Temporal 잘 아네" → 문의 → A-1 컨설팅 💰
```

---

### 🥈 TIER B — 제품화 (6개월~1년)

#### B-1. 버티컬 AI 에이전트 SaaS ⭐⭐⭐⭐⭐ (최강 추천)

> 💡 **핵심: Temporal을 파는 게 아니라, Temporal로 만든 "안 죽는 서비스"를 판다.**
> 고객은 Temporal이 뭔지 몰라도 됨.

**아이디어 예시**

| 서비스                     | 흐름                                                              | 가격                     |
| -------------------------- | ----------------------------------------------------------------- | ------------------------ |
| 📑 리서치/보고서 자동화    | 웹 크롤링 → 논문 검색 → 요약 → 차트 → PDF (20~40분)               | 건당 2만원 / 월 30만원   |
| 🛒 커머스 상품 등록 자동화 | 이미지 분석 → 상품명 → 상세페이지 → 번역 → 4개 채널 동시 등록     | 상품당 500원 / 월 20만원 |
| 📞 고객 문의 자동 처리     | 분류 → DB 조회 → 답변 생성 → **사람 검토 대기** → 발송 → CRM 기록 | 월 50만~200만원 (B2B)    |

**수익 구조 예시**

```
매출: 월 30만원 × 고객 30곳 = 900만원
비용: Temporal Cloud 약 30~50만원 + LLM API + 서버
→ 마진 70%+
```

- ✅ 반복 수익 · 기술 해자 · 확장 가능 · Temporal이 가장 빛나는 영역
- ⚠️ LLM API 비용 관리 필수, **만들기 전에 고객 5명에게 검증**

**첫 3단계**

1. 주변에서 "제일 귀찮은 반복작업" 조사
2. 하나만 골라 2주 안에 MVP
3. 무료 사용 → 지불 의향 확인 후 확장

#### B-2. 개발자 도구 (Temporal 애드온) ⭐⭐⭐⭐

| 도구                                 | 해결하는 문제                                                                | 가격       |
| ------------------------------------ | ---------------------------------------------------------------------------- | ---------- |
| **AI 에이전트 비용 추적기** (틈새 ②) | Actions + LLM 토큰 비용이 섞여 추적 불가 → 워크플로우별 실시간 비용 대시보드 | 월 $49~199 |
| **비개발자용 대시보드**              | 기본 UI는 개발자 전용 → CS/운영팀이 "주문 #1234 어디까지?" 확인              | 월 $99~299 |
| **워크플로우 테스트 자동화**         | 결정론 규칙 위반이 배포 후 터짐 → 배포 전 검증 + 리플레이 CI                 | 월 $29~99  |

> 💡 `contrib/interceptors-opentelemetry`, `contrib/langsmith` 패키지를 응용하면 구현 가능

#### B-3. 프리미엄 템플릿 판매 ⭐⭐⭐

- 🛒 커머스 결제 풀 세트 (결제→재고→배송→환불 보상)
- 🔄 SaaS 구독 관리 (갱신/실패/유예기간/해지)
- 🤖 AI 에이전트 스타터 (Next.js + Temporal + AI SDK)
- 📊 데이터 파이프라인 템플릿

**가격:** 개당 $99~499 (Gumroad, Lemon Squeezy 등)
✅ 한 번 만들면 계속 판매 / ⚠️ 마케팅이 8할

---

### 🥉 TIER C — 장기 도전 (1년+)

#### C-1. "작은 팀을 위한 매니지드 Temporal" ⭐⭐⭐

틈새 ① 공략. 월 $19~49로 쓸 수 있는 가벼운 매니지드 서비스.

⚠️ **난이도 최상** — 인프라 운영 전문성 필수, 장애 대응 부담, Temporal Cloud와 직접 경쟁, 경쟁자 다수(Trigger.dev/Inngest/Restate/DBOS/Hatchet).
→ **B-1로 수익과 팀을 먼저 확보한 뒤 고려할 것.**

---

### 📉 하지 말아야 할 것

| ❌                          | 이유                                                    |
| --------------------------- | ------------------------------------------------------- |
| "Temporal 자체"를 판매      | 고객은 도구가 아니라 **문제 해결**을 구매함             |
| 한국어 문서 번역 사업       | [docs.temporal.kr](https://docs.temporal.kr/) 이미 존재 |
| 간단한 작업에 Temporal 적용 | 오버엔지니어링 = 비용만 증가                            |
| 제품 먼저 만들고 고객 찾기  | 순서가 반대                                             |
| Temporal Cloud 재판매       | 마진 없음 + 약관 문제 소지                              |

---

### 🗺️ 추천 로드맵 (12개월)

```
📅 1~2개월차   데모 프로젝트 공개 + 블로그 3~5편          💰 0원 (투자기)
📅 3~4개월차   "Temporal + AI 에이전트" 시리즈 + 커뮤니티   💰 첫 문의 시작
📅 5~8개월차   컨설팅 1~2건 수주 + SaaS 아이디어 검증       💰 월 300만~1,000만원
📅 9~12개월차  SaaS MVP 출시 + 초기 고객 (컨설팅 병행)      💰 컨설팅 + 구독
```

**핵심 전략**

```
콘텐츠(신뢰) → 컨설팅(현금 + 진짜 고객 문제 파악) → SaaS(확장)
```

아이디어부터 시작하면 헛발질하지만, 컨설팅으로 **고객의 실제 고통을 먼저 파악**하면 실패 확률이 크게 줄어듭니다.

---

### 🎯 수익화 요약표

| 등급 | 아이디어                | 난이도     | 수익까지 | 추천도     |
| ---- | ----------------------- | ---------- | -------- | ---------- |
| 🥇   | 컨설팅/구축 대행        | ⭐⭐       | 3개월    | ⭐⭐⭐⭐⭐ |
| 🥇   | 교육 콘텐츠 (빈 주제만) | ⭐⭐       | 3~6개월  | ⭐⭐⭐⭐   |
| 🥈   | 버티컬 AI 에이전트 SaaS | ⭐⭐⭐⭐   | 6~12개월 | ⭐⭐⭐⭐⭐ |
| 🥈   | 비용 추적 도구          | ⭐⭐⭐     | 6개월    | ⭐⭐⭐⭐   |
| 🥈   | 템플릿 판매             | ⭐⭐       | 3~6개월  | ⭐⭐⭐     |
| 🥉   | 매니지드 서비스         | ⭐⭐⭐⭐⭐ | 1년+     | ⭐⭐       |

> 💖 **한 줄 조언: "Temporal을 팔지 말고, Temporal로 만든 '안 죽는 무언가'를 팔아라."**

---

## 10. ⚡ 다음에 할 일 체크리스트

- [ ] `temporal server start-dev` 실행 후 http://localhost:8233 확인
- [ ] `npx @temporalio/create@latest my-first-workflow` 로 첫 프로젝트 생성
- [ ] `packages/test/src/` 예제 코드 둘러보기 (실전 예제 다수)
- [ ] `packages/workflow/src/index.ts` 상단 주석 정독 (튜토리얼 수준)
- [ ] `contrib/ai-sdk/` 코드 분석 (AI 에이전트 연동 패턴)
- [ ] "내 주변에서 자꾸 실패하는, 오래 걸리는 작업" 목록 작성 → 첫 제품 아이디어

---

## 📎 참고 자료

- [Temporal Cloud pricing | Temporal Documentation](https://docs.temporal.io/cloud/pricing)
- [About Temporal SDKs | Temporal Documentation](https://docs.temporal.io/encyclopedia/temporal-sdks)
- [Temporal Cloud vs Self-Hosted 2026: True Cost](https://automationatlas.io/guides/temporal-cloud-vs-self-hosted-2026/)
- [Temporal Pricing Guide - ZenML Blog](https://www.zenml.io/blog/temporal-pricing)
- [Best Durable Execution Platforms in 2026 · PandaStack](https://www.pandastack.ai/blog/best-durable-execution-platforms-2026/)
- [Temporal 플랫폼 한국어 문서](https://docs.temporal.kr/)
- [워크플로우 오케스트레이션 플랫폼 Temporal 알아보기 – BizSpring BLOG](https://blog.bizspring.co.kr/%ED%85%8C%ED%81%AC/%EC%9B%8C%ED%81%AC%ED%94%8C%EB%A1%9C%EC%9A%B0-%EC%98%A4%EC%BC%80%EC%8A%A4%ED%8A%B8%EB%A0%88%EC%9D%B4%EC%85%98-%ED%94%8C%EB%9E%AB%ED%8F%BC-temporal-%EC%95%8C%EC%95%84%EB%B3%B4%EA%B8%B0/)

---

_이 문서는 Claude Code와의 대화를 정리한 학습 노트입니다._
