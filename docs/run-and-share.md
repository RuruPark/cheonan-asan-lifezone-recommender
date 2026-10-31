# UI 실행 및 공유 방법

## 로컬 실행

이 프로젝트는 Next.js가 아니라 정적 HTML/CSS/JS 웹앱입니다. `src/main.js`가 `src/components/app.js`를 불러와 UI를 렌더링하고, `scripts/dev-server.mjs`가 프로젝트 루트를 정적 파일로 서빙합니다.

```powershell
cd cheonan-asan-lifezone-recommender
node scripts\dev-server.mjs
```

기본 주소는 다음과 같습니다.

```text
http://localhost:4173
```

포트를 바꾸려면 `PORT` 환경변수를 지정합니다.

```powershell
$env:PORT=8010
node scripts\dev-server.mjs
```

`package.json` 기준 실행 명령은 `npm run dev`입니다. 로컬에서 `npm`이 PATH에 잡혀 있다면 아래 명령도 같은 개발 서버를 실행합니다.

```powershell
npm run dev
```

기존 보조 실행 스크립트도 유지되어 있습니다. 이 스크립트들은 Anaconda의 `alldam` 환경에서 Python 정적 서버를 실행하며, 기본 포트는 `8000`입니다.

```powershell
.\scripts\start-ui.bat
powershell -ExecutionPolicy Bypass -File .\scripts\start-ui.ps1
```

## 테스트 실행

`package.json` 기준 테스트 명령은 다음과 같습니다.

```powershell
npm test
```

`npm`을 사용할 수 없는 환경에서는 Node 테스트 러너를 직접 실행할 수 있습니다.

```powershell
node --test --test-isolation=none tests/*.test.mjs
```

정적 앱 구성 파일 검증은 다음 명령으로 실행합니다.

```powershell
node scripts\verify-static-app.mjs
```

## 환경변수와 키 관리

`.env.local`은 실제 API 키를 넣는 로컬 전용 파일이며 Git에 올리지 않습니다. 현재 `.gitignore`에서 `.env.local`, `.env*.local`, `/data/`, API key 텍스트 파일을 제외합니다.

`.env.example`은 팀원 공유용 예시 파일입니다. 실제 키를 넣지 말고 필요한 변수 이름만 비워 둡니다.

이 프로젝트는 Next.js가 아니므로 `NEXT_PUBLIC_*` 환경변수가 자동으로 브라우저 코드에 주입되지 않습니다. 현재 `scripts/dev-server.mjs`는 `PORT`만 읽고, 네이버 지도 API 키나 Directions API 호출은 아직 처리하지 않습니다.

Client Secret은 프론트엔드 코드에 포함하면 안 됩니다. 이후 네이버 Directions 5 API처럼 secret이 필요한 호출은 별도 서버 또는 안전한 백엔드 경유 방식으로 붙입니다.

## 지도 API 동작 방식

네이버 지도 Client ID가 없으면 기존 CSS/SVG 기반 fallback 지도 UI가 표시됩니다. 이 상태에서도 직장 마커, 생활권 마커, 직장과 생활권을 잇는 보조 연결선은 계속 표시됩니다.

정적 배포 환경에서 실제 네이버 지도를 사용하려면 브라우저에서 `window.__APP_CONFIG__.NAVER_MAP_CLIENT_ID` 또는 `window.__APP_CONFIG__.NEXT_PUBLIC_NAVER_MAP_CLIENT_ID` 값을 제공해야 합니다. `.env.local`의 값은 Next.js처럼 자동 주입되지 않습니다.

GitHub Pages에서 실제 네이버 지도를 보려면 프로젝트 루트의 `public-config.js`에 발급받은 Client ID를 직접 입력합니다.

```js
window.__APP_CONFIG__ = {
  NAVER_MAP_CLIENT_ID: "여기에_실제_Client_ID_입력"
};
```

`public-config.js`에는 Client Secret을 절대 넣지 않습니다. 현재 파일에는 placeholder만 들어 있으며, placeholder 상태이거나 Client ID가 비어 있으면 기존 fallback 지도로 동작합니다.

네이버 Maps Application의 Web 서비스 URL에는 호스트 도메인인 `http://rurupark.github.io`를 등록합니다. GitHub Pages 실제 접속 주소는 보통 `https://rurupark.github.io/alldam-project/` 형식이지만, 네이버 Application에는 경로가 포함된 주소가 아니라 호스트 도메인을 등록합니다. Dynamic Map은 네이버 Maps Application에서 선택되어 있어야 합니다.

GitHub Pages 같은 정적 배포에서는 Client Secret을 사용할 수 없습니다. `NAVER_MAP_CLIENT_SECRET`은 프론트엔드 코드에 노출하지 않고, Vercel 서버리스 함수(`api/commute/`)에서만 사용합니다. 그래서 자동차·도보 실제 경로 조회는 Vercel 배포 환경에서만 동작합니다.

지도 위 연결선은 실제 길찾기 경로가 아니라 직장 좌표와 생활권 좌표를 잇는 보조선입니다.

현재 읍면동 경계는 국토교통부 센서스경계 행정동경계 SHP를 WGS84(EPSG:4326) GeoJSON으로 변환한 뒤 천안·아산 지역만 추출한 실제 행정동경계 데이터입니다. 네이버 지도 Client ID가 있으면 실제 네이버 지도 위에 행정동 경계를 함께 표시할 수 있고, Client ID가 없으면 fallback 지도로 계속 동작합니다.

실제 VWorld GeoJSON을 확보하면 `scripts/prepare-vworld-boundaries.mjs`로 천안·아산 읍면동만 추출해 `src/data/cheonanAsanEmdBoundaries.js`를 실제 경계 데이터로 교체할 수 있습니다.

```powershell
node scripts\prepare-vworld-boundaries.mjs --input .\data\map\vworld_emd.geojson --output .\src\data\cheonanAsanEmdBoundaries.js
```

SHP 원본은 이 스크립트에서 직접 처리하지 않습니다. QGIS, mapshaper, ogr2ogr 등으로 GeoJSON FeatureCollection으로 먼저 변환한 뒤 입력합니다. 변환 결과가 실제 경계 데이터이면 `metadata.isSample`은 `false`가 됩니다. 통근시간은 아직 네이버 Directions 5 결과가 아니라 거리 기반 fallback 추정값입니다.

국토교통부 센서스경계 행정동경계 원본은 `data/행정동경계`에 보관합니다. 현재 확인된 원본은 `BND_ADM_DONG_PG.shp`, `.dbf`, `.shx`, `.prj`, `.cpg`로 구성된 SHP 묶음이며 `.prj` 기준 KGD2002 Central Belt 2010 투영좌표계입니다. 변환 결과는 `data/행정동경계/converted/admin_emd_4326.geojson`에 저장했습니다.

WGS84 GeoJSON으로 변환한 뒤에는 범용 스크립트로 실제 행정동경계를 적용합니다.

```powershell
node scripts\prepare-admin-boundaries.mjs --input .\data\행정동경계\converted\admin_emd_4326.geojson --output .\src\data\cheonanAsanEmdBoundaries.js --source molit-census-boundary
```

변환 성공 후 `src/data/cheonanAsanEmdBoundaries.js`의 `metadata.source`는 `molit-census-boundary`, `metadata.isSample`은 `false`입니다. 좌표계가 확실하지 않거나 천안·아산 feature를 찾지 못하면 스크립트는 기존 `src/data/cheonanAsanEmdBoundaries.js`를 덮어쓰지 않습니다.

## 데이터 위치

루트의 `data/` 폴더는 Git에서 제외되어 원본/전처리 작업 파일 보관용으로 사용합니다.

웹앱에서 직접 import해서 사용할 공개 데이터는 `src/data/` 아래에 둡니다. 예를 들어 읍면동 중심좌표 데이터는 `src/data/cheonanAsanEmdCenters.js`에서 관리합니다.

VWorld 원본 GeoJSON이나 SHP 묶음은 `data/` 또는 `data/map/`에 보관해도 됩니다. 다만 브라우저 앱에서 import하는 최종 JS 데이터는 항상 `src/data/`에 있어야 합니다.

## 공유 주의

`http://localhost:4173` 또는 `http://localhost:8000` 같은 주소는 현재 컴퓨터 안에서만 열리는 로컬 주소입니다. 다른 사람에게 공유하려면 GitHub, 압축 파일, 사내 공유 폴더, GitHub Pages, Netlify, Vercel 같은 정적 호스팅 방식을 사용해야 합니다.
