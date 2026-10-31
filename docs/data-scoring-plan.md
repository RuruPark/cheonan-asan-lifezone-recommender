# 생활권 데이터 전처리 및 점수산정 계획

## 1. 현재 단계

- 개발 초기에는 UI와 점수산정 구조를 먼저 검증하기 위해 mock 데이터(`src/data/mockLifeZones.js`)를 사용했다.
- 현재는 전처리 CSV로 생성한 `src/data/generatedLifeZones.js`를 기본 데이터로 사용하며, mock 데이터는 fallback으로만 남겨 두었다.
- 실제 데이터 기반 생성 방식은 10장, 실제 길찾기 API 연동은 11장에 정리했다.

## 2. 생활권 단위

- 생활권 단위는 읍·면·동이다.
- 대표 좌표는 가능하면 행정구역 polygon 내부 대표점을 사용한다.
- mock 데이터에서는 생활권별 `lat`/`lng` 좌표를 사용한다.
- 면적은 `areaKm2`로 관리하며, 밀도 지표는 시설 수를 면적으로 나누어 산정한다.

## 3. 활용 예정 공공데이터

교통 인프라:

- 전국 도시철도역사 정보
- 국토교통부 전국 버스정류장 정보

생활 편의 인프라:

- 충청남도 도서관 현황
- 충청남도 작은도서관 현황
- 충청남도 체육관련 비영리법인 현황

치안·의료 인프라:

- 충청남도 약국 현황
- 충청남도 보안등 정보
- 충청남도 안전비상벨 위치 정보
- 충청남도 119안전센터 현황
- 경찰청 전국 지구대·파출소 주소현황

## 4. 전처리 방향

- 주소 기반 데이터는 좌표화한다.
- 시설 point 데이터는 읍·면·동 polygon 내부 포함 여부로 집계한다.
- 거리 기반 지표는 생활권 대표 좌표에서 가장 가까운 시설까지의 거리로 계산한다.
- 밀도 기반 지표는 생활권 내 시설 수 / 면적(km²)로 계산한다.
- 거리값은 가까울수록 높은 점수로 변환한다.
- 밀도값은 p5~p95 기준으로 0~100점 정규화한다.
- 결측값은 데이터 성격에 따라 0, `null`, 중앙값, 또는 `unknown`으로 처리한다.
- 극단값은 percentile clipping을 적용한다.
- 최종 UI에는 원천값 전체를 직접 노출하지 않고 0~100 점수와 설명형 요약을 제공한다.

## 5. 점수산정 산식

### 사용자 중요도 가중치

중요도 계수:

- 낮음: 0.5
- 보통: 1.0
- 높음: 1.5

정규화:

```text
transportWeight = transportCoefficient / (transportCoefficient + livingCoefficient + safetyMedicalCoefficient)
livingWeight = livingCoefficient / (transportCoefficient + livingCoefficient + safetyMedicalCoefficient)
safetyMedicalWeight = safetyMedicalCoefficient / (transportCoefficient + livingCoefficient + safetyMedicalCoefficient)
```

### 거리점수

```text
distanceScore = 100 * max(0, 1 - distanceKm / (2 * standardDistanceKm))
```

- 거리값이 `null`, `undefined`, `NaN`이면 0점이다.
- 기준거리가 0 이하이면 0점이다.
- 결과는 0~100 범위로 보정한다.

### 밀도점수

```text
density = facilityCount / areaKm2
densityScore = 100 * (zoneDensity - p5Density) / (p95Density - p5Density)
clampedDensityScore = min(100, max(0, densityScore))
```

- `areaKm2`가 0 이하이면 밀도는 0으로 처리한다.
- `p95Density`와 `p5Density`가 같으면 모든 값에 50점 또는 0점을 일관되게 적용한다.

### 교통 인프라 점수

```text
railAccessibilityScore = 100 * max(0, 1 - nearestRailDistanceKm / 3.0)
busAccessibilityScore = 100 * max(0, 1 - nearestBusStopDistanceKm / 1.0)
transportDiversityScore = (busWithin500m ? 50 : 0) + (railWithin1_5km ? 50 : 0)

transportScore =
  45 * railAccessibilityScore / 100
  + 45 * busAccessibilityScore / 100
  + 10 * transportDiversityScore / 100
```

### 생활 편의 인프라 점수

```text
nearestLibraryDistanceScore = 100 * max(0, 1 - nearestLibraryDistanceKm / 3.0)
libraryAccessibilityScore = 0.7 * nearestLibraryDistanceScore + 0.3 * libraryDensityScore

nearestSportsInfraDistanceScore = 100 * max(0, 1 - nearestSportsInfraDistanceKm / 3.0)
sportsInfraAccessibilityScore = 0.7 * nearestSportsInfraDistanceScore + 0.3 * sportsInfraDensityScore

livingDiversityScore = 100 * matchedTypeCount / 3

livingScore =
  45 * libraryAccessibilityScore / 100
  + 45 * sportsInfraAccessibilityScore / 100
  + 10 * livingDiversityScore / 100
```

### 치안·의료 인프라 점수

```text
nearestPharmacyDistanceScore = 100 * max(0, 1 - nearestPharmacyDistanceKm / 2.0)
pharmacyAccessibilityScore = 0.7 * nearestPharmacyDistanceScore + 0.3 * pharmacyDensityScore

fire119AccessibilityScore = 100 * max(0, 1 - nearest119CenterDistanceKm / 8.0)
policeSubstationAccessibilityScore = 100 * max(0, 1 - nearestPoliceSubstationDistanceKm / 8.0)

safetyMedicalScore =
  25 * pharmacyAccessibilityScore / 100
  + 20 * streetlightDensityScore / 100
  + 5 * emergencyBellDensityScore / 100
  + 20 * fire119AccessibilityScore / 100
  + 30 * policeSubstationAccessibilityScore / 100
```

### 최종 생활권 적합도 점수

```text
totalScore =
  userTransportWeight * transportScore
  + userLivingWeight * livingScore
  + userSafetyMedicalWeight * safetyMedicalScore
```

### 상대등급 산정 기준

- A: 상위 20%, 매우 적합
- B: 상위 20~40%, 적합
- C: 상위 40~60%, 보통
- D: 상위 60~80%, 다소 부족
- F: 하위 20%, 부적합 또는 개선 필요

등급은 전체 `scoredLifeZones` 기준으로 계산하며, 화면에 표시되는 추천 2개와 비추천 1개만으로 재계산하지 않는다.

## 6. 현재 UI 구현과 실제 데이터 적용의 차이

- `getLifeZoneDataset()`은 실제 공공데이터 기반 `generatedLifeZones`를 우선 반환하고, 데이터가 없을 때만 `mockLifeZones`를 사용한다.
- 원천 공공데이터는 전처리 후 mock 데이터와 동일한 `LifeZone` 구조로 변환한다.
- UI 컴포넌트는 데이터 출처가 mock인지 실제 공공데이터인지와 무관하게 동일하게 동작한다.
- UI는 데이터 저장소를 통해 받은 `lifeZones → calculateLifeZoneScores → assignRelativeGrades → getTopAndLowZones → ResultMap / ResultSidePanel` 흐름을 따른다.

## 7. 추후 확장 방향

- 생활권 군집화
- 인구·이동 데이터 반영
- 추천 사유 설명 문구 고도화
- B2G/B2B 확장

## 8. 통근 점수 결합 방식

- 통근 점수는 고정 구간표가 아니라 사용자가 입력한 희망 통근시간을 기준으로 계산한다.
- 실제 또는 추정 통근시간이 희망 통근시간 이내이면 통근 적합도는 100점이다.
- 희망 통근시간을 초과하면 `max(0, actualMinutes - targetMinutes) / targetMinutes` 비율에 따라 완만하게 감점한다.
- 통근 중요도는 낮음, 보통, 높음으로 나뉘며 반영 비중은 각각 10%, 20%, 30%다.
- 통근 중요도가 높아도 기존 교통, 생활 편의, 치안·의료 인프라 3축 점수는 최소 70% 이상 반영된다.
- API 연동 전에는 `src/utils/commuteEstimator.js`의 거리 기반 fallback 통근시간을 사용한다.
- 통근 점수 결합 단계에서는 지도 API 호출, 네이버 Directions 5 API 호출, VWorld GeoJSON 표시는 수행하지 않았다.

## 9. 읍면동 경계 데이터 적용 방식

- 현재 경계 데이터는 `src/data/cheonanAsanEmdBoundaries.js`에 둔 국토교통부 센서스경계 기반 실제 행정동경계 데이터다.
- `metadata.source`는 `molit-census-boundary`, `metadata.isSample`은 `false`다.
- 실제 VWorld GeoJSON을 확보하면 `scripts/prepare-vworld-boundaries.mjs`로 천안·아산 읍면동만 추출해 같은 FeatureCollection 구조로 변환할 수 있다.
- 국토교통부 센서스경계 행정동경계 GeoJSON은 `scripts/prepare-admin-boundaries.mjs`로 같은 구조로 변환할 수 있다.
- 현재 `data/행정동경계`의 원본은 SHP 묶음이며 `.prj` 기준 KGD2002 Central Belt 2010 투영좌표계다. 네이버 지도 적용을 위해 WGS84(EPSG:4326) GeoJSON으로 변환했고, 변환 결과는 `data/행정동경계/converted/admin_emd_4326.geojson`에 둔다.
- 루트 `data/` 폴더는 Git에서 제외될 수 있으므로 VWorld 원본 보관용으로 적합하고, 웹앱이 import하는 최종 JS 데이터는 `src/data/cheonanAsanEmdBoundaries.js`에 둔다.
- SHP 파일은 스크립트에서 직접 처리하지 않고, 먼저 GeoJSON FeatureCollection으로 변환한 뒤 입력한다.
- 실제 경계 데이터로 변환되면 `metadata.isSample`은 `false`가 된다.
- 변환 실패, 좌표계 불확실, 천안·아산 feature 미식별 상태에서는 기존 샘플 경계 데이터를 유지한다.
- 네이버 지도 Client ID가 있으면 지도 위에 직장 읍면동 경계와 매칭되는 생활권 경계를 표시한다.
- Client ID가 없거나 지도 로딩에 실패하면 기존 fallback 지도 UI를 유지한다.

## 10. 실제 CSV 기반 생활권 점수 생성 방식

- `scripts/generate-life-zones-from-csv.mjs`는 `data/전처리파일(csv)` 폴더의 CSV를 자동 탐색해 `src/data/generatedLifeZones.js`를 생성한다.
- CSV 유형과 추천 축은 `src/data/infrastructureCsvConfig.js`에서 관리한다.
- 주소 기반 데이터는 `src/utils/addressParser.js`로 천안·아산의 시, 구, 읍면동, 리 정보를 추출한 뒤 행정동 경계와 매칭한다.
- 버스정류장 데이터는 위도/경도 좌표를 `src/utils/pointInPolygon.js`로 실제 행정동 polygon에 매칭한다.
- 인프라 count는 항목별 `log1p` 변환 후 전체 행정동 기준 min-max 정규화한다.
- 교통 인프라는 버스정류장 70%, 도시철도역사 30%를 반영한다.
- 생활 편의 인프라는 도서관/작은도서관 60%, 체육 관련 시설 40%를 반영한다.
- 치안 의료 인프라는 병원 20%, 약국 20%, 119안전센터 15%, 지구대/파출소 15%, 보안등 15%, 알람벨 10%, 실내구호소 5%를 반영한다.
- `baseScore`는 교통 35%, 생활 편의 35%, 치안 의료 30%로 계산한다.
- 리 정보가 있는 주소 데이터는 `riHighlights`로 보조 집계하지만, 리 경계 데이터가 없으므로 지도 polygon은 행정동 단위로 유지한다.
- `getLifeZoneDataset()`은 `generatedLifeZones`가 있으면 실제 CSV 기반 데이터를 우선 사용하고, 없거나 비어 있으면 `mockLifeZones`를 fallback으로 사용한다.
- 결과 카드는 `riHighlights`가 있는 생활권에 한해 주요 리 인프라를 최대 3개까지 표시한다.

## 11. 통근 현실성 필터와 실제 길찾기 API 확장

- 초기에는 `src/utils/commuteEstimator.js`의 거리 기반 추정값으로 통근시간을 계산했다. 현재는 실제 길찾기 API가 성공한 값만 통근시간으로 표시하고 점수에 반영하며, 거리 기반 추정값을 실제 값처럼 표시하지 않는다.
- 추천 후보 48개는 먼저 모두 점수화한 뒤, 추천 상위 2개를 고를 때 통근수단별 현실성 필터를 적용한다.
- 도보는 희망 통근시간의 25% 또는 10분 중 큰 값을 허용오차로 사용해 과도하게 먼 후보를 상위 추천에서 제외한다.
- 자동차는 희망 통근시간의 30% 또는 15분 중 큰 값, 대중교통은 35% 또는 20분 중 큰 값을 허용오차로 사용한다.
- 현실성 필터는 추천 상위 후보 산출에만 적용하며, 비추천 후보에는 통근 조건과 인프라 점수가 낮은 후보가 포함될 수 있다.
- 자동차는 네이버 Directions 5, 대중교통은 ODsay, 도보는 TMAP 보행자 경로 API로 실제 통근시간을 조회한다.
- Naver Client Secret과 TMAP appKey는 Vercel 서버리스 함수에서만 사용해 프론트엔드에 노출하지 않는다.
- 도보는 실제 경로 결과가 60분을 넘으면 TOP 추천에서 제외하고, 희망 통근시간 초과분은 통근 중요도에 따라 패널티로 반영한다.
