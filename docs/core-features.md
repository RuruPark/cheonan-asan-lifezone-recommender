# 핵심기능 기술설명

## 1. 공공데이터 기반 생활권 데이터

`src/data/generatedLifeZones.js`는 전처리 CSV를 바탕으로 생성된 천안·아산 생활권 48개 데이터를 담고 있습니다. 각 생활권에는 교통, 생활 편의, 안전·의료 점수와 인프라 count, 태그, 리 단위 보조 정보가 포함됩니다.

`src/data/cheonanAsanEmdBoundaries.js`는 국토교통부 센서스경계 기반 행정동 경계 FeatureCollection입니다. 천안·아산 48개 행정동 경계를 지도 표시와 좌표 기반 매칭에 사용합니다.

`src/data/lifeZoneRepository.js`, `src/data/lifeZoneDataMode.js`, `src/data/workplaceOptions.js`는 generated 모드와 mock 모드를 분리하고, 실제 generated 데이터가 있는 경우 이를 우선 사용하도록 구성합니다.

## 2. 추천 알고리즘

추천은 교통, 생활 편의, 안전·의료 3축을 중심으로 계산됩니다. 사용자가 선택한 중요도는 낮음 0.5, 보통 1.0, 높음 1.5 계수로 변환된 뒤 정규화되어 `totalScore`에 반영됩니다.

통근 조건이 있는 경우 `src/utils/commuteScoring.js`가 `baseScore`, `commuteFitScore`, `commuteWeight`를 결합해 `finalScoreWithCommute`를 계산합니다. 통근 중요도별 반영 비중은 낮음 10%, 보통 20%, 높음 30%입니다.

희망 통근시간을 초과하면 실제 API 성공값에 한해 초과 패널티를 적용합니다. 보통/높음 중요도에서는 30분 이상 초과 후보가 전부인 경우 no-result 상태를 만들 수 있습니다.

도보 추천은 TMAP 실제 경로 결과가 60분을 초과하면 TOP 추천 후보에서 제외됩니다. 이 제한은 도보 통근 현실성을 제어하기 위한 하드 캡입니다.

## 3. API 연동

자동차는 `api/commute/driving-batch.js`와 `api/commute/_driving-core.js`에서 Naver Directions 5를 서버리스로 호출합니다. Client Secret은 서버리스 환경변수에서만 사용하도록 설계되어 있습니다.

대중교통은 `src/utils/odsayTransitApi.js`에서 ODsay URI/Web API를 호출하고, 성공한 실제 경로 소요시간만 점수에 반영합니다.

도보는 `api/commute/walking-batch.js`에서 TMAP 보행자 경로 API를 서버리스로 호출합니다. appKey는 서버리스 환경변수에서만 사용하도록 구성했습니다.

세 API 모두 실패 시 `durationMinutes`를 `null`로 유지하고, 거리 기반 가짜 시간을 선택된 통근수단의 실제 결과처럼 표시하지 않습니다.

## 4. API 호출량 제한

`src/utils/commutePreselection.js`는 전체 48개 생활권을 먼저 proxy 점수로 평가해 외부 API 호출 대상을 제한합니다.

- 자동차: 추천 shortlist 최대 10개 + 비추천 후보 1개, 최대 11회
- 대중교통: 추천 shortlist 최대 12개 + 비추천 후보 1개, 최대 13회
- 도보: 추천 shortlist 최대 10개 + 비추천 후보 1개, 최대 11회

preselection은 `preApiScore`, `proxyCommuteScore`, `baseInfraScore`를 조합해 API 호출량과 추천 품질의 균형을 맞춥니다.

## 5. 지도 시각화

`src/components/NaverMapView.js`는 네이버 지도 위에 천안·아산 행정동 경계, 추천 생활권 마커, 비추천 생활권 마커, 직장 마커, 직장과 생활권 연결선을 표시합니다.

네이버 지도 Client ID가 없거나 지도 로딩에 실패하면 `src/components/app.js`의 fallback 지도 UI가 표시됩니다. 직장 위치를 `선택안함`으로 두면 직장 마커와 연결선 없이 인프라 추천 결과만 표시됩니다.

## 6. 테스트 및 배포

`tests` 폴더에는 데이터 모드, 추천 점수, 통근 preselection, 자동차 API, ODsay API, TMAP API, 도보 60분 제한, 통근시간 초과 패널티, infraOnly 모드 등을 검증하는 Node 테스트가 포함됩니다.

`scripts/verify-static-app.mjs`는 핵심 파일 존재 여부, 보안상 금지된 Secret 노출, 지도/추천/API 기능 연결을 정적으로 검증합니다.

`scripts/build-vercel-static.mjs`는 Vercel 배포용 `dist`를 생성하는 스크립트입니다.

`vercel.json`은 Vercel 빌드 시 정적 검증 후 `dist`를 생성하는 배포 구조를 보여줍니다. GitHub Pages 같은 정적 배포는 UI 시연에는 사용할 수 있으나 서버리스 API가 필요한 기능은 Vercel 구조가 더 적합합니다.
