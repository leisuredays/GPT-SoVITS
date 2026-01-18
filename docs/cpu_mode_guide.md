# CPU 모드로 TTS 실행 가이드

## 현재 상태 (GPU 모드)

```yaml
custom:
  device: cuda       # ← GPU 사용
  is_half: true      # ← FP16 (GPU 전용)
```

---

## CPU 모드로 전환

### 방법 1: tts_infer.yaml 직접 수정 ✏️

**파일:** `GPT_SoVITS/configs/tts_infer.yaml`

**변경:**
```yaml
custom:
  bert_base_path: GPT_SoVITS/pretrained_models/chinese-roberta-wwm-ext-large
  cnhuhbert_base_path: GPT_SoVITS/pretrained_models/chinese-hubert-base
  device: cpu        # ← cuda에서 cpu로 변경
  is_half: false     # ← true에서 false로 변경 (CPU는 FP16 미지원)
  t2s_weights_path: GPT_weights_v2ProPlus/yuzuha_4m-e20.ckpt
  version: v2ProPlus
  vits_weights_path: SoVITS_weights_v2ProPlus/yuzuha_4m_e8_s144.pth
```

**중요:** CPU는 FP16을 지원하지 않으므로 `is_half: false` 필수!

---

### 방법 2: 별도 설정 파일 생성 📄

CPU 전용 설정 파일을 만들어서 사용:

**파일 생성:** `GPT_SoVITS/configs/tts_infer_cpu.yaml`

```yaml
custom:
  bert_base_path: GPT_SoVITS/pretrained_models/chinese-roberta-wwm-ext-large
  cnhuhbert_base_path: GPT_SoVITS/pretrained_models/chinese-hubert-base
  device: cpu
  is_half: false
  t2s_weights_path: GPT_weights_v2ProPlus/yuzuha_4m-e20.ckpt
  version: v2ProPlus
  vits_weights_path: SoVITS_weights_v2ProPlus/yuzuha_4m_e8_s144.pth
```

**API 서버 시작:**
```bash
python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer_cpu.yaml
```

---

### 방법 3: 환경 변수로 제어 🌍

TTS.py가 환경 변수를 읽도록 수정할 수도 있지만, 가장 간단한 방법은 설정 파일 수정입니다.

---

## 빠른 적용 (1분) 🚀

```bash
# 1. 백업 생성
cp GPT_SoVITS/configs/tts_infer.yaml GPT_SoVITS/configs/tts_infer.yaml.gpu

# 2. CPU 모드로 변경
sed -i 's/device: cuda/device: cpu/' GPT_SoVITS/configs/tts_infer.yaml
sed -i '5s/is_half: true/is_half: false/' GPT_SoVITS/configs/tts_infer.yaml

# 3. 변경 확인
head -10 GPT_SoVITS/configs/tts_infer.yaml

# 4. API 서버 재시작
# Ctrl+C로 현재 서버 종료
python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer.yaml
```

---

## 성능 차이 예상 📊

### GPU vs CPU 비교

| 항목 | GPU (현재) | CPU (전환 후) |
|------|------------|--------------|
| **TTFB** | 0.42초 ⚡ | 2-5초 🐌 |
| **전체 시간** | 1.40초 | 5-15초 |
| **메모리** | GPU VRAM 2GB | RAM 4GB |
| **전력** | 높음 | 낮음 |
| **정밀도** | FP16 | FP32 |

### 속도 차이

```
GPU (CUDA + FP16):    0.42초  ████
CPU (FP32):           3.50초  ████████████████████████████████ (약 8배 느림)
```

**예상 성능:**
- 첫 청크 도착 (TTFB): **2-5초** (GPU의 0.42초 대비 5-12배 느림)
- 전체 생성: **5-15초** (GPU의 1.4초 대비 3-10배 느림)

---

## CPU 최적화 팁 ⚙️

CPU에서도 가능한 한 빠르게:

### 1. 가벼운 모델 사용
```yaml
custom:
  version: v2  # v2ProPlus 대신 v2 사용
  vits_weights_path: GPT_SoVITS/pretrained_models/gsv-v2final-pretrained/s2G2333k.pth  # 102MB
```

### 2. 샘플링 파라미터 최적화
```python
payload = {
    "top_k": 5,              # 15 → 5
    "temperature": 0.8,      # 1.0 → 0.8
    "batch_size": 1,
    "streaming_mode": 3,     # 가장 빠른 모드
}
```

### 3. 텍스트 분할 최적화
```python
payload = {
    "text_split_method": "cut5",  # 작은 청크로 분할
}
```

### 4. 병렬 처리 활성화
```python
payload = {
    "parallel_infer": True,  # CPU 멀티코어 활용
}
```

---

## 언제 CPU 모드를 사용할까? 🤔

### CPU 모드 권장 상황:
- ✅ GPU가 없는 환경
- ✅ 서버 비용 절감 (GPU 인스턴스 비쌈)
- ✅ 낮은 요청 빈도 (실시간 불필요)
- ✅ 배치 처리 (오프라인 생성)
- ✅ 개발/테스트 환경

### GPU 모드 권장 상황:
- ✅ 실시간 대화형 TTS
- ✅ 높은 요청 빈도
- ✅ 빠른 응답 필수
- ✅ 프로덕션 환경
- ✅ 동시 사용자 많음

---

## CPU 전용 설정 파일 예시

**파일:** `GPT_SoVITS/configs/tts_infer_cpu.yaml`

```yaml
custom:
  bert_base_path: GPT_SoVITS/pretrained_models/chinese-roberta-wwm-ext-large
  cnhuhbert_base_path: GPT_SoVITS/pretrained_models/chinese-hubert-base
  device: cpu
  is_half: false
  # 가벼운 v2 모델 사용 (CPU에 적합)
  t2s_weights_path: GPT_SoVITS/pretrained_models/gsv-v2final-pretrained/s1bert25hz-5kh-longer-epoch=12-step=369668.ckpt
  version: v2
  vits_weights_path: GPT_SoVITS/pretrained_models/gsv-v2final-pretrained/s2G2333k.pth
v1:
  # ... 기존 내용 유지 ...
```

---

## 테스트 방법

### 1. CPU 모드로 전환
```bash
# tts_infer.yaml 수정 후 서버 재시작
python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer.yaml
```

### 2. 속도 측정
```bash
python compare_model_speed.py "CPU모드" 3
```

### 3. GPU 모드와 비교
```bash
# 결과 비교
# GPU: TTFB 0.42초
# CPU: TTFB ???초
```

---

## 문제 해결 🔧

### 에러: "Half precision is not supported on CPU"
```
Warning: Half precision is not supported on CPU, set is_half to False.
```
**해결:** `is_half: false`로 설정

### 에러: "CUDA out of memory"
이미 CPU 모드면 발생하지 않음. GPU 모드에서만 발생.

### 너무 느림
**해결책:**
1. v2 모델 사용 (가벼움)
2. 샘플링 파라미터 최적화 (top_k=5)
3. CPU 코어 수 확인 (`lscpu`)
4. 백그라운드 프로세스 종료

---

## GPU로 다시 돌아가기

```bash
# 백업에서 복원
cp GPT_SoVITS/configs/tts_infer.yaml.gpu GPT_SoVITS/configs/tts_infer.yaml

# 또는 수동 변경
# device: cpu → cuda
# is_half: false → true

# 서버 재시작
python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer.yaml
```

---

## 요약 ✅

### CPU 모드 전환 2단계:

1. **설정 파일 수정:**
   ```yaml
   device: cpu
   is_half: false
   ```

2. **서버 재시작:**
   ```bash
   python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer.yaml
   ```

### 성능 비교:
- GPU: 0.42초 (빠름) ⚡
- CPU: 2-5초 (느림) 🐌

**지금 바로 CPU 모드로 전환할까요?** 🚀
