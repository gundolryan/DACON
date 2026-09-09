# 블랙박스 영상 기반 지능형 고의사고 분석 AI 경진대회

블랙박스 영상만으로 재녹화 여부, 교통사고 주요 시점과 상황, 차량의 가감속·조향 상태를 분석하는 DACON 대회의 공식 베이스라인 자료를 정리한 저장소입니다.

> 주최: 행정안전부, 한국지능정보사회진흥원<br>
> 주관: 국립과학수사연구원<br>
> 운영: 데이콘

## 대회 배경

블랙박스 영상은 사고 과정과 차량 움직임을 확인하는 핵심 자료지만, 재녹화 여부 판별, 차량 진입·충돌 시점 탐색, 사고 전후의 가감속·조향 분석은 감정관의 반복적인 영상 확인과 수작업에 크게 의존합니다.

이 대회는 다양한 촬영 환경과 사고 유형에 대응하면서 감정 판단에 참고할 정보를 자동으로 산출하는 영상 분석 모델 개발을 목표로 합니다. 단순 사고 유무 판별을 넘어 재녹화 여부, 피의차량과 피해차량의 주요 시점·상황, 사고 구간의 차량 거동을 종합적으로 분석합니다.

## 과제 구성

| Stage | 분석 과제 | 출력 |
|---|---|---|
| Stage 1 | 입력 영상의 재녹화 여부 판별 | `ORIGINAL` / `RERECORDED` |
| Stage 2 | 사고 주요 시점·상황 분석 | 충돌 프레임, 진입 프레임, 진입 방향, 회피 공간 여부 |
| Stage 3 | 0.1초 단위 차량 거동 분석 | 가감속 및 조향 범주 |

### Stage 1 — 재녹화 여부 판별

입력 영상을 파일 단위로 분석해 사고 원본인지, 다른 화면이나 기기를 통해 다시 촬영한 영상인지 분류합니다. 화면 테두리, 주사 패턴, 반사광, 밝기·색상·명암 변화, 압축 열화, 촬영 기기의 움직임, 화면 비율과 같은 공간적·시간적 특징을 종합적으로 활용할 수 있습니다.

- 출력 컬럼: `ID`, `answer`
- 평가: 두 클래스의 Macro-F1

$$
Score_{stage1}=\frac{F1_{ORIGINAL}+F1_{RERECORDED}}{2}
$$

### Stage 2 — 사고 주요 시점·상황 분석

사고 원본 영상에서 추출한 프레임을 이용해 다음 항목을 예측합니다.

- `collision_frame`: 피의차량과 피해차량이 실제로 접촉한 원본 프레임 번호
- `entry_frame`: 피해차량의 바퀴가 피의차량 차선에 처음 닿은 원본 프레임 번호
- `evasion_space`: 충돌 당시 피의차량이 진행하거나 회피할 공간이 있었는지 여부
- `entry_side`: 블랙박스 화면 기준 피해차량 진입 방향(`LEFT` / `RIGHT`)

시점 예측은 파일명에 포함된 원본 프레임 번호를 제출해야 합니다. 프레임 번호를 영상별 시간으로 변환한 뒤 정답과 비교하며, 오차가 ±0.3초 이내이면 정답으로 처리됩니다.

$$
Score_{stage2}=0.35S_{collision}+0.35S_{entry}+0.15S_{direction}+0.15S_{evasion}
$$

### Stage 3 — 차량 거동 특성 분석

10Hz 실차 주행 영상을 입력으로 받아 모든 `sample_index`에 대해 다음 범주를 예측합니다.

- 가감속: `ACCELERATING`, `DECELERATING`, `CONSTANT`, `STOPPED`
- 조향: `LEFT`, `STRAIGHT`, `RIGHT`
- 출력 컬럼: `ID`, `sample_index`, `accel_label`, `steer_label`

정지 상태로 판정된 프레임도 `steer_label`을 출력해야 하지만 조향 Macro-F1 계산에서는 제외됩니다.

$$
Score_{stage3}=0.7F1_{accel}+0.3F1_{steer}
$$

## 종합 평가

1차 평가는 한 제출물에서 산출된 세 Stage 점수를 다음 가중치로 합산합니다. 각 Stage의 개별 최고점을 조합하지 않습니다.

$$
Total=0.2Score_{stage1}+0.4Score_{stage2}+0.4Score_{stage3}
$$

Private 리더보드 상위 15팀이 2차 평가 대상으로 선정되며, 모델 개발 보고서와 학습데이터 구성 보고서 등을 종합 평가해 최종 상위 7팀을 선정합니다.

## 베이스라인

제공된 노트북은 세 Stage의 학습, 특징 추출, 추론 및 제출 ZIP 생성을 하나의 흐름으로 보여 줍니다.

| Stage | 베이스라인 구성 | 개요 |
|---|---|---|
| Stage 1 | MViTv2-S | 여러 구간에서 16프레임 클립을 추출하고 로짓을 평균해 재녹화 여부 분류 |
| Stage 2 | ResNet18 + BiGRU | 프레임별 공간 특징과 시간 흐름을 결합해 두 시점과 두 상황 분류값 예측 |
| Stage 3 | MViTv2-S 다중 헤드 | 10Hz 영상에서 가감속 4종과 조향 3종을 시점별 예측 |

공개 예제는 Stage별 5건으로 구성된 참고용 소규모 데이터입니다. 실제 성능을 위한 충분한 학습 데이터가 아니며, 다양한 촬영 환경과 사고 유형에 일반화하도록 별도 데이터 구성과 검증이 필요합니다.

## 저장소 구성

```text
.
├── notebooks/
│   ├── [Baseline]_MViT·ResNet18·GRU를 활용한 3-Stage 모델 학습 및 피쳐 추출(학습).ipynb
│   └── [Baseline]_MViTv2-S·ResNet18·GRU를 활용한 3-Stage 모델 추론(추론).ipynb
├── Baseline.zip
├── .gitattributes
└── README.md
```

`Baseline.zip`에는 두 베이스라인 노트북, `requirements.txt`, Stage별 공개 예제 데이터와 라벨이 포함되어 있습니다. ZIP은 GitHub의 일반 파일 크기 제한을 넘으므로 Git LFS로 관리합니다.

## 실행 순서

1. Git LFS가 설치된 환경에서 저장소를 클론합니다.
2. `Baseline.zip`을 압축 해제합니다.
3. 압축을 푼 디렉터리에서 학습 노트북을 실행해 `model/stage1`, `model/stage2`, `model/stage3` 체크포인트를 생성합니다.
4. 추론 노트북을 실행해 `inference.py`를 생성하고 공개 예제로 반환 형식을 점검합니다.
5. 추론 노트북의 마지막 단계에서 평가 서버용 `submit.zip`을 생성하고 구조를 검사합니다.

```bash
git lfs install
git clone https://github.com/gundolryan/DACON.git
cd DACON
unzip Baseline.zip -d baseline
cd baseline
python -m pip install -r requirements.txt
jupyter notebook
```

> 베이스라인 학습 노트북은 torchvision 사전학습 가중치를 사용할 수 있습니다. 반면 평가 서버의 추론 단계는 인터넷이 차단되므로, 추론 코드는 `weights=None`으로 구조만 만든 뒤 제출 ZIP에 포함한 체크포인트를 `load_state_dict`로 불러와야 합니다.

## 코드 제출 형식

평가 서버에 업로드하는 `submit.zip`의 최상위 구조는 반드시 아래와 같아야 합니다.

```text
submit.zip
├── model/
│   ├── stage1/
│   ├── stage2/
│   └── stage3/
├── inference.py
└── requirements.txt
```

`inference.py`에는 아래 세 함수가 모두 구현되어야 하며 각각 지정된 컬럼의 `pandas.DataFrame`을 반환해야 합니다.

```python
def predict_stage1(data_dir, model_dir):
    ...  # ID, answer

def predict_stage2(data_dir, model_dir):
    ...  # ID, collision_frame, entry_frame, evasion_space, entry_side

def predict_stage3(data_dir, model_dir):
    ...  # ID, sample_index, accel_label, steer_label
```

평가 서버가 읽기 전용 `data/`, 채점용 `script.py`, 결과 저장용 `output/`을 추가합니다. 실행 결과는 반드시 `output/submission.csv`로 저장되어야 합니다.

## 평가 환경과 제한

- GPU: NVIDIA L40S, VRAM 44.7 GiB
- CPU / RAM: 7 vCPU / 60 GB
- 공유 메모리: 30 GB
- 패키지 설치: 최대 10분
- 전체 추론: 최대 60분
- 제출 ZIP: 최대 10 GB
- 압축 해제 후: 최대 32 GB
- 패키지 설치 이후 인터넷 연결 불가
- 비공개 평가 데이터로 추가 학습, 튜닝, Pseudo-Labeling 금지
- 평가 파일 간 정보·예측값·통계를 공유하지 않고 각 파일을 독립적으로 예측

평가 서버에는 PyTorch 2.8.0+cu128, torchvision 0.23.0+cu128, pandas 2.2.2, NumPy 1.26.4, OpenCV headless 4.10.0.84, timm 1.0.15 등 주요 패키지가 기본 설치되어 있습니다. 설치 시간과 충돌을 줄이기 위해 기본 패키지는 `requirements.txt`에 중복 기재하지 않는 편이 권장됩니다.

## 주요 일정

| 구분 | 일정 |
|---|---|
| 참가 기간 | 2026-08-18 10:00 ~ 2026-09-29 10:00 |
| 대회 기간 | 2026-08-26 10:00 ~ 2026-09-30 10:00 |
| 팀 병합 마감 | 2026-09-23 23:59 |
| 리더보드 제출 마감 | 2026-09-29 10:00 |
| 2차 평가 자료 제출 | 2026-09-30 12:00 ~ 2026-10-05 10:00 |
| 2차 평가 및 검증 | 2026-10-05 12:00 ~ 2026-10-15 10:00 |
| 최종 결과 발표 | 2026-10-16 10:00 |
| 오프라인 시상식 | 2026-11-27 예정 |

## 제출 전 체크리스트

- 세 예측 함수가 모두 구현되어 있고 정확한 컬럼의 DataFrame을 반환하는가?
- Stage 2가 이미지 순번이 아닌 파일명의 원본 프레임 번호를 반환하는가?
- Stage 3가 모든 `sample_index`의 가감속·조향 값을 반환하는가?
- 모델 가중치와 설정 파일이 제출 ZIP에 포함되어 있는가?
- 외부 다운로드나 API 호출 없이 완전히 오프라인으로 실행되는가?
- ZIP 구조, 용량, 설치 시간과 전체 추론 시간 제한을 만족하는가?
- 평가 파일별 독립 예측 원칙과 비공개 데이터 추가 학습 금지 규정을 준수하는가?

## 유의사항

본 저장소의 코드는 대회 이해와 구현을 위한 베이스라인입니다. 참가자는 사용하는 데이터, 사전학습 모델, API 및 기타 외부 자원의 라이선스와 이용 조건을 직접 확인하고, 2차 평가 자료에 모든 출처를 명시해야 합니다. 세부 규정과 변경 사항은 반드시 DACON 대회 페이지의 최신 공지를 기준으로 확인하세요.
