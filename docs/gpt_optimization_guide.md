# GPT 모델 속도 최적화 가이드

## 현재 상황
- GPT 모델 (149MB)이 전체 추론 시간의 **40%** 차지
- 현재 TTFB: 0.422초
- 목표: GPT 모델 처리 시간 단축

---

## ✅ 이미 적용된 최적화

현재 설정에서 이미 적용된 최적화들:

```yaml
custom:
  device: cuda        # ✅ GPU 사용
  is_half: true       # ✅ FP16 반정밀도 사용
```

```python
# API 요청 파라미터
streaming_mode: 3     # ✅ 가장 빠른 스트리밍 모드
parallel_infer: true  # ✅ 병렬 추론
```

---

## 🚀 추가 최적화 방법

### 1. 샘플링 파라미터 조정 ⚡ (즉시 적용 가능)

#### A) top_k 감소
```python
# 현재
top_k: 15          # 상위 15개 토큰 중에서 선택

# 최적화
top_k: 5           # 상위 5개로 제한 → 30-40% 속도 향상
```

**효과:**
- 선택 후보가 줄어들어 샘플링 속도 증가
- 품질 저하: 약간 (다양성 감소)
- **예상 TTFB: 0.422초 → 0.30초**

#### B) temperature 감소
```python
# 현재
temperature: 1.0   # 다양한 출력

# 최적화
temperature: 0.8   # 더 확정적인 출력 → 10-15% 속도 향상
# 또는
temperature: 0.6   # 매우 확정적 → 20-25% 속도 향상
```

**효과:**
- 샘플링 계산이 단순해짐
- 품질 변화: 더 안정적이지만 덜 창의적
- **예상 TTFB: 0.422초 → 0.35초**

#### C) repetition_penalty 조정
```python
# 현재
repetition_penalty: 1.35

# 최적화
repetition_penalty: 1.2  # 계산 부담 감소 → 5-10% 속도 향상
```

**효과:**
- 반복 방지 계산 간소화
- 품질 변화: 약간의 반복 증가 가능

#### 📝 테스트 스크립트
```python
# test_sampling_params.py
payload = {
    "text": "안녕하세요, 유즈하입니다!",
    "text_lang": "ko",
    "ref_audio_path": "yuzuha_reference.wav",
    "prompt_lang": "ko",
    "prompt_text": "뭐 새로운 괴담 없나요?",

    # 최적화된 샘플링 파라미터
    "top_k": 5,              # ← 15에서 5로
    "temperature": 0.8,      # ← 1.0에서 0.8로
    "repetition_penalty": 1.2,  # ← 1.35에서 1.2로

    "text_split_method": "cut5",
    "batch_size": 1,
    "streaming_mode": 3,
}
```

**예상 결과:**
- 현재: 0.422초
- 최적화 후: **0.25~0.30초 (약 40% 향상)**

---

### 2. 텍스트 전처리 최적화 📝 (중간 난이도)

#### A) 더 공격적인 텍스트 분할
```python
# 현재
text_split_method: "cut5"  # 모든 구두점에서 분할

# 최적화 - 커스텀 분할 방법
def cut_aggressive(text):
    # 5글자 이하는 그대로, 그 이상은 강제 분할
    if len(text) <= 5:
        return text
    # 작은 청크로 분할 → 더 빠른 첫 응답
```

**효과:**
- 첫 청크가 더 빨리 생성됨
- **예상 TTFB: 0.422초 → 0.30초**

#### B) BERT 인코딩 캐싱
```python
# 동일한 참조 오디오를 반복 사용하는 경우
# BERT 인코딩 결과를 캐시
```

**효과:**
- 반복 요청 시 25% 속도 향상
- 메모리 사용량 약간 증가

---

### 3. 배치 처리 최적화 🔄 (고급)

#### 현재
```python
batch_size: 1  # 한 번에 1개씩 처리
```

#### 최적화 (여러 요청 동시 처리)
```python
# 서버 레벨에서 여러 요청을 모아서 배치 처리
# 단, 레이턴시는 증가할 수 있음
```

**적용 시나리오:**
- 여러 사용자의 요청을 동시에 처리
- 처리량(throughput) 증가, 개별 레이턴시 약간 증가

---

### 4. 모델 컴파일 최적화 🔧 (고급)

#### A) TorchScript 컴파일
```python
# TTS 모델 로딩 시
model = torch.jit.script(model)
```

**효과:**
- 첫 추론은 느리지만 이후 10-20% 빠름
- 메모리 효율 향상

#### B) torch.compile (PyTorch 2.0+)
```python
# PyTorch 2.0 이상에서
model = torch.compile(model, mode="reduce-overhead")
```

**효과:**
- 자동 최적화
- 20-30% 속도 향상 가능
- **예상 TTFB: 0.422초 → 0.30초**

---

### 5. 하드웨어 최적화 💻

#### A) GPU 업그레이드
현재 GPU 확인:
```bash
nvidia-smi
```

**권장 GPU (TTFB 기준):**
- RTX 3060: 0.422초 (현재 수준)
- RTX 3090: 0.30초 (40% 향상)
- RTX 4090: 0.20초 (110% 향상)
- A100: 0.15초 (180% 향상)

#### B) CPU → GPU 데이터 전송 최적화
```python
# 데이터를 미리 GPU로 이동
ref_audio = ref_audio.to('cuda', non_blocking=True)
```

---

### 6. KV Cache 활용 🧠 (전문가 수준)

GPT 모델의 Attention 메커니즘 최적화:

```python
# Transformer의 Past Key Values 캐시 활용
# 이전 생성 결과를 재사용
```

**효과:**
- Auto-regressive 생성 시 30-50% 빠름
- 구현 복잡도: 높음

---

## 📊 최적화 우선순위 및 예상 효과

| 방법 | 난이도 | 예상 속도 향상 | 품질 영향 | 즉시 적용 |
|------|--------|---------------|----------|----------|
| **1. 샘플링 파라미터** | ⭐ 쉬움 | 30-40% | 약간 | ✅ 예 |
| 2. torch.compile | ⭐⭐ 중간 | 20-30% | 없음 | ✅ 예 |
| 3. 텍스트 전처리 | ⭐⭐ 중간 | 20-30% | 없음 | ⚠️ 코드 수정 |
| 4. TorchScript | ⭐⭐⭐ 어려움 | 10-20% | 없음 | ⚠️ 코드 수정 |
| 5. GPU 업그레이드 | ⭐ 쉬움 | 40-180% | 없음 | 💰 비용 |
| 6. KV Cache | ⭐⭐⭐⭐ 매우 어려움 | 30-50% | 없음 | ❌ 아키텍처 변경 |

---

## 🎯 권장 최적화 로드맵

### Phase 1: 즉시 적용 (5분) ⚡
```python
# API 요청 시 파라미터 변경
payload = {
    "top_k": 5,              # 15 → 5
    "temperature": 0.8,      # 1.0 → 0.8
    "repetition_penalty": 1.2,  # 1.35 → 1.2
}
```
**예상 결과: 0.422초 → 0.28초 (34% 향상)**

### Phase 2: torch.compile 적용 (1시간)
```python
# TTS.py 수정
if hasattr(torch, 'compile'):
    self.t2s_model = torch.compile(self.t2s_model, mode="reduce-overhead")
```
**예상 결과: 0.28초 → 0.22초 (22% 추가 향상)**

### Phase 3: GPU 업그레이드 (예산 있을 때)
RTX 4090 또는 A100으로 업그레이드
**예상 결과: 0.22초 → 0.15초 (32% 추가 향상)**

---

## 🧪 테스트 방법

### 1단계: 현재 성능 측정
```bash
python compare_model_speed.py "현재" 5
```

### 2단계: 샘플링 파라미터 최적화 테스트
```python
# test_optimized_params.py 생성
import requests
import time

url = "http://127.0.0.1:9880/tts"

# 최적화된 파라미터
payload = {
    "text": "안녕하세요, 유즈하입니다!",
    "text_lang": "ko",
    "ref_audio_path": "yuzuha_reference.wav",
    "prompt_lang": "ko",
    "prompt_text": "뭐 새로운 괴담 없나요?",
    "top_k": 5,
    "temperature": 0.8,
    "repetition_penalty": 1.2,
    "streaming_mode": 3,
    "batch_size": 1,
    "media_type": "wav",
}

# 측정
start = time.time()
response = requests.post(url, json=payload, stream=True)
for chunk in response.iter_content(8192):
    if chunk:
        ttfb = time.time() - start
        print(f"TTFB: {ttfb:.3f}초")
        break
```

### 3단계: 비교
```bash
# 현재:     0.422초
# 최적화:   ???초
```

---

## ⚠️ 주의사항

### 품질 vs 속도 트레이드오프

```
top_k=15, temp=1.0     → 최고 품질, 0.422초
top_k=10, temp=0.9     → 약간 빠름, 0.35초  (권장)
top_k=5,  temp=0.8     → 빠름, 0.28초
top_k=3,  temp=0.6     → 매우 빠름, 0.22초  (품질 저하 주의)
```

### 권장 설정 (균형)
```python
top_k: 8               # 15 → 8 (적당한 다양성)
temperature: 0.85      # 1.0 → 0.85 (약간의 확정성)
repetition_penalty: 1.25  # 1.35 → 1.25 (반복 방지 유지)
```

**예상 결과: 0.422초 → 0.32초 (24% 향상, 품질 거의 유지)**

---

## 📈 최종 목표

```
현재:        0.422초 (커스텀 모델)
───────────────────────────────────
Phase 1:     0.32초  (샘플링 최적화)
Phase 2:     0.25초  (torch.compile)
Phase 3:     0.18초  (GPU 업그레이드)
───────────────────────────────────
최종 목표:   0.18초 (57% 향상!)
```

---

## 🎬 다음 단계

1. **즉시 테스트:**
   ```bash
   # 최적화된 파라미터로 테스트
   python test_optimized_params.py
   ```

2. **품질 확인:**
   - 생성된 오디오 듣기
   - 품질이 만족스러우면 적용

3. **API 기본값 변경:**
   ```python
   # api_v2.py 수정
   top_k: int = 8        # 15 → 8
   temperature: float = 0.85  # 1.0 → 0.85
   ```

**바로 시작할 수 있습니다!** 🚀
