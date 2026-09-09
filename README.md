# AlphaEarth 서울 침수 위험도 분석

이 저장소의 핵심 실행 스크립트는 `seoul_flood/run_analysis.py`입니다. Google Earth Engine의 AlphaEarth/Satellite Embedding과 지형, 수문, 토지피복, 배수 인프라 지표를 결합해 서울시 침수 위험도를 학습하고, 공간 기반 5-fold 검증 결과를 CSV/JSON으로 저장합니다.

## 분석 개요

`run_analysis.py`는 다음 흐름으로 동작합니다.

1. `.env` 또는 셸 환경변수에서 Earth Engine 프로젝트 ID와 분석 설정을 읽습니다.
2. 서울 행정경계를 Earth Engine `geoBoundaries`에서 가져옵니다.
3. `source/seoul_flood_reference_points.geojson`의 침수 기준점을 양성 샘플로 사용합니다.
4. 서울 전역을 공간 블록으로 나누고 5개 fold를 배정해 공간적으로 분리된 교차검증을 수행합니다.
5. 지형/수문/토지피복 feature와 배수 인프라 feature를 생성합니다.
6. AlphaEarth 64차원 embedding을 학습 fold의 침수 기준점 평균과 비교해 `alpha_score` 1개 feature로 축약합니다.
7. 양성/음성 샘플을 균형 추출하고 Earth Engine `smileGradientTreeBoost` 모델을 학습합니다.
8. fold별 성능, feature importance, 상위 위험지역 hit율, 선택적 외부 침수구역 검증 결과를 저장합니다.

## 주요 파일

```text
.
├── README.md
├── requirements.txt
├── .env.example
└── seoul_flood
    ├── run_analysis.py
    ├── source
    │   ├── seoul_flood_reference_points.geojson
    │   ├── seoul_pump_stations.csv
    │   ├── sewer_level_sensor_gu_stats.csv
    │   ├── sewer_level_202605.csv
    │   ├── official_city_flood_by_frequency  # 선택
    │   └── official_city_flood               # 선택
    └── outputs
```

## 설치

Python 가상환경을 만든 뒤 의존성을 설치합니다.

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

Windows에서는 가상환경 활성화 명령만 다음처럼 바꿉니다.

```bat
venv\Scripts\activate
```

## Earth Engine 설정

Google Earth Engine 사용 권한이 있는 Google Cloud 프로젝트가 필요합니다. 최초 1회 인증을 진행합니다.

```bash
earthengine authenticate
```

저장소 루트, `seoul_flood` 폴더, 또는 실행 위치에 `.env` 파일을 만들 수 있습니다. 최소 설정은 다음과 같습니다.

```env
EE_PROJECT_ID=your-earth-engine-project-id
```

## 실행

저장소 루트에서 다음 명령으로 실행합니다.

```bash
python seoul_flood/run_analysis.py
```

스크립트는 입력 파일을 현재 실행 위치, `seoul_flood` 폴더, `seoul_flood/source` 폴더 순서로 찾습니다. 출력 경로가 상대경로이면 `seoul_flood` 폴더 기준으로 저장됩니다.

## 환경변수

| 변수 | 기본값 | 설명 |
| --- | --- | --- |
| `EE_PROJECT_ID` | 없음 | 필수. Earth Engine 초기화에 사용할 Google Cloud 프로젝트 ID |
| `YEAR` | `2024` | AlphaEarth, Dynamic World 분석 연도 |
| `ANALYSIS_SCALE` | `30` | 샘플링과 raster 계산 스케일(m) |
| `SPATIAL_BLOCK_DEGREES` | `0.015` | 공간 fold 배정을 위한 경위도 블록 크기 |
| `SPATIAL_FOLDS` | `5` | 공간 교차검증 fold 수 |
| `POSITIVE_SAMPLE_POINTS` | `200` | fold별 양성 샘플 수 |
| `NEGATIVE_POINTS` | `200` | fold별 음성 샘플 수 |
| `POSITIVE_BUFFER_M` | `60` | 침수 기준점 주변 양성 후보 buffer(m) |
| `NEGATIVE_BUFFER_M` | `300` | 음성 후보에서 제외할 침수 기준점 주변 buffer(m) |
| `HOTSPOT_EVAL_PERCENTILES` | `80,90,95` | 위험도 상위 영역 hit율 평가 percentile |
| `OUTPUT_DIR` | `outputs/analysis` | 결과 저장 폴더 |
| `SEOUL_REFERENCE_GEOJSON` | `source/seoul_flood_reference_points.geojson` | 침수 기준점 GeoJSON |
| `PUMP_STATION_CSV` | `source/seoul_pump_stations.csv` | 배수펌프장 원자료 |
| `SEWER_SENSOR_GU_STATS_CSV` | `source/sewer_level_sensor_gu_stats.csv` | 자치구별 하수관로 수위 센서 요약 CSV |
| `SEWER_LEVEL_SENSOR_CSV` | `source/sewer_level_202605.csv` | 하수관로 수위 센서 원자료 |
| `RUN_EXTERNAL_VALIDATION` | 공식 침수구역 폴더가 있으면 `1`, 없으면 `0` | 공식 침수구역 shapefile 기반 외부 검증 실행 여부 |
| `OFFICIAL_FLOOD_SHP_FREQ_ROOT_DIR` | `source/official_city_flood_by_frequency`가 있으면 사용 | 빈도별 공식 침수구역 zip 폴더 루트 |
| `OFFICIAL_FLOOD_SHP_ZIP_DIR` | `source/official_city_flood`가 있으면 사용 | 공식 침수구역 zip 폴더 |
| `OFFICIAL_FLOOD_SHP_PROJ` | `EPSG:5186` | 공식 침수구역 shapefile 좌표계 |
| `OFFICIAL_FLOOD_SHP_ENCODING` | `cp949` | 공식 침수구역 shapefile 인코딩 |
| `OFFICIAL_FLOOD_SHP_SIMPLIFY_M` | `30` | 외부 검증 geometry 단순화 허용 오차(m) |
| `EXTERNAL_VALIDATION_PERCENTILES` | `80,90,95` | 외부 침수구역과 비교할 위험도 percentile |

예시:

```env
EE_PROJECT_ID=your-earth-engine-project-id
YEAR=2024
OUTPUT_DIR=outputs/run_analysis
POSITIVE_SAMPLE_POINTS=200
NEGATIVE_POINTS=200
RUN_EXTERNAL_VALIDATION=1
```

## 입력 데이터

기본 입력 데이터는 `seoul_flood/source` 아래에 둡니다.

- `seoul_flood_reference_points.geojson`: 침수 기준점. 각 feature의 좌표를 양성 target으로 사용합니다.
- `seoul_pump_stations.csv`: 배수펌프장 자료. 자치구별 펌프장 수, 배수량, 유역면적, 유수지용량 feature로 요약됩니다.
- `sewer_level_sensor_gu_stats.csv`: 자치구별 하수관로 수위 센서 수 요약 자료입니다.
- `sewer_level_202605.csv`: 센서 요약 파일이 없을 때 센서 원자료에서 자치구별 센서 수를 계산하는 fallback 입력입니다.
- 공식 침수구역 shapefile zip 폴더: 있으면 외부 검증에 사용됩니다.

Earth Engine에서는 다음 공개 데이터셋을 사용합니다.

- `WM/geoLab/geoBoundaries/600/ADM1`, `ADM2`
- `USGS/SRTMGL1_003`
- `MERIT/Hydro/v1_0_1`
- `JRC/GSW1_4/GlobalSurfaceWater`
- `GOOGLE/DYNAMICWORLD/V1`
- `GOOGLE/SATELLITE_EMBEDDING/V1/ANNUAL`

## 산출물

기본 산출물은 `seoul_flood/outputs/analysis`에 저장됩니다.

| 파일 | 내용 |
| --- | --- |
| `metrics.json` | 실행 설정, 입력 데이터 요약, 모델 정보, fold별 검증 결과, feature importance, top-k 요약, 외부 검증 요약 |
| `cv_results.csv` | fold별 validation 성능과 혼동행렬 |
| `feature_importance.csv` | feature별 평균 중요도와 표준편차 |
| `topk_summary.csv` | 위험도 상위 20%, 10%, 5% 영역의 침수 기준점 포함률과 lift |
| `external_validation.csv` | 공식 침수구역 shapefile이 있을 때 위험도 상위 영역과의 면적 중첩 검증 |

실행 중에는 fold별 accuracy, kappa, ROC-AUC, PR-AUC, recall, F1이 콘솔에 출력됩니다.

## 모델 feature

기본 feature는 다음과 같습니다.

- `slope`: SRTM 기반 경사도
- `hnd`: MERIT Hydro 기반 하천 대비 상대고도
- `log_upa`: 상류 유역면적 로그값
- `water_occ`: JRC Global Surface Water occurrence
- `built`: Dynamic World built 평균값
- `lowland`: 서울 내부 상대 저지대 점수
- `alpha_score`: AlphaEarth embedding과 침수 기준점 평균 embedding의 유사도
- 배수 인프라 feature: 입력 CSV가 충분하면 자치구 단위 정규화 feature로 추가됩니다.

최종 모델은 Earth Engine `smileGradientTreeBoost`이며, 확률 출력값 `flood_prob`을 기준으로 검증 지표와 hotspot 지표를 계산합니다.

## `run_analysis.py` 라인 범위별 역할

아래 라인 번호는 현재 `seoul_flood/run_analysis.py` 기준입니다.

- `1-14`: 라이브러리 import, 스크립트 기준 경로(`SCRIPT_DIR`, `REPO_DIR`) 설정
- `17-40`: `.env` 파일 로드 및 환경변수 주입
- `43-127`: 입력/출력 경로 해석, 정수 리스트 파싱, JSON/CSV 저장, 숫자 파싱 유틸리티
- `130-212`: 서울 자치구명과 geoBoundaries ADM2 이름 매핑, 자치구명 추출, CSV 인코딩 fallback, 구별 통계 정규화
- `215-368`: 배수펌프장과 하수관로 수위 센서 CSV를 자치구별 배수 인프라 feature로 요약
- `370-441`: 분석 설정값, 입력 파일 경로, 출력 파일 경로, 공식 침수구역 외부 검증 설정 정의
- `443-485`: Earth Engine 초기화, 공간 fold 배정 함수, 서울 행정경계 로드
- `488-530`: 침수 기준점 GeoJSON을 양성 target으로 읽고, 서울 전체 raster에 공간 fold 이미지 생성
- `533-669`: 자치구 단위 배수 인프라 통계를 ADM2 polygon raster feature로 변환하고 정적 feature 이미지 생성
- `672-688`: AlphaEarth/Satellite Embedding annual image 로드
- `691-749`: 음성 후보 mask 생성, 학습 fold 기준 `alpha_score` feature 계산
- `751-802`: fold별 입력 캐시, train/validation mask 구성, 최종 입력 band 목록과 Gradient Tree Boost 모델 설정
- `804-856`: 양성/음성 sample 균형 추출, 모델 학습, validation 혼동행렬 계산
- `858-956`: accuracy, precision, recall, F1, ROC-AUC, PR-AUC 등 분류 성능 지표 계산
- `958-1069`: 위험도 상위 20%, 10%, 5% 영역의 침수 기준점 포함률과 lift 계산
- `1072-1163`: feature importance 추출, fold 하나에 대한 학습/검증/중요도/위험지역 평가 실행
- `1165-1275`: 5-fold 결과 평균/표준편차 요약, feature importance 요약, top-k hotspot 요약
- `1277-1298`: 저장용 fold 결과에서 classifier/image 객체를 제외하고 CSV/JSON에 필요한 값만 정리
- `1300-1427`: 공식 침수구역 shapefile geometry 단순화 및 Earth Engine feature 변환
- `1430-1598`: 공식 침수구역 zip 폴더 탐색, 빈도 정보 추론, shapefile zip 로드
- `1601-1647`: 외부 검증용 feature collection 정규화와 면적 계산 유틸리티
- `1650-1821`: 공식 침수구역과 예측 위험도 상위 영역의 면적 중첩, recall, precision, lift 계산
- `1823-1853`: 외부 검증 실행 여부 판단 및 공식 침수구역 reference별 검증 실행
- `1856-1941`: 전체 실행 진입점, 5-fold 반복 실행, `metrics.json`/`cv_results.csv`/`feature_importance.csv`/`topk_summary.csv`/`external_validation.csv` 저장
