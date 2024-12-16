# 프롬프트 튜닝

프롬프트 튜닝은 모델 가중치 대신 입력 표현을 변경하는 파라미터 효율적인 접근법입니다. 모델의 모든 파라미터를 업데이트하는 전통적인 미세 조정과 다르게, 프롬프트 튜닝은 기본 모델의 파라미터가 고정된 상태를 유지하면서 소수의 학습 가능한 토큰을 추가하고 최적화합니다. 

## 프롬프트 튜닝 이해하기

프롬프트 튜닝은 모델 미세 조정의 대안으로, 학습 가능한 연속 벡터(소프트 프롬프트)를 입력 텍스트에 추가하는 파라미터 효율적인 방식입니다. 소프트 프롬프트는 이산적인 텍스트 프롬프트와 다르게 모델 파라미터가 고정된 상태에서 역전파를 통해 학습됩니다. 이 방식은 ["The Power of Scale for Parameter-Efficient Prompt Tuning"](https://arxiv.org/abs/2104.08691) (Lester et al., 2021)에서 처음 소개되었으며, 이 논문은 모델 크기가 커질수록 미세 조정과 비교했을 때 프롬프트 튜닝이 경쟁력을 가진다는 것을 보였습니다. 논문에 따르면, 약 100억 개의 파라미터를 가진 모델에서 프롬프트 튜닝의 성능은 태스크당 수백 개의 매개변수 수정만으로 모델 미세 조정의 성능과 일치하는 것으로 나타났습니다.

모델의 임베딩 공간에 존재하는 연속적인 벡터인 소프트 프롬프트는 학습 과정에서 최적화됩니다. 자연어 토큰을 사용하는 전통적인 이산 프롬프트와 달리 소프트 프롬프트는 고유한 의미를 가지고 있지는 않지만 경사 하강법을 통해 파라미터가 고정된 모델에서 원하는 동작을 이끌어내는 방법을 학습합니다. 이 기법은 모델 전체를 복사하지 않고 적은 수(일반적으로 몇백 개의 파라미터)에 해당하는 프롬프트 벡터만을 저장하기 때문에 멀티 태스크 시나리오에 더욱 효과적입니다. 이 접근 방식은 최소한의 메모리 사용량을 유지할 뿐만 아니라, 프롬프트 벡터만 교체하여 모델을 다시 로드하지 않고도 신속하게 작업을 전환할 수 있게 합니다.

## 학습 과정

소프트 프롬프트는 일반적으로 8~32개의 토큰으로 구성되며 무작위 또는 기존 텍스트로 초기화할 수 있습니다. 초기화 방법은 학습 과정에서 중요한 역할을 하며 텍스트 기반 초기화가 무작위 초기화보다 좋은 성능을 보이는 경우가 많습니다.

학습이 진행되는 동안 기본 모델의 파라미터는 고정되어 있고 프롬프트 파라미터만 업데이트 됩니다. 프롬프트 파라미터에 초점을 맞춘 이 접근 방식은 일반적인 학습 목표를 따르지만 학습률과 프롬프트 토큰의 경사 동작에 세심한 주의가 필요합니다.

## PEFT 라이브러리를 활용한 프롬프트 튜닝 구현

PEFT 라이브러리를 이용해 프롬프트 튜닝을 직관적으로 구현할 수 있습니다. 기본적인 예제는 다음과 같습니다:

```python
from peft import PromptTuningConfig, TaskType, get_peft_model
from transformers import AutoModelForCausalLM, AutoTokenizer

# 기본 모델 불러오기
model = AutoModelForCausalLM.from_pretrained("your-base-model")
tokenizer = AutoTokenizer.from_pretrained("your-base-model")

# 프롬프트 튜닝 configuration 설정
peft_config = PromptTuningConfig(
    task_type=TaskType.CAUSAL_LM,
    num_virtual_tokens=8,  # 학습 가능한 토큰 수
    prompt_tuning_init="TEXT",  # 텍스트 기반 초기화
    prompt_tuning_init_text="Classify if this text is positive or negative:",
    tokenizer_name_or_path="your-base-model",
)

# 프롬프트 튜닝 가능한 모델 생성
model = get_peft_model(model, peft_config)
```

## 프롬프트 튜닝과 다른 PEFT 방법 비교

다른 PEFT 접근 방식과 비교했을 때 프롬프트 튜닝은 효율성이 뛰어납니다. LoRA는 파라미터 수와 메모리 사용량이 적지만 태스크 전환을 위해 어댑터를 불러와야 하는 반면 프롬프트 튜닝은 자원을 더 적게 사용하면서 즉각적인 태스크 전환이 가능합니다. 반면에 전체 미세 조정은 상당한 양의 자원을 필요로 하고 서로 다른 태스크를 수행하기 위해 태스크에 맞는 각각의 모델을 필요로 합니다.

| 방법 | 파라미터 수 | 메모리 사용량 | 태스크 전환 |
|--------|------------|---------|----------------|
| 프롬프트 튜닝 | 매우 적음 | 메모리 사용량 최소화 | 쉬움 |
| LoRA | 적음 | 적음 | 모델을 다시 불러와야 함 |
| 전체 미세 조정 | 많음 | 많음 | 새로운 모델 복사 필요 |

프롬프트 튜닝을 구현할 때는 적은 수(8~16개)의 가상 토큰으로 시작한 후 태스크의 복잡성에 따라 토큰 수를 점차 늘려가세요. 태스크와 관련된 텍스트로 초기화를 수행하는 것이 무작위 초기화보다 일반적으로 더 나은 결과를 가져옵니다. 초기화 전략은 목표 태스크의 복잡성을 반영해야 합니다.

학습 과정은 전체 미세 조정과 약간의 차이가 있습니다. 보통 높은 학습률이 효과적이지만 프롬프트 토큰 경사를 신중하게 모니터링해야 합니다. 다양한 예제를 활용한 주기적인 검증은 서로 다른 시나리오에서 강력한 성능을 보장하는 데 도움이 됩니다.

## 응용

프롬프트 튜닝은 다음과 같은 시나리오에서 특히 뛰어납니다:

1. 멀티 태스크 배포
2. 자원이 제한적인 환경
3. 신속한 태스크 적응
4. 프라이버시에 민감한 애플리케이션

그러나 모델 크기가 작아질수록 전체 미세 조정과 비교했을 때 프롬프트 튜닝의 경쟁력이 떨어집니다. 예를 들어, SmolLM2 크기의 모델은 프롬프트 튜닝보다 전체 미세 조정를 수행하는 것이 좀 더 의미있습니다.

## 다음 단계

⏭️ [LoRA 어댑터 튜토리얼](./notebooks/finetune_sft_peft.ipynb)에서 LoRA 어댑터를 활용해 모델을 미세 조정하는 법을 배워봅시다.

## 참고
- [Hugging Face PEFT 문서](https://huggingface.co/docs/peft)
- [프롬프트 튜닝 논문](https://arxiv.org/abs/2104.08691)
- [Hugging Face Cookbook](https://huggingface.co/learn/cookbook/prompt_tuning_peft)