# Baseball Trajectory 3D

https://baseball-proto.vercel.app

3D 야구장에서 타격 궤적을 시뮬레이션하고 시각화하는 인터랙티브 WebGL 프로토타입입니다.
타격 위치, 발사각, 타구 속도(EV), 방향각을 입력받아 중력 기반 궤적을 계산합니다.
계산된 궤적은 3D 야구장 위에 렌더링하고, 착지 지점은 5개 부채꼴 zone 으로 분류해 색상으로 시각화합니다.
단일 시뮬레이션과 랜덤 5개 일괄 시뮬레이션을 지원합니다.

## 스크린샷
<img width="3442" height="1828" alt="Group 1" src="https://github.com/user-attachments/assets/16e3779c-7cf8-46f6-8f33-e6cae3c02d87" />

## 기술 스택

- **3D / WebGL**: Three.js 0.174, @react-three/fiber 8, @react-three/drei 9, meshline 3
- **프레임워크**: Next.js 14 (App Router), React 18, TypeScript
- **상태 관리**: Zustand 5
- **UI**: HeroUI, react-circular-slider-svg, Tailwind CSS
- **배포**: Vercel

## 3D 렌더링 / 시뮬레이션

**굵은 궤적 라인 렌더**
- WebGL 환경에서는 `gl.LineWidth` 값이 대부분 1로 고정되어 기본 Three.js `<Line>` 만으로는 궤적이 지나치게 가늘게 보였습니다. 처음에는 Line2(LineGeometry + LineMaterial)를 적용했지만, 궤적이 새로 생성될 때마다 선이 정상적으로 갱신되지 않는 문제가 있었습니다. 최종적으로 meshline 으로 전환해 새로 생성되는 궤적과 포커스 상태의 굵기 · 색상 변경을 안정적으로 렌더링했습니다.

**65MB 야구장 모델 다운로드 진행률**
- 구매한 `baseball.glb` 모델이 약 65MB라 첫 진입 시 로딩 시간이 길었습니다. 초기에는 drei `useProgress` 를 사용했지만, 단일 `.glb` 파일에서는 진행률이 0%에서 100%로 바로 점프해 실제 다운로드 상태를 보여주기 어려웠습니다. `fetch` 와 `ReadableStream` reader 를 사용해 byte 단위 진행률을 직접 계산하도록 변경했습니다.

- 배포 환경에서도 진행률이 0%에서 100%로 점프하는 문제가 남아 있었고, 원인은 `useGLTF.preload()` 로 인한 fetch 경합이었습니다. 모듈 로드 시점의 preload 가 먼저 browser HTTP cache 를 채운 뒤, mount 이후 실행된 진행률 측정용 fetch 가 cache hit 으로 파일을 한 번에 받아 진행률이 생략됐습니다. preload 를 제거해 진행률 측정용 fetch 가 캐시 적용 전의 다운로드 과정을 직접 측정하도록 바꿨고, 이후 다운로드 카운트업이 정상적으로 표시됐습니다.
  
**클라이언트 전용 렌더 강제**
- Three.js 와 R3F 는 `window`, `document`, `WebGLRenderingContext` 같은 브라우저 API 에 의존합니다. Next.js App Router 의 SSR 단계에서 직접 import 하면 빌드 또는 런타임 에러가 발생했습니다. 메인 야구장 컴포넌트를 `dynamic(() => import(...), { ssr: false })` 로 감싸 클라이언트 환경에서만 로드되도록 분리했습니다.

**궤적 한 번에 렌더**
- 매 frame 마다 line geometry 에 점을 추가하면 GPU 업로드가 반복되고, 카메라가 비행 중인 공을 계속 추적해야 했습니다. 시뮬레이션 함수가 종료 시점까지의 전체 궤적을 `Vector3` 배열로 먼저 계산하고, meshline 에 한 번만 전달하도록 구성했습니다. 일괄 5개 시뮬레이션에서도 각 궤적당 1회 GPU 업로드만 발생하도록 처리했습니다.
  
**2D 그리드 ↔ 3D 월드 좌표 변환**
- 사용자는 8×11 그리드에서 타격 위치를 선택하지만, 시뮬레이션은 3D 월드 좌표를 입력으로 사용합니다. 그리드 cell index 를 월드 좌표로 선형 매핑하고, 방향각은 `atan2` 기반 단위 direction vector 로 변환했습니다.

**Three.js 0.175 호환성 회피**
- Three.js 0.175 환경에서 drei `<Text>` 컴포넌트의 zone 라벨과 UI 텍스트가 깨지는 문제가 있었습니다. 라벨 렌더링 안정성을 우선해 Three.js 버전을 0.174 로 고정했습니다.

## 상태 관리 / 데이터 시각화

**trajectories ↔ zones ↔ focused state 동기화**
- 테이블 행 클릭은 야구장 궤적 강조로 이어지고, 야구장 zone 클릭은 테이블 필터링으로 이어집니다. 이 흐름을 props drilling 으로 연결하면 컴포넌트 간 결합도가 빠르게 커질 수 있었습니다. Zustand store 에 trajectories, zones, focused id 를 모으고, 각 컴포넌트는 selector 로 필요한 상태만 구독하도록 구성했습니다. 그 결과 테이블, 야구장, 필터 UI 가 동일한 상태를 공유하면서도 props 변경 없이 확장할 수 있게 됐습니다.
  
**Zone 분할 + 빈도 기반 색상 랭킹**
- 착지점만 표시하면 타구가 어느 방향으로 많이 분포하는지 파악하기 어렵습니다. 구장을 -45°부터 45°까지 18° 간격의 5개 부채꼴 zone 으로 나눴습니다. 시뮬레이션이 추가될 때마다 zone 별 hit count 를 다시 계산하고, 1 · 2 · 3위 zone 에 각각 녹색 · 주황색 · 빨간색을 부여했습니다.

**테이블 · 야구장 양방향 하이라이트**
- 결과가 누적되면 테이블에서 선택한 행이 야구장의 어느 궤적인지 구분하기 어려워집니다. focused id 가 있는 경우 해당 궤적은 빨간색 · 굵은 라인으로 표시하고, 나머지 궤적은 황금색 · 기본 굵기로 유지했습니다. 이를 통해 테이블 선택 상태와 3D 야구장 시각화를 한눈에 연결할 수 있게 했습니다.
  
## UI / 인터랙션

**원형 슬라이더로 발사각 컨트롤**
- 발사각은 단순 수치보다 공이 위로 떠오르는 방향을 함께 보여줄 때 더 직관적으로 이해됩니다. 일반 horizontal slider 대신 `react-circular-slider-svg` 를 사용해 시계 방향 호 형태의 컨트롤을 구성했습니다. 현재 발사 방향은 SVG 화살표로 함께 표시해, 입력 UI 자체가 결과의 미리보기 역할을 하도록 만들었습니다.

**8×11 스트라이크 존 그리드**
- 타격 위치는 자유 좌표 입력 대신 클릭 가능한 8×11 스트라이크 존 그리드로 선택하게 했습니다. 선택된 cell 은 시각적으로 강조하고, 내부적으로는 2D grid index 를 3D 월드 좌표로 변환해 시뮬레이션 입력값으로 사용했습니다.

- 초기에는 spot 값을 공의 시작 좌표로만 사용해, 야구장 스케일 대비 결과 변화가 거의 보이지 않았습니다. 중앙 스윗스팟과의 거리를 정규화해 EV multiplier 를 적용하고, 좌우 · 상하 offset 을 방향각과 발사각 보정값으로 변환했습니다. 그 결과 타격 위치에 따라 비거리와 착지 zone 이 달라지도록 만들었습니다.

