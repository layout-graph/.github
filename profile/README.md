# ARCHITECTURE

"지능형 학술 문서 생성 시스템: 데이터 파이프라인 아키텍처"

전체 데이터 흐름은 기획(LLM) ➡️ 설계(GNN) ➡️ 시공(LLM/CLIP) ➡️ 렌더링의 4단계로 구성됩니다.

### Step 1. 기획: 시맨틱 그래프 생성 (Semantic Graph Generation)

사용자의 파편화된 원시 데이터를 분석하여, 학술 문서에 필요한 논리적 구성 요소와 순서를 기획합니다.

- **담당 모듈:** Google Gemini API (`back/app/core/llm.py`)
- **입력:** 사용자의 문서 주제 (`topic`)
- **동작:** 모델이 학습한 9가지 카테고리(`Title`, `Section-header`, `Text`, `List-item`, `Picture`, `Table`, `Formula`, `Caption`, `Footnote`) 내에서 페이지 단위로 필요한 노드들을 생성합니다.
- **JSON 상태:** GNN이 필요로 하는 **8가지 핵심 구조적 속성**(`importance`, `text_length`, `aspect_ratio`, `reading_order`, `has_paragraph`, `tree_depth`, `children_count`)과 함께 **초기 추측(rough estimation) 좌표인 `box`**를 포함하여 반환합니다.

```json
{
  "page1": {
    "nodes": [
      {
        "category": "Title",
        "importance": 0.1,
        "text_length": 0.2,
        "aspect_ratio": 5.0,
        "reading_order": 0,
        "has_paragraph": 1,
        "tree_depth": 0,
        "children_count": 0,
        "box": [0.1, 0.05, 0.8, 0.1]
      }
    ]
  }
}
```

### Step 2. 설계: 레이아웃 좌표/크기 계산 (Spatial Arrangement)

LLM이 기획한 초기 JSON 구조를 바탕으로, GNN이 문서의 시맨틱(가시성, 계층)을 분석하여 최적의 정밀 좌표를 예측합니다.

- **담당 모듈:** Graph Builder & LayoutGNN (`back/app/core/predictor.py`)
- **입력:** Step 1의 LLM JSON 데이터
- **동작 상세:**
  - **데이터 전처리 (`graph_builder.py`):** LLM JSON의 8개 속성 + 카테고리(One-Hot 9차원) + 초기 좌표(4차원)를 병합하여 **20차원 피처**를 추출합니다. 노드 간 가시성(Visibility) 및 계층(Hierarchy) 관계를 분석하여 **15차원 엣지 속성**(거리, 겹침 등)과 인접 행렬(COO)을 생성합니다.
  - **GNN 연산 (`layer.py`):** `20 -> 256 -> 128 -> 64 -> 4` 차원으로 수렴하는 4계층 `DualGNNLayer`를 거칩니다. **Mean Aggregation**과 **LayerNorm**을 통해 안정적으로 메시지를 취합하며, 초기 `box` 좌표로부터의 변위(`delta`)를 계산해 최종 좌표를 도출합니다.
  - **좌표 보정 (`predictor.py`):** GNN이 반환한 `initial_coords + delta` 값을 0.0 ~ 1.0 사이로 클리핑(`max(0.0, min(1.0, v))`)하여 안전한 범위를 보장합니다.
- **결과 상태:** 빈 공간들의 정교한 위치와 크기가 담긴 **정밀한 4차원 `[x, y, w, h]` 정규화 좌표**가 반환됩니다.

```json
{
  "id_box": 0,
  "predicted_box": [0.119, 0.224, 0.555, 0.168]
}
```

### Step 3. 렌더 준비: 픽셀 좌표 역정규화 (API Endpoint)

FastAPI 엔드포인트에서 예측된 정규화 좌표를 클라이언트의 캔버스 픽셀(Pixel) 크기에 맞춰 변환하고 최종 응답을 반환합니다.

- **담당 모듈:** FastAPI (`back/app/api/endpoints/documents.py`)
- **입력:** 클라이언트의 `LayoutRequest` 내 캔버스 규격(`canvas_width`, `canvas_height`)
- **동작:**
  - GNN이 뱉은 `predicted_box` 정규화 좌표에 캔버스의 너비/높이를 곱해 `int`형 픽셀 좌표(`x, y, w, h`)로 역정규화합니다.
  - LLM 원본 JSON에 있던 `category` 정보를 병합하여 픽셀 좌표와 맵핑합니다.
- **출력 (JSON 상태):** 프론트엔드가 즉시 컴포넌트를 브라우저에 배치할 수 있는 **픽셀 기반 LayoutResponse**가 반환됩니다.

```json
{
  "pages": [
    {
      "page": 1,
      "nodes": [
        {
          "category": "Title",
          "x": 95,
          "y": 40,
          "w": 640,
          "h": 80
        }
      ]
    }
  ]
}
```

### Step 4. 시공 및 콘텐츠 매핑 (Content Optimization)

`/api/documents/{doc_id}/optimize` 등의 향후 파이프라인을 통해, 확정된 픽셀 도면(공간)에 실제 텍스트 및 시각적 콘텐츠를 다듬어 끼워 넣습니다.

- **담당 모듈:** LLM (텍스트 최적화) & CLIP (이미지 매칭)
- **동작:**
  - **[LLM 최적화]:** 픽셀로 할당된 `box`의 너비, 높이, 비율을 인지하여 오버플로우가 발생하지 않도록 원본 텍스트 분량을 축소/요약하거나 확장합니다.
  - **[CLIP 매칭]:** `Picture` 등의 노드 주변 텍스트(문맥)를 분석하고, 사용자가 제공한 원시 이미지 풀에서 가장 관련성이 높은 시각 자료를 선별해 맵핑합니다.
- **결과:** 좌표와 텍스트, 이미지 경로(`content`)가 모두 채워진 **최종 통합 데이터 소스(Single Source of Truth)**가 프론트엔드에 전달되어 PDF/HTML 등으로 출력됩니다.

---ent)

기획된 뼈대를 바탕으로, 문서의 카테고리 시맨틱을 분석하여 최적의 물리적 공간 배치를 수행합니다.

- **담당 모듈:** LayoutGNN (Graph Neural Network)
- **입력:** Step 1의 뼈대 JSON 구조 (Graph Builder를 통해 변환)
- **동작 상세:**
  - **데이터 전처리:** `graph_builder.py`를 거쳐 각 노드의 속성을 **20차원 피처**(원핫 인코딩, 영역 중요도, 트리 깊이 등)로 추출하고, 가시성 및 계층 관계를 바탕으로 **15차원 엣지 속성**(거리, 겹침 등)과 인접 행렬(COO)을 생성합니다.
  - **GNN 연산:** `20 -> 256 -> 128 -> 64 -> 4` 차원으로 수렴하는 4계층 `DualGNNLayer`를 거칩니다. 이 과정에서 **Mean Aggregation**을 사용해 메시지 폭발을 막고, **LayerNorm**을 통해 깊은 레이어에서도 안정성을 유지합니다.
  - **학습 특징:** 모델은 항등 함수를 피하기 위해 원본 좌표에 가우시안 노이즈(Data Augmentation)를 주입받아 스스로 교정하는 법을 학습했습니다. 그 결과, 단순 복사가 아니라 `Section-header -> Text -> List-item`으로 이어지는 **다중 계층 들여쓰기(Multi-level Indentation)**와 캡션의 정교한 동기화를 스스로 연산합니다.
- **JSON 상태:** 빈 공간들의 최종 위치와 크기가 담긴 **정밀한 4차원 `[x, y, w, h]` 도면**이 완성됩니다.

```json
{
  "id": "n2",
  "category": "Text",
  "box": [119, 224, 555, 168]
}
```
