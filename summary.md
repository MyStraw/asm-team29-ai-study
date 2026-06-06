# Agent1 최종 입출력 요약

Agent1 관련 입출력 스키마는 주로 아래 파일에 정의되어 있습니다.

- 공통 State / Agent1 출력 스키마: `agents/schemas.py`
- Agent1 처리 로직: `agents/agent1/service.py`
- API 요청/응답: `main.py`

## 1. Agent1 입력

### 최초 이미지 분석 요청

프론트엔드 또는 API에서 `/recommend`로 아래 값을 보낼 수 있습니다.

관련 코드:

- `main.py`의 `RecommendRequest`
- `agents/schemas.py`의 `AgentState`

```json
{
  "image_path": "tests/fixtures/agent1/images/jeyuk.png",
  "image_id": "optional-image-id",
  "annotation_output_path": "runtime_outputs/agent1/result.png",
  "user_input_ingredients": [],
  "confidence_threshold": 0.4,
  "user_mood_input": "피곤해",
  "user_situation_input": "빠른 저녁",
  "servings": 2
}
```

필드 설명:

- `image_path`: Agent1이 분석할 이미지 파일 경로
- `image_id`: 이미지 식별자
- `annotation_output_path`: 바운딩박스 결과 이미지를 저장할 경로. 없으면 기본값 사용
- `user_input_ingredients`: 이미지 외에 사용자가 직접 입력한 재료 목록
- `confidence_threshold`: 이 값보다 detection confidence가 낮으면 사용자 확인 필요
- `user_mood_input`, `user_situation_input`, `servings`: 이후 Agent2/Agent3/레시피 흐름에서 사용하는 사용자 상황 정보

## 2. Agent1 출력

Agent1은 아래 형태로 결과를 반환합니다.

관련 코드:

- `agents/schemas.py`의 `IngredientAnalyzerOutput`
- `agents/agent1/service.py`의 `analyze_ingredients`

```json
{
  "detected_ingredients": [],
  "uncertain_ingredients": [],
  "available_ingredients": [],
  "ingredient_info": {},
  "confirmation_options": [],
  "annotated_image_path": "",
  "annotated_image_url": "",
  "vision_status": "",
  "vision_message": "",
  "raw_vision_result": {}
}
```

주요 필드 설명:

- `detected_ingredients`
  - 이미지/Solar/사용자 입력으로 확인된 재료 상세 목록
  - 각 재료는 `name`, `category`, `nutrition_type`, `boundary_box`, `confidence`, `needs_confirmation`, `source`를 가짐
- `available_ingredients`
  - 최종적으로 사용할 수 있다고 판단된 재료명 목록
- `ingredient_info`
  - Agent3가 사용하기 가장 중요한 구조화된 재료 정보
  - Agent3 입력의 `ingredients` 구조와 필드명이 맞춰져 있음

```json
{
  "main_ingredients": ["돼지고기"],
  "sub_ingredients": ["양파", "대파"],
  "seasonings": ["고추장"],
  "carbohydrates": [],
  "proteins": ["돼지고기"],
  "fats": [],
  "vegetables": ["양파", "대파"]
}
```

- `confirmation_options`
  - 사용자 확인이 필요한 재료 목록
  - 프론트엔드는 이 값을 보고 후보 선택 UI나 직접 입력 UI를 만들면 됨
- `annotated_image_path`
  - 서버 내부에 저장된 바운딩박스 결과 이미지 경로
- `annotated_image_url`
  - 프론트엔드에서 `<img>`로 바로 보여줄 수 있는 이미지 URL
  - 예: `/outputs/agent1/jeyuk_agent1_annotated.png`
- `raw_vision_result`
  - detector/Solar 중간 결과
  - detector 원본 라벨, 표준화 라벨, bbox, confidence, Solar 응답 등을 확인할 수 있음

## 3. Agent3가 보면 되는 핵심 출력

Agent3 입장에서 가장 중요한 값은 `ingredient_info`입니다.

Agent3 입력 스키마:

- `agents/agent3/schema.py`
- `Agent3Request.ingredients`

Agent1 출력:

- `agents/schemas.py`
- `IngredientAnalyzerOutput.ingredient_info`

두 구조는 아래 필드가 일치합니다.

```json
{
  "main_ingredients": [],
  "sub_ingredients": [],
  "seasonings": [],
  "carbohydrates": [],
  "proteins": [],
  "fats": [],
  "vegetables": []
}
```

즉 graph에서 연결할 때는 개념적으로 아래처럼 사용하면 됩니다.

```python
Agent3Request(
    ingredients=state["ingredient_info"],
    food_directions=state["food_directions"]
)
```

## 4. 프론트엔드가 보면 되는 핵심 출력

프론트엔드는 주로 아래 4개를 사용하면 됩니다.

### 1. `confirmation_options`

사용자 확인이 필요한 재료와 후보 목록입니다.

예시:

```json
[
  {
    "name": "채소",
    "boundary_box": [1037, 414, 1122, 838],
    "confidence": 0.59,
    "reason": "detector가 너무 일반적인 라벨로 판단했습니다.",
    "candidates": ["채소", "청경채", "대파", "양파", "고추"],
    "allow_manual_input": true
  }
]
```

프론트에서는 이걸 보고 사용자가 후보를 선택하거나 직접 입력하게 만들 수 있습니다.

### 2. `annotated_image_url`

바운딩박스와 재료명이 표시된 결과 이미지입니다.

```html
<img src="/outputs/agent1/xxx_agent1_annotated.png" />
```

### 3. `raw_vision_result`

detector/Solar 중간 결과입니다.

예시로 detector가 실제로 어떤 라벨을 냈는지 볼 수 있습니다.

```json
{
  "detections": [
    {
      "label": "돼지고기",
      "original_label": "pork",
      "boundary_box": [147, 406, 570, 928],
      "confidence": 0.48
    }
  ],
  "llm_result": {}
}
```

### 4. `ingredient_confirmation`

사용자가 확인/수정한 값을 다시 보낼 때 사용하는 입력입니다.

## 5. 사용자 확인 입력

Agent1이 아래 상태를 반환하면:

```json
{
  "vision_status": "need_user_confirmation"
}
```

graph는 다음 agent로 진행하지 않고 멈춥니다.

이때 프론트엔드는 사용자의 수정 결과를 아래 형태로 다시 보내면 됩니다.

```json
{
  "detected_ingredients": "<이전 Agent1 응답의 detected_ingredients 그대로>",
  "ingredient_confirmation": {
    "accepted_ingredients": [],
    "rejected_ingredients": ["채소"],
    "replacements": {
      "대파": "쪽파"
    },
    "additional_ingredients": [],
    "additional_ingredients_text": "간마늘, 설탕"
  },
  "user_mood_input": "피곤해",
  "user_situation_input": "빠른 저녁",
  "servings": 2
}
```

필드 설명:

- `accepted_ingredients`: 사용자가 그대로 인정한 재료를 명시적으로 추가할 때 사용
- `rejected_ingredients`: 잘못 인식되어 제거할 재료
- `replacements`: 잘못 인식된 재료를 다른 재료로 수정
- `additional_ingredients`: 누락된 재료를 배열로 추가
- `additional_ingredients_text`: 누락된 재료를 문자열로 추가. 예: `"간마늘, 설탕"`

확정 후 Agent1은 다시 Solar로 표준화하고 최종 재료 목록을 반환합니다.

예시 결과:

```json
{
  "vision_status": "success",
  "available_ingredients": ["돼지고기", "쪽파", "다진마늘", "설탕"],
  "uncertain_ingredients": [],
  "ingredient_info": {
    "main_ingredients": ["돼지고기"],
    "sub_ingredients": ["쪽파", "다진마늘"],
    "seasonings": ["설탕"],
    "proteins": ["돼지고기"],
    "vegetables": ["쪽파", "다진마늘"]
  }
}
```

## 6. 전체 흐름 요약

```text
이미지 입력
-> Agent1 detection
-> Solar 재료명 표준화/분류
-> confidence 낮거나 애매한 재료가 있으면 need_user_confirmation
-> 프론트에서 annotated_image_url + confirmation_options 표시
-> 사용자가 삭제/수정/추가 입력
-> ingredient_confirmation으로 재요청
-> Agent1이 최종 ingredient_info 생성
-> Agent2/Agent3로 진행
```
