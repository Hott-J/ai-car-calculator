# 멀티 에이전트 아키텍처 설계

## 🎯 전체 구조

```
[Webhook] 사용자 입력
    ↓
[Router Agent] 의도 파악 (어떤 에이전트로 보낼지만 결정)
    ↓
[Switch Node] type에 따라 분기
    ↓
    ├─ type="maintenance" → [Maintenance Agent] → [Calculate & Merge Tool]
    ├─ type="price" → [Price Agent] → [Web Search Tool / API]
    ├─ type="compare" → [Compare Agent] → [Database / Memory]
    └─ type="consultation" → [Consultation Agent] → [Knowledge Base]
    ↓
[Respond to Webhook]
```

---

## 🤖 1. Router Agent (라우터)

**역할:** 사용자 의도만 파악 (최대한 간단하게)

**프롬프트:**
```
사용자 질문을 분석하여 다음 중 하나로 분류하세요:

1. "maintenance" - 유지비, 월비용, 관리비 질문
2. "price" - 가격, 시세, 얼마 질문
3. "compare" - vs, 비교, 차이, 뭐가 나아 질문
4. "consultation" - 살까, 추천, 고민, 조언 질문

JSON 형식으로만 응답:
{
  "type": "maintenance" | "price" | "compare" | "consultation",
  "query": "원본 질문"
}

예시:
입력: "아반떼 유지비 얼마야?"
출력: {"type": "maintenance", "query": "아반떼 유지비 얼마야?"}
```

**장점:**
- 매우 빠른 응답 (간단한 분류만)
- 오류 가능성 낮음
- 다른 에이전트에 영향 없음

---

## 🤖 2. Maintenance Agent (유지비 전문)

**역할:** 유지비 계산을 위한 차량 정보 추출

**프롬프트:**
```
사용자가 차량 유지비를 물어봤습니다.
다음 정보를 추출하여 JSON으로 응답하세요:

{
  "manufacturer": "제조사",
  "model": "모델명",
  "year": "연식" (없으면 null)
}

예시:
입력: "BMW 520d 2020년식 유지비"
출력: {
  "manufacturer": "BMW",
  "model": "520d",
  "year": "2020"
}
```

**연결 Tool:**
- Calculate & Merge 노드 (기존)

**확장 가능성:**
- 실제 자동차세 API 연동
- 보험료 비교 API
- 지역별 유류비 계산

---

## 🤖 3. Price Agent (시세 전문)

**역할:** 중고차 시세 조회 및 분석

**프롬프트:**
```
사용자가 중고차 시세를 물어봤습니다.

1. 차량 정보 추출
2. 시세 예측
3. 시장 분석

JSON 응답:
{
  "manufacturer": "제조사",
  "model": "모델명",
  "year": "연식",
  "estimatedPrice": "예상 가격대",
  "priceAnalysis": "시세 분석",
  "marketTrend": "상승세 | 하락세 | 안정세"
}
```

**연결 Tool:**
- Web Search Tool (실시간 시세 검색)
- 헤이딜러 API (가능하면)
- 가격 예측 모델

**확장 가능성:**
- 실제 매물 크롤링 (선택적)
- 가격 히스토리 차트
- 지역별 시세 차이

---

## 🤖 4. Compare Agent (비교 전문)

**역할:** 2개 이상 차량 비교 분석

**프롬프트:**
```
사용자가 차량 비교를 요청했습니다.

각 차량의:
1. 장단점 분석
2. 유지비 비교
3. 시세 비교
4. 재판매 가치
5. 상황별 추천

JSON 응답:
{
  "cars": [
    {
      "name": "차량명",
      "pros": ["장점1", "장점2"],
      "cons": ["단점1", "단점2"],
      "maintenanceCost": "월 XX만원",
      "price": "X천만원대"
    }
  ],
  "winner": "상황별 추천",
  "recommendation": "구체적 조언"
}
```

**연결 Tool:**
- Maintenance Agent 호출 (유지비)
- Price Agent 호출 (시세)
- Database (과거 비교 데이터)

**확장 가능성:**
- 사용자 선호도 학습
- 비교 히스토리 저장
- PDF 리포트 생성

---

## 🤖 5. Consultation Agent (상담 전문)

**역할:** 구매 조언 및 종합 상담

**프롬프트:**
```
사용자가 차량 구매 상담을 요청했습니다.

다음을 종합 분석:
1. 차량의 장단점
2. 사용자 상황 추론
3. 대안 제시
4. 구매 타이밍 조언

JSON 응답:
{
  "verdict": "recommend | caution | not_recommend",
  "reasons": ["이유1", "이유2"],
  "userSituation": "추론된 사용자 상황",
  "advice": "구체적 조언",
  "alternatives": ["대안1", "대안2"],
  "timing": "지금 사기 좋음 | 조금 더 기다리세요"
}
```

**연결 Tool:**
- Knowledge Base (차량별 특성 DB)
- 시장 동향 데이터
- 사용자 히스토리

**확장 가능성:**
- 개인 맞춤 추천
- 구매 체크리스트
- 딜러 협상 팁

---

## 🔀 Switch Node 설정

**n8n Switch 노드:**

```javascript
// Router Agent의 출력을 받아서 분기

switch ($json.type) {
  case "maintenance":
    return { output: 0 }; // Maintenance Agent로
  case "price":
    return { output: 1 }; // Price Agent로
  case "compare":
    return { output: 2 }; // Compare Agent로
  case "consultation":
    return { output: 3 }; // Consultation Agent로
  default:
    return { output: 4 }; // Error 처리
}
```

---

## 📊 워크플로우 비주얼

```
┌─────────────┐
│   Webhook   │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Router Agent   │ ← 의도 파악만 (빠름)
│  "type 결정"    │
└────────┬────────┘
         │
         ▼
   ┌────────────┐
   │ Switch Node │
   └─┬──┬──┬──┬─┘
     │  │  │  │
  ┌──┘  │  │  └──┐
  ▼     ▼  ▼     ▼
┌─────┐┌─┐┌─┐┌──────┐
│Maint││P││C││Consul│ ← 각 전문 Agent
│Agent││r││o││Agent │
└──┬──┘│i││m│└───┬──┘
   │   │c││p│    │
   │   │e││a│    │
   ▼   │ ││r│    ▼
┌──────┐│ ││e│┌──────┐
│Calc &││ │└─┤│Know  │ ← 각 Tool
│Merge ││ │  ││Base  │
└──┬───┘│ │  │└───┬──┘
   │    ▼ ▼  ▼    │
   │  ┌────────┐  │
   └─→│ Format │←─┘ ← 최종 포맷팅
      │ Output │
      └───┬────┘
          ▼
    ┌──────────┐
    │ Response │
    └──────────┘
```