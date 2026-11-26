# EDGE RUN — Project Handoff

## 앱 개요

**앱 이름:** EDGE RUN (내부 작업명: 갓생)
**컨셉:** 사이버펑크 세계관(Edgerunner 스타일)의 습관 형성 앱. Life Reset 앱을 한국 사용자 맞춤으로 재해석.
**플랫폼 목표:** iOS PWA → 추후 React Native로 전환
**현재 상태:** 완성된 단일 HTML 프로토타입 (`gatsaeng-prototype.html`)

---

## 디자인 컨셉

### 사이버펑크 에지러너 테마
- **배경:** 거의 검정 (#07090D) + 미세 스캔라인 오버레이
- **Primary accent:** Electric Cyan (#00F0FF) — 주요 UI, 완료 상태
- **Secondary:** Hot Magenta (#FF003C) — Chrome Mode, 위험
- **Tertiary:** Electric Yellow (#FFE500) — bonus, streak
- **Purple:** #BD00FF — 특수 태그
- **텍스트:** Cool blue-white (#B8CDE0)

### 용어 체계 (한국어 ↔ 사이버펑크)
| 일반 | EDGE RUN |
|---|---|
| 미션/할일 | CONTRACT |
| XP | REP (Reputation) |
| 레벨 | GRADE |
| 하드 모드 | CHROME MODE |
| 연속 일수 | STREAK |
| 배지 | IMPLANT |
| 루틴 기록 | HABIT MATRIX |
| 성장 기록 | DATA LOG |
| 나 | PROFILE |

### 레벨 시스템 (Edgerunner 등급)
```
GR.1  🔌 NEWBIE          0 REP
GR.2  💻 NETRUNNER      200 REP
GR.3  🏙️ STREET KID    600 REP
GR.4  🔧 TECHIE        1200 REP
GR.5  ⚔️ SOLO          2000 REP
GR.6  🤝 FIXER         3200 REP
GR.7  ⚡ EDGERUNNER    5000 REP
GR.8  👁️ LEGEND        8000 REP
```

---

## 행동과학 설계 원칙

이 앱은 다음 행동과학 원리를 의도적으로 적용해서 설계됨:

### 현재 적용된 것
1. **Loss Aversion (손실 회피)** — Kahneman & Tversky
   - Chrome Mode: streak 초기화 위협. 숫자가 클수록 잃기 싫어서 행동하게 됨
2. **Endowed Progress Effect** — Nunes & Dreze (2006)
   - 퀘스트 젬(◆)이 하나씩 채워지는 방식. "7개 중 0개"가 아닌 "채워진 젬 N개" 프레이밍
3. **Variable Ratio Reinforcement** — Skinner
   - 미션 완료 시 20% 확률로 BONUS REP (1.5x) 등장. 슬롯머신 원리 → 가장 강한 습관 형성
4. **Goal Gradient Effect**
   - 완료 직전(N-1) 메시지가 가장 강렬 ("FINAL CONTRACT — DO NOT BREAK CHAIN")
5. **Commitment Device** — Ariely & Wertenbroch
   - 임시 할일 "내일로 미루기" = 단순 삭제가 아닌 약속. 내일 반드시 한다는 심리적 계약

### 추후 추가 고려 사항
- **Implementation Intentions** (Gollwitzer, 1999): 할일에 "언제 할건지" 필드 추가 → 완수율 2-3배 향상
- **Fresh Start Effect**: 주 단위 / 월 단위 리셋 메시지
- **Social Commitment**: 나중에 친구와 streak 공유 기능

---

## 데이터 모델 (localStorage key: `edgerun_v1`)

```javascript
DB = {
  user: {
    name: string,          // 사용자 이름
    startDate: "YYYY-MM-DD", // 66일 챌린지 시작일
    hardMode: boolean      // Chrome Mode on/off
  },
  asset: {
    current: number,       // 현재 자산 (억 단위)
    goal: number           // 목표 자산 (억 단위)
  },
  quests: [               // 할일 목록 (절대 삭제 안 함, soft delete)
    {
      name: string,
      tags: string[],      // DB.categories 중에서
      xp: number,          // REP 보상량
      deleted: boolean     // soft delete — 인덱스 보존용
    }
  ],
  categories: string[],    // 태그 목록 (기본: 아침,신체,정신,공부,재테크,기타)
  stats: {
    totalXP: number,       // 누적 REP
    totalMissions: number, // 누적 완료 미션 수
    currentStreak: number, // 현재 연속 일수
    longestStreak: number, // 최장 연속 기록
    level: number,         // 현재 레벨 인덱스 (0-7)
    earnedBadges: string[], // 획득한 badge ID 배열
    perfectDays: number,   // 하루 미션 100% 완료한 날 수
    habitDone: {           // 각 할일 누적 완료 횟수
      [questName]: number
    }
  },
  daily: {                 // 날짜별 데이터
    "YYYY-MM-DD": {
      done: number[],      // 완료한 quest 인덱스 배열
      allDone: boolean,    // 오늘 모든 미션 완료 여부
      tempTasks: [         // 하루 임시 할일
        { id: string, text: string, done: boolean }
      ]
    }
  },
  history: {               // 날짜별 완료율 (heatmap용)
    "YYYY-MM-DD": number   // 0-100
  }
}
```

---

## 핵심 로직

### 날짜 롤오버 (매일 앱 열 때 실행)
```javascript
// 1. 어제 미션 완료 여부로 streak 증감
// 2. Chrome Mode ON → 어제 미완료 시 streak 0으로 초기화
// 3. 어제의 미완료 임시 할일 → 오늘로 자동 이월
// 4. 어제 완료율을 history에 기록
```

### 개별 퀘스트 streak 계산
```javascript
// 각 퀘스트 인덱스에 대해 today부터 과거로 역순으로 체크
// done 배열에 해당 인덱스가 있는 연속 일수 = 그 퀘스트의 streak
```

### 레벨 계산
```javascript
// totalXP 기준으로 LEVELS 배열 역순 탐색 → 현재 레벨 결정
```

### Variable Reward
```javascript
// 미션 완료 시 Math.random() < 0.2 → 1.5x REP 보너스
// BONUS REP! 토스트 + 젬에 애니메이션 표시
```

---

## 화면 구조 (4개 탭)

### ⚡ TODAY
1. 헤더 (날짜, Grade badge)
2. 동기부여 카드 (progress 기반 텍스트 변화)
3. **퀘스트 젬 Row** — 핵심 UX. 미션 수만큼 젬이 나열되고 완료 시 cyan glow
4. 66일 진행 바 (DAY N / 66)
5. 목표 자산 트래커 (탭하면 수정)
6. Stats strip (STREAK / DONE / REP)
7. Chrome Mode 배너 (토글 가능)
8. DAILY CONTRACTS — 할일 목록 (체크 시 ▲ 표시, 개별 streak 🔥N 표시)
9. **TODAY ONLY** — 임시 할일 (추가/완료/내일로 미루기/삭제)

### 🔗 MATRIX (루틴)
- 카테고리 필터 칩
- 각 루틴 카드: 7일 도트, 연속 streak, REP

### 📊 LOG (기록)
- Grade 카드 (REP bar)
- Grade roadmap (8단계)
- 주요 통계 4개
- Activity grid (91일 heatmap)
- Implant collection (12개 배지)

### 👤 PROFILE (나)
- 이름 편집
- 주요 통계
- 설정: 이름변경, 할일관리, Chrome Mode, 자산수정, 데이터내보내기, 초기화

---

## 할일 관리 오버레이
- 오른쪽에서 슬라이드인
- 카테고리 태그 추가/삭제
- 할일 추가/수정/삭제 (soft delete)
- 각 할일에 현재 streak 표시

---

## 기본 할일 (DEFAULT_Q)
```
오전 6시 기상  [아침]       20 REP
물 2L 마시기   [신체]       20 REP
30분 운동      [신체]       30 REP
독서 30분      [정신]       25 REP
영어 공부 20분 [공부]       25 REP
10분 명상      [정신]       15 REP
오늘 일기 작성 [정신]       15 REP
```

---

## Implant (배지) 조건
```
FIRST RUN    — 미션 1개 완료
7 STREAK     — 7일 연속
30 STREAK    — 30일 연속
50 JOBS      — 미션 50개 완료
200 JOBS     — 미션 200개 완료
1K REP       — 누적 REP 1000
5K REP       — 누적 REP 5000
GRADE 3      — STREET KID 달성
EDGERUNNER   — EDGERUNNER 달성
PERFECT DAY  — 하루 미션 100% 완료
CREDIT 1억+  — 자산 1억 이상 입력
66 DAYS      — 66일 챌린지 완주
```

---

## React Native 전환 시 주요 변경사항

| HTML/JS | React Native |
|---|---|
| `localStorage` | `@react-native-async-storage/async-storage` |
| CSS animations | `Animated` API or `react-native-reanimated` |
| SVG | `react-native-svg` |
| `position: absolute` overlay | Modal component |
| `window.addEventListener('load')` | `useEffect(() => {}, [])` |
| `confirm()` | `Alert.alert()` |
| `prompt()` | Custom TextInput modal |
| File download (export) | `expo-sharing` + `expo-file-system` |

---

## 미완성 / 추후 작업

- [ ] Apple Health 연동 (걸음수, 수면, 운동)
- [ ] 푸시 알림 (오늘 미션 리마인더)
- [ ] 목표 자산 수동 입력 외 → 연동 (후순위)
- [ ] 친구와 streak 비교 (소셜 기능)
- [ ] Implementation Intentions 적용 ("언제 할건지" 필드)
- [ ] 주간/월간 리포트
- [ ] 66일 완주 보상 화면

---

## 파일
- `gatsaeng-prototype.html` — 완성된 싱글 HTML 앱 (모든 로직 포함)
- localStorage key: `edgerun_v1`
