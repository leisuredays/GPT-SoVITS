# 서버 측 스트리밍 제어 가이드

## 현재 상태

스트리밍은 **클라이언트가 요청 시 제어**합니다:

```python
# 클라이언트 요청 예시
payload = {
    "text": "안녕하세요",
    "streaming_mode": 3,  # ← 클라이언트가 지정
}
```

하지만 **서버 측에서 기본값 변경 및 강제** 가능합니다.

---

## 방법 1: 서버 기본값 변경 ⚙️

### A) API 기본값 변경 (api_v2.py)

**파일:** `api_v2.py` 라인 171

**현재:**
```python
class TTS_Request(BaseModel):
    # ... 다른 파라미터들 ...
    streaming_mode: Union[bool, int] = False  # ← 기본값: 비활성화
```

**변경 1: 스트리밍 항상 켜기**
```python
class TTS_Request(BaseModel):
    streaming_mode: Union[bool, int] = 3  # ← 가장 빠른 스트리밍
```

**변경 2: 고품질 스트리밍**
```python
class TTS_Request(BaseModel):
    streaming_mode: Union[bool, int] = 1  # ← Best Quality
```

**변경 3: 중간 품질**
```python
class TTS_Request(BaseModel):
    streaming_mode: Union[bool, int] = 2  # ← Medium Quality
```

**적용 방법:**
```bash
# 1. api_v2.py 수정
# 2. API 서버 재시작
python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer.yaml
```

---

### B) GET 엔드포인트 기본값 변경

**파일:** `api_v2.py` 라인 478

**현재:**
```python
@APP.get("/tts")
async def tts_get_endpoint(
    # ... 다른 파라미터들 ...
    streaming_mode: Union[bool, int] = False,  # ← GET 기본값
```

**변경:**
```python
@APP.get("/tts")
async def tts_get_endpoint(
    streaming_mode: Union[bool, int] = 3,  # ← GET 요청 시 기본 스트리밍
```

---

## 방법 2: 스트리밍 강제 적용 🔒

클라이언트 요청을 무시하고 서버가 강제로 스트리밍 모드 지정:

**파일:** `api_v2.py`
**위치:** `async def tts_handle(req: dict):` 함수 내부

**현재 코드 (라인 380-410):**
```python
async def tts_handle(req: dict):
    streaming_mode = req.get("streaming_mode", False)
    # ... 클라이언트 요청 그대로 사용
```

**수정: 스트리밍 강제**
```python
async def tts_handle(req: dict):
    # 클라이언트 요청 무시하고 강제로 스트리밍 모드 3 사용
    streaming_mode = 3
    req["streaming_mode"] = 3

    # 또는 조건부 강제
    # streaming_mode = req.get("streaming_mode", False)
    # if streaming_mode == False or streaming_mode == 0:
    #     streaming_mode = 3  # 비활성화 요청은 모드 3으로 강제

    # ... 나머지 코드
```

---

## 방법 3: 특정 모드만 허용 ✅

**파일:** `api_v2.py`
**위치:** `async def tts_handle(req: dict):` 함수 내부

```python
async def tts_handle(req: dict):
    streaming_mode = req.get("streaming_mode", False)

    # 허용 모드 검증
    ALLOWED_MODES = [2, 3]  # 모드 2, 3만 허용

    if streaming_mode not in ALLOWED_MODES:
        streaming_mode = 3  # 기본값으로 변경
        req["streaming_mode"] = 3

    # 또는 에러 반환
    # if streaming_mode not in ALLOWED_MODES:
    #     return JSONResponse(
    #         status_code=400,
    #         content={"message": f"streaming_mode must be one of {ALLOWED_MODES}"}
    #     )

    # ... 나머지 코드
```

---

## 방법 4: 환경변수로 제어 🌍

더 유연한 방법:

**파일:** `api_v2.py` 상단에 추가

```python
import os

# 환경변수에서 기본 스트리밍 모드 읽기
DEFAULT_STREAMING_MODE = int(os.getenv("DEFAULT_STREAMING_MODE", "0"))
FORCE_STREAMING = os.getenv("FORCE_STREAMING", "false").lower() == "true"
```

**TTS_Request 클래스 수정:**
```python
class TTS_Request(BaseModel):
    streaming_mode: Union[bool, int] = DEFAULT_STREAMING_MODE
```

**tts_handle 함수 수정:**
```python
async def tts_handle(req: dict):
    if FORCE_STREAMING:
        streaming_mode = DEFAULT_STREAMING_MODE
        req["streaming_mode"] = DEFAULT_STREAMING_MODE
    else:
        streaming_mode = req.get("streaming_mode", DEFAULT_STREAMING_MODE)
```

**사용 방법:**
```bash
# 기본 모드 3, 강제 적용
export DEFAULT_STREAMING_MODE=3
export FORCE_STREAMING=true

python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer.yaml
```

---

## 실전 예시 코드

### 예시 1: 항상 모드 3 (가장 빠름)

```python
# api_v2.py 수정

class TTS_Request(BaseModel):
    # ... 다른 파라미터들 ...
    streaming_mode: Union[bool, int] = 3  # ← 이렇게 변경

@APP.get("/tts")
async def tts_get_endpoint(
    # ... 다른 파라미터들 ...
    streaming_mode: Union[bool, int] = 3,  # ← 이렇게 변경
```

### 예시 2: 조건부 강제

```python
# api_v2.py의 tts_handle 함수에 추가

async def tts_handle(req: dict):
    streaming_mode = req.get("streaming_mode", False)

    # 텍스트가 길면 스트리밍 강제
    text = req.get("text", "")
    if len(text) > 50 and streaming_mode == False:
        streaming_mode = 3
        req["streaming_mode"] = 3
        print(f"Long text detected, forcing streaming mode 3")

    # ... 나머지 코드
```

### 예시 3: 완전히 끄기 (스트리밍 비활성화)

```python
# api_v2.py의 tts_handle 함수에 추가

async def tts_handle(req: dict):
    # 스트리밍 완전히 비활성화
    streaming_mode = False
    req["streaming_mode"] = False

    # 또는 요청을 무시하고 강제
    # req["streaming_mode"] = 0

    # ... 나머지 코드
```

---

## 추천 설정 ⭐

### 일반적인 경우 (기본값)
```python
# api_v2.py
class TTS_Request(BaseModel):
    streaming_mode: Union[bool, int] = False  # 클라이언트가 선택
```
→ 클라이언트가 필요에 따라 선택

### 실시간 대화형 서비스
```python
# api_v2.py
class TTS_Request(BaseModel):
    streaming_mode: Union[bool, int] = 3  # 항상 가장 빠른 스트리밍
```
→ 모든 요청에 빠른 응답

### 고품질 우선 서비스
```python
# api_v2.py
class TTS_Request(BaseModel):
    streaming_mode: Union[bool, int] = 1  # Best Quality
```
→ 품질 우선, 속도 희생

---

## 테스트 방법

### 1. 기본값 테스트
```bash
# 파라미터 없이 요청
curl "http://127.0.0.1:9880/tts?text=안녕하세요&text_lang=ko&ref_audio_path=yuzuha_reference.wav&prompt_lang=ko"
```

### 2. 클라이언트 지정 무시 테스트
```python
# 강제 설정이 적용되는지 확인
payload = {
    "text": "안녕하세요",
    "streaming_mode": 0,  # 비활성화 요청
}
# → 서버가 3으로 강제하는지 확인
```

---

## 현재 권장 사항

**상황에 따라:**

### 시나리오 1: 일반 API 서버
```python
# 변경 안함 - 기본값 유지
streaming_mode: Union[bool, int] = False
```
→ 클라이언트가 상황에 맞게 선택

### 시나리오 2: 실시간 대화 전용
```python
# 항상 스트리밍
streaming_mode: Union[bool, int] = 3
```
→ 모든 요청 빠른 응답

### 시나리오 3: 내부 서비스 (제어됨)
```python
# 환경변수로 제어
DEFAULT_STREAMING_MODE = int(os.getenv("DEFAULT_STREAMING_MODE", "3"))
```
→ 배포 환경마다 다르게 설정

---

## 빠른 적용 가이드

**1분 만에 적용:**

```bash
# 1. api_v2.py 열기
nano api_v2.py  # 또는 vi, vim

# 2. 라인 171 찾기 (Ctrl+W로 검색: "streaming_mode: Union")
streaming_mode: Union[bool, int] = False

# 3. False를 3으로 변경
streaming_mode: Union[bool, int] = 3

# 4. 저장 후 서버 재시작
# Ctrl+C로 현재 서버 종료
python api_v2.py -a 127.0.0.1 -p 9880 -c GPT_SoVITS/configs/tts_infer.yaml

# 완료! 이제 모든 요청이 기본적으로 스트리밍 모드 3 사용
```

**적용할까요?** 🚀
