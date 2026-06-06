# Agent1 PR Q&A

Agent1 관련 PR 4개에서 작성한 PR 퀴즈와 답변 정리입니다.

## #11 / PR #13

**PR 제목:** `feat(#11): agent1 재료 파악 파이프라인 구현`

### Q1

`user_input_ingredients`에 `["달걀", "계란", "쌀밥"]`이 들어오면 `available_ingredients`에는 어떤 값이 저장되나요?

### A1

```python
["계란", "밥"]
```

`달걀`과 `계란`은 표준명 `계란`으로 합쳐지고, `쌀밥`은 `밥`으로 표준화됩니다.

### Q2

기존 `graph.py`의 `analyze_ingredients` stub을 제거하고 어디로 분리시켰을까요?

### A2

```text
agents/agent1/service.py
```

`analyze_ingredients`를 Agent1 전용 서비스 파일로 분리하고, `agents/graph.py`에서는 해당 함수를 import해서 사용합니다.

## #15 / PR #17

**PR 제목:** `feat(#15): agent1 Solar모델 기반 재료명 표준화(한글)`

### Q1

detector가 `egg`, `onion` 같은 영어 label을 반환했을 때 Agent1은 최종적으로 어떤 한국어 재료명과 `ingredient_info` 구조를 만들어야 할까요?

### A1

`egg`는 `계란`, `onion`은 `양파`로 변환합니다.

예상 구조:

```python
available_ingredients = ["계란", "양파"]

ingredient_info = {
    "main_ingredients": ["계란"],
    "sub_ingredients": ["양파"],
    "seasonings": [],
    "proteins": ["계란"],
    "vegetables": ["양파"],
}
```

### Q2

Solar 응답에서 `category = protein`처럼 허용되지 않은 값이 들어왔을 때, 현재 구현은 이를 어떻게 보정하나요?

### A2

`category`의 허용값은 `main`, `sub`, `seasoning`이므로 `protein`은 그대로 사용하지 않습니다.

`nutrition_type`이 `protein`이면 `category`를 `main`으로 보정합니다.

```text
category = protein -> category = main
nutrition_type = protein 유지
```

## #18 / PR #19

**PR 제목:** `feat(#18): agent1 이미지 재료 탐지 연동`

### Q1

detector가 `green leek` 또는 `scallion`을 반환했을 때, Agent1은 최종적으로 어떤 한국어 재료명으로 Agent3에 전달하나요?

### A1

```text
대파
```

`green leek`, `green onion`, `scallion` 계열은 표준명 매핑을 거쳐 `대파`로 정리됩니다.

### Q2

실제 이미지 detection 결과 신뢰도가 낮거나, `vegetable`처럼 너무 일반적인 라벨이 반환된다면 Agent1 출력에서 어떤 필드가 `true`로 동작하며, 왜 필요한가요?

### A2

```python
needs_confirmation = True
```

자동 인식 결과를 바로 확정하지 않고, 사용자가 확인하거나 수정할 수 있게 하기 위해 필요합니다.

예를 들어 `vegetable`은 너무 넓은 표현이고, confidence가 낮은 결과도 틀릴 가능성이 있으므로 사용자 확인 대상으로 넘깁니다.

## #20 / PR #21

**PR 제목:** `feat(#20): Agent1 재료 인식 결과 확인 및 수정 흐름 구현`

### Q1

Agent1이 `vision_status == "need_user_confirmation"`을 반환하면 graph는 다음 agent로 계속 진행하나요, 아니면 멈추나요?

### A1

멈춥니다.

`agents/graph.py`에서 `vision_status == "need_user_confirmation"`이면 다음 agent로 넘어가지 않고 `END`로 빠집니다.

사용자 확인/수정 입력이 들어온 뒤 다시 실행해야 다음 agent 흐름으로 진행됩니다.

### Q2

사용자가 `채소`를 제거하고 `대파`를 `쪽파`로 수정한 뒤 `간마늘, 설탕`을 추가하면, Agent1은 이 입력을 어떤 필드로 받아 최종 재료 목록에 반영하나요?

### A2

```python
ingredient_confirmation
```

구체적인 입력 예시는 다음과 같습니다.

```python
{
    "rejected_ingredients": ["채소"],
    "replacements": {"대파": "쪽파"},
    "additional_ingredients_text": "간마늘, 설탕",
}
```

Agent1은 이 값을 받아 다시 표준화하고 최종 재료 목록에 반영합니다.

예시 결과:

```python
available_ingredients = ["돼지고기", "쪽파", "다진마늘", "설탕"]
```

## Summary

```text
#11 기본 파이프라인
#15 Solar 표준화
#18 이미지 detection
#20 사용자 확인/수정
```
