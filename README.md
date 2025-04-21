# 과일 퀴즈 점수 보기 (Fruit Quiz Viewer)

과일 퀴즈 점수를 방 ID(room_id)별로 확인할 수 있는 Vue.js 3 웹 애플리케이션입니다.

## 기능

- URL의 room_id 파라미터를 기반으로 해당 방에 있는 사용자들의 과일 리스트 표시
- Supabase의 quiz_score 테이블에서 데이터 가져오기
- 각 사용자별 과일 수집 현황 시각화

## 설치 및 실행

1. 프로젝트 클론 후 종속성 설치:
```bash
npm install
```

2. `.env` 파일에 Supabase 정보 설정:
```
VITE_SUPABASE_URL=your_supabase_url
VITE_SUPABASE_ANON_KEY=your_supabase_anon_key
```

3. 개발 서버 실행:
```bash
npm run dev
```

4. 빌드:
```bash
npm run build
```

## Supabase 테이블 구조

`quiz_score` 테이블에는 다음 필드가 필요합니다:

- `id`: 고유 식별자
- `room_id`: 방 ID
- `username`: 사용자 이름
- `fruit_list`: 과일 이모지 목록 (예: "💛x1 🍌🍏🍅🍒🍇...")

## 사용 방법

1. 홈 페이지에서 room ID를 입력하고 입장 버튼을 클릭합니다.
2. 해당 방에 있는 모든 사용자의 과일 수집 현황이 표시됩니다.
