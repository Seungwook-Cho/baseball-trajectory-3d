# Baseball Trajectory 3D

3D 야구장에서 타격 궤적을 시뮬레이션하고 시각화하는 인터랙티브 WebGL 프로토타입. 

https://baseball-proto.vercel.app

타격 위치 · 발사각 · 타구 속도(EV) · 방향각 4개 파라미터로 중력 기반 궤적을 계산해 렌더하고, 구장을 5개 zone 으로 부채꼴 분할해 착지 분포를 색상으로 시각화합니다. 단일 · 일괄(랜덤 5개) 시뮬레이션을 지원하며, 결과 테이블의 다중 선택 · zone 필터링이 야구장 하이라이트와 양방향으로 연동됩니다.

## 기술 스택

- **3D / WebGL**: Three.js 0.174, @react-three/fiber 8, @react-three/drei 9, meshline 3
- **프레임워크**: Next.js 14 (App Router), React 18, TypeScript
- **상태 관리**: Zustand 5
- **UI**: HeroUI, react-circular-slider-svg, Tailwind CSS
- **배포**: Vercel

## 3D 렌더링 / 시뮬레이션

**굵은 궤적 라인 렌더**
- WebGL 스펙상 `gl.LineWidth` 가 1 이상의 값을 무시해서, 기본 Three.js `<Line>` 으로는 궤적이 너무 가늘어 안 보이는 문제가 있었습니다. Line2 (LineGeometry + LineMaterial) 를 먼저 시도했지만 동적 업데이트 시 geometry 재바인딩이 불안정했습니다. meshline 으로 전환해, 매번 새로 생성되는 궤적과 포커스 시 굵기 · 색상 전환까지 안정적으로 렌더했습니다.

**65MB 야구장 모델 다운로드 진행률**
- 구매한 baseball.glb 가 65MB 라 첫 진입 시 다운로드가 길어 진행률을 표시했습니다. drei `useProgress` 는 파일 개수 단위라 단일 .glb 에서는 0% → 100% 두 번만 호출되어 카운트업이 보이지 않았고, `fetch` + `ReadableStream` reader 로 byte 단위 진행률을 직접 측정하도록 바꿨습니다.

- prod 에서 여전히 0 → 100 으로 점프했는데, 원인은 race 였습니다. `useGLTF.preload()` 가 모듈 로드 시점에 먼저 fetch 를 trigger 해 browser HTTP cache 를 채웠고, 직후 mount 된 progress fetch 가 cache hit 으로 65MB 를 한 chunk 에 받았습니다. preload 를 제거해 두 fetch 가 동시에 cold path 로 진행되도록 바꾼 뒤 카운트업이 정상 동작했습니다.

**클라이언트 전용 렌더 강제**
- Three.js · R3F 는 `window`, `document`, `WebGLRenderingContext` 같은 브라우저 전용 API 에 의존합니다. Next.js App Router 의 기본 SSR 단계에서 직접 import 하면 빌드 · 런타임 에러가 납니다. 메인 야구장 컴포넌트를 `dynamic(() => import(...), { ssr: false })` 로 감싸 페이지 진입 시점 클라이언트에서만 로드되게 했습니다.

**궤적 한 번에 렌더**
- 매 frame 마다 line geometry 에 점을 push 하면 GPU 업로드가 과부하되고 카메라가 비행 중인 공을 따라가야 합니다. 시뮬레이션 함수가 종료 시점까지의 전체 궤적을 Vector3 배열로 미리 계산해서 meshline 에 한 번에 전달하도록 만들었습니다. 일괄 5개도 동일하게 1회씩 GPU 업로드만 발생합니다.

**2D 그리드 ↔ 3D 월드 좌표 변환**
- 사용자는 8×11 그리드 (스트라이크 존 + 여백) 로 타격 위치를 지정하지만 시뮬레이션은 3D 월드 좌표를 입력으로 받습니다. 그리드 cell index 를 월드 좌표로 선형 매핑하고, 방향각은 `atan2` 로 단위 Direction Vector 로 바꿨습니다.

**Three.js 0.175 호환성 회피**
- drei `<Text>` 컴포넌트가 Three 0.175 에서 zone 라벨 · UI 텍스트가 깨지는 문제로 0.174 로 다운그레이드했습니다.

## 상태 관리 / 데이터 시각화

**trajectories ↔ zones ↔ focused state 동기화**
- 테이블 행 클릭 → 야구장 궤적 강조, 야구장 zone 클릭 → 테이블 필터링이 양방향이라 props drilling 으로 풀면 컴포넌트 간 결합도가 빠르게 올라갑니다. Zustand store 하나에 trajectories · zones · focused id 를 모두 넣고 컴포넌트마다 selector 로 구독했습니다. 어디서든 동일한 상태를 참조할 수 있고, 새 화면을 붙여도 props 변경 없이 연결됩니다.

**Zone 분할 + 빈도 기반 색상 랭킹**
- 단순 착지점만 시각화하면 "공이 어디로 많이 가는지" 패턴이 안 보입니다. 구장을 부채꼴 5분할 (-45°~45° 를 18° 간격) 하고, 각 zone 의 hit count 를 시뮬레이션 추가 때마다 재계산해 1 · 2 · 3위 zone 에 녹색 · 주황색 · 빨간색을 부여합니다.

**테이블 · 야구장 양방향 하이라이트**
- 결과가 누적되면 "지금 테이블에서 보고 있는 행이 야구장의 어느 궤적인지" 헷갈리기 시작합니다. focused id 가 있으면 해당 궤적은 빨강 · 굵게, 다른 궤적은 황금색 · 기본 굵기로 렌더해 테이블과 야구장의 시각적 연결을 유지합니다.

## UI / 인터랙션

**원형 슬라이더로 발사각 컨트롤**
- 발사각 (10°~85°) 은 0°~360° 가 아니라 수직 방향의 의미가 있는 각도라, 일반 horizontal slider 로는 "공이 어디로 날아갈지" 가 직관적으로 안 읽힙니다. react-circular-slider-svg 로 시계 방향 호를 그리고 SVG 화살표 아이콘으로 현재 발사 방향을 함께 표시했습니다. UI 자체가 결과의 미리보기 역할을 합니다.

**8×11 스트라이크 존 그리드**
- "스트라이크 존" 은 친숙한 좌표계입니다. 자유 좌표 입력 대신 클릭 가능한 8×11 cell 로 타격 위치를 선택하게 하고, 선택 상태를 시각화한 뒤 위의 2D ↔ 3D 좌표 매핑과 연동했습니다.

- 처음엔 spot 값을 단순 시작 좌표로만 썼는데, 야구장 스케일 대비 spot 범위가 너무 작아 결과가 거의 안 변했습니다. 중앙(스윗스팟)에서의 거리를 정규화해 EV 에 multiplier 를 적용하고, 좌우/상하 offset 으로 방향각·발사각에 shift 를 줘서 spot 이 결과 분포(비거리·존)에 실제로 반영되도록 했습니다.

