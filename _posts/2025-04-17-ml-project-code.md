---
title: "생성형 AI 미니 프로젝트(2)-2: 실시간 STT 구현하기"
categories: [Fast API]
tags:
  [
    ML,
    Fast API,
    STT,
    Open AI API,
    WebSocket,
    Thread,
    Coroutine,
    SKALA,
    SKALA1기,
    SK
  ]
---

> Skala 과정에서 생성형 AI 대해 새롭게 알게 되었습니다.<br>
> 생성형 AI 수업 중 진행하게 된 미니 프로젝트 구현 과정을 기술합니다.

[처음 글 보러가기](<https://sermadl.github.io/posts/ml-project(1)/>)

<hr>

이번 글에서는 웹소켓, 쓰레드, 코루틴을 활용해서 실제로 실시간으로 STT를 구현해보겠습니당<br>

## STT 변환

### stt_router.py

```python
import logging
from fastapi import APIRouter, WebSocket, WebSocketDisconnect, UploadFile, File
from fastapi.responses import StreamingResponse
from app.services.google_stt_service import transcribe_streaming_v2
from app.services.websocket_stt_service import handle_websocket_connection

router = APIRouter()

# 로깅 설정
logger = logging.getLogger(__name__)

@router.websocket("/websocket")
async def websocket_endpoint(websocket: WebSocket):
    """
    ## 웹소켓으로 오디오 파일 전송 및 실시간 STT 결과 변환
    """
    try:
        # 웹소켓 연결 및 처리 로직은 websocket_stt.py로 위임
        await websocket.accept()
        await handle_websocket_connection(websocket)
    except WebSocketDisconnect:
        logger.info("WebSocket 연결이 종료되었습니다")
    except Exception as e:
        logger.error(f"WebSocket 오류: {str(e)}")
```

라우터에서는 위와 같이 웹소켓 변수를 만들고, 연결한 후에<br>
`handle_websocket_connection` 함수로 웹소켓을 전달하여<br>
라우터의 역할만 하도록 책임을 분리했습니다<br>

<hr>

### main.py

```python
import logging
import asyncio
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
from app.api import stt_router, openai_router, user_router, script_router, receipt_router
from app.db.reset_database import reset_database
from dotenv import load_dotenv
from app.services import websocket_stt_service

load_dotenv()

app = FastAPI()

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler()
    ]
)

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# 정적 파일 서빙 설정
app.mount("/static", StaticFiles(directory="app/static", html=True), name="static")

@app.on_event("startup")
async def on_startup():
    """애플리케이션 시작 시 데이터베이스 초기화"""
    await reset_database(force_reset=False)
    websocket_stt_service.loop = asyncio.get_running_loop()

app.include_router(stt_router.router, prefix="/stt", tags=["Google STT"])
app.include_router(openai_router.router, prefix="/openai", tags=["OpenAI"])
app.include_router(user_router.router, prefix="/user", tags=["User"])
app.include_router(script_router.router, prefix="/script", tags=["Script"])
app.include_router(receipt_router.router, prefix="/receipt", tags=["Receipt"])
```

`@app.on_evenet("startup")` 에서 동기 환경에서도 비동기 코드(코루틴)를 실행하기 위해 이벤트 루프 객체를 만듭니다.<br>

### websocket_stt_service.py

```python
import os
import json
import logging
import asyncio
import threading
import queue
import time
from dotenv import load_dotenv
from fastapi import WebSocket
from google.cloud.speech_v2 import SpeechClient
from google.cloud.speech_v2.types import cloud_speech as cloud_speech_types

# 환경 변수 로드
load_dotenv()
PROJECT_ID = os.getenv("PROJECT_ID")

loop = None

async def handle_websocket_connection(websocket: WebSocket):
    """
    WebSocket 연결을 처리하는 함수

    Args:
        websocket (WebSocket): 연결된 WebSocket 객체
    """
    # 상태 관리 변수
    is_active = False
    language_code = "ko-KR"  # 기본 언어

    # 스레드 간 데이터 교환을 위한 큐
    audio_queue = queue.Queue()
    response_queue = asyncio.Queue()

    # STT 스트리밍 스레드
    stt_thread = None

    try:
        while True:
            # 클라이언트로부터 메시지 수신
            message = await websocket.receive()

            # 텍스트 메시지인 경우 (명령 처리)
            if "text" in message:
                try:
                    data = json.loads(message["text"])
                    msg_type = data.get("type")

                    if msg_type == "start":
                        # 이전 스레드가 실행 중이면 종료
                        if stt_thread and stt_thread.is_alive():
                            audio_queue.put(None)  # 종료 신호
                            stt_thread.join(timeout=2)

                        # 큐 초기화
                        while not audio_queue.empty():
                            audio_queue.get()
                        while not response_queue.empty():
                            await response_queue.get()

                        # 녹음 시작 명령
                        language_code = data.get("lang", "ko-KR")
                        is_active = True

                        await websocket.send_text(json.dumps({
                            "type": "system",
                            "message": f"인식 시작: 언어 - {language_code}"
                        }, ensure_ascii=False))

                        logging.info(f"음성 인식 시작: 언어 - {language_code}")

                        # 새로운 STT 스레드 시작
                        stt_thread = threading.Thread(
                            target=run_stt_stream,
                            args=(audio_queue, response_queue, language_code)
                        )
                        stt_thread.daemon = True
                        stt_thread.start()

                        # 응답 처리 태스크 시작
                        asyncio.create_task(process_responses(response_queue, websocket))

                    elif msg_type == "end":
                        # 녹음 종료 명령
                        is_active = False

                        # 스트리밍 종료 신호
                        audio_queue.put(None)

                        logging.info("음성 인식 종료, 다음 세션 대기 중")

                except json.JSONDecodeError:
                    logging.error("잘못된 JSON 형식")

            # 바이너리 메시지인 경우 (오디오 데이터)
            elif "bytes" in message and is_active:
                audio_chunk = message["bytes"]
                # 오디오 데이터를 큐에 추가
                audio_queue.put(audio_chunk)

    except Exception as e:
        logging.error(f"WebSocket 오류: {str(e)}")
        # 스레드 정리
        if stt_thread and stt_thread.is_alive():
            audio_queue.put(None)  # 종료 신호

def run_stt_stream(audio_queue, response_queue, language_code):
    """
    별도 스레드에서 STT 스트리밍을 실행하는 함수

    Args:
        audio_queue (queue.Queue): 오디오 데이터를 받는 큐
        response_queue (asyncio.Queue): 처리 결과를 전달할 큐
        language_code (str): 인식 언어 코드
    """
    try:
        client = SpeechClient()

        # 음성 인식 설정
        recognition_config = cloud_speech_types.RecognitionConfig(
            explicit_decoding_config=cloud_speech_types.ExplicitDecodingConfig(
                encoding=cloud_speech_types.ExplicitDecodingConfig.AudioEncoding.LINEAR16,
                sample_rate_hertz=16000,
                audio_channel_count=1,
            ),
            language_codes=[language_code],
            model="long",
        )

        streaming_features = cloud_speech_types.StreamingRecognitionFeatures(
            interim_results=True,
        )

        streaming_config = cloud_speech_types.StreamingRecognitionConfig(
            config=recognition_config,
            streaming_features=streaming_features,
        )

        config_request = cloud_speech_types.StreamingRecognizeRequest(
            recognizer=f"projects/{PROJECT_ID}/locations/global/recognizers/_",
            streaming_config=streaming_config,
        )

        # 동기 제너레이터 - STT API 호출에 사용
        def request_generator():
            # 설정 요청 먼저 보내기
            yield config_request

            while True:
                # 큐에서 오디오 데이터 가져오기 (blocking)
                chunk = audio_queue.get()

                # None은 스트림 종료 신호
                if chunk is None:
                    break

                # 오디오 데이터 요청 생성 및 전송
                yield cloud_speech_types.StreamingRecognizeRequest(audio=chunk)

        # Google STT API 호출
        responses = client.streaming_recognize(requests=request_generator())

        # 응답 처리 및 결과 큐에 추가
        for response in responses:
            # 결과가 없는 경우 스킵
            if not response.results:
                continue

            # 마지막 결과만 처리 (일반적으로 가장 최신, 가장 정확한 결과)
            result = response.results[-1]

            if result.alternatives:
                transcript = result.alternatives[0].transcript
                is_final = result.is_final
                confidence = result.alternatives[0].confidence if hasattr(result.alternatives[0], 'confidence') else 0.0

                def send_response():
                    asyncio.create_task(response_queue.put({
                        "transcript": transcript,
                        "is_final": is_final,
                        "confidence": confidence,
                    }))

                loop.call_soon_threadsafe(send_response)

    except Exception as e:
        logging.error(f"STT 스트리밍 오류: {str(e)}")
        # 오류 정보 전달
        try:
            asyncio.run(response_queue.put({
                "error": str(e)
            }))
        except:
            pass

async def process_responses(response_queue, websocket):
    """
    STT 응답을 처리하고 WebSocket으로 전송하는 함수
    중간 결과는 'interim' 타입으로, 최종 결과는 'final' 타입으로 전송합니다.

    Args:
        response_queue (asyncio.Queue): STT 결과를 받는 큐
        websocket (WebSocket): 결과를 전송할 WebSocket
    """
    # 현재 상태 관리
    last_interim_text = ""  # 마지막으로 전송한 중간 텍스트
    last_final_text = ""    # 마지막으로 전송한 최종 텍스트

    try:
        while True:
            # 큐에서 응답 가져오기
            response = await response_queue.get()

            # 오류 발생 시
            if "error" in response:
                error_response = {
                    "type": "error",
                    "message": f"음성 인식 오류: {response['error']}"
                }
                await websocket.send_text(json.dumps(error_response, ensure_ascii=False))
                continue

            # 정상 응답 처리
            transcript = response["transcript"].strip()
            is_final = response["is_final"]
            # confidence = response.get("confidence", 0.0)

            # 텍스트가 비어있으면 무시
            if not transcript:
                continue

            # 결과 유형에 따라 처리
            if is_final:
                # 최종 결과가 이전 최종 결과와 다를 경우에만 전송
                if transcript != last_final_text:
                    json_response = {
                        "type": "final",
                        "text": transcript,
                        # "confidence": confidence
                    }
                    await websocket.send_text(json.dumps(json_response, ensure_ascii=False))
                    last_final_text = transcript

                # 중간 결과 초기화
                last_interim_text = ""

            else:
                # 중간 결과가 이전 중간 결과와 다를 경우에만 전송
                if transcript != last_interim_text:
                    json_response = {
                        "type": "interim",
                        "text": transcript,
                        # "confidence": confidence
                    }
                    await websocket.send_text(json.dumps(json_response, ensure_ascii=False))
                    last_interim_text = transcript

    except asyncio.CancelledError:
        # 태스크 취소
        return
    except Exception as e:
        logging.error(f"응답 처리 오류: {str(e)}")
        error_response = {
            "type": "error",
            "message": f"응답 처리 오류: {str(e)}"
        }
        await websocket.send_text(json.dumps(error_response, ensure_ascii=False))
```

코드가 쪼오금 길지만 천천히 함수 하나씩 뜯어보겠습니다.<br>

**[ handel_websocket_connection ]**

```python
async def handle_websocket_connection(websocket: WebSocket):
    """
    WebSocket 연결을 처리하는 함수

    Args:
        websocket (WebSocket): 연결된 WebSocket 객체
    """
    # 상태 관리 변수
    is_active = False
    language_code = "ko-KR"  # 기본 언어

    # 스레드 간 데이터 교환을 위한 큐
    audio_queue = queue.Queue()
    response_queue = asyncio.Queue()

    # STT 스트리밍 스레드
    stt_thread = None

    try:
        while True:
            # 클라이언트로부터 메시지 수신
            message = await websocket.receive()

            # 텍스트 메시지인 경우 (명령 처리)
            if "text" in message:
                try:
                    data = json.loads(message["text"])
                    msg_type = data.get("type")

                    if msg_type == "start":
                        # 이전 스레드가 실행 중이면 종료
                        if stt_thread and stt_thread.is_alive():
                            audio_queue.put(None)  # 종료 신호
                            stt_thread.join(timeout=2)

                        # 큐 초기화
                        while not audio_queue.empty():
                            audio_queue.get()
                        while not response_queue.empty():
                            await response_queue.get()

                        # 녹음 시작 명령
                        language_code = data.get("lang", "ko-KR")
                        is_active = True

                        await websocket.send_text(json.dumps({
                            "type": "system",
                            "message": f"인식 시작: 언어 - {language_code}"
                        }, ensure_ascii=False))

                        logging.info(f"음성 인식 시작: 언어 - {language_code}")

                        # 새로운 STT 스레드 시작
                        stt_thread = threading.Thread(
                            target=run_stt_stream,
                            args=(audio_queue, response_queue, language_code)
                        )
                        stt_thread.daemon = True
                        stt_thread.start()

                        # 응답 처리 태스크 시작
                        asyncio.create_task(process_responses(response_queue, websocket))

                    elif msg_type == "end":
                        # 녹음 종료 명령
                        is_active = False

                        # 스트리밍 종료 신호
                        audio_queue.put(None)

                        # await websocket.send_text(json.dumps({
                        #     "type": "system",
                        #     "message": "인식 종료: 다음 세션 대기 중"
                        # }, ensure_ascii=False))

                        logging.info("음성 인식 종료, 다음 세션 대기 중")

                except json.JSONDecodeError:
                    logging.error("잘못된 JSON 형식")

            # 바이너리 메시지인 경우 (오디오 데이터)
            elif "bytes" in message and is_active:
                audio_chunk = message["bytes"]
                # 오디오 데이터를 큐에 추가
                audio_queue.put(audio_chunk)

    except Exception as e:
        logging.error(f"WebSocket 오류: {str(e)}")
        # 스레드 정리
        if stt_thread and stt_thread.is_alive():
            audio_queue.put(None)  # 종료 신호
```

우선 제일 먼저 실행되는 `handel_websocket_connection` 함수를 살펴보면<br>

쓰레드 간 데이터 교환을 위해 음성 바이트를 저장할 `audio_queue`,<br>
API 호출 후 STT 변환 결과가 저장될 `response_queue` 를 선언해두었습니다.<br>

웹소켓이 연결되어 있는 한, 계속해서 클라이언트로부터 메시지를 수신하도록 기다립니다.<br>

클라이언트로부터 전달 받을 데이터의 형태는 "type: start, lang: {언어 코드}", "type: end", 음성 바이트 입니다.<br>
이렇게 구분한 이유는

1. 구글 API가 5분 이상 연결을 할 수 없다는 졔약사항이 있었고<br>
2. 계속해서 음성을 받는다면 사용자가 원하지 않는 음성도 전달받게될 가능성이 컸기 때문입니다.

> 클라이언트에서 "시작" 버튼을 누르면 음성을 녹음 해 0.5초 단위로 끊어 웹소켓으로 전달하고, <br>
> "종료" 버튼을 누르면 녹음을 중단하도록 설계했습니다.

if 문 안을 들여다보겠습니다.<br>

클라이언트에서 시작 버튼이 눌린다면<br>
이전에 만들어두었던 큐, 스레드를 모두 비우거나 종료하고<br>
인식할 언어를 전달된 텍스트로부터 받아옵니다.<br>

새로운 스레드를 열어 구글 API를 호출해 STT 변환을 시작합니다.<br>

변환 결과를 처리할 응답 처리용 코루틴 태스크도 시작합니다.(`create_task` 를 사용해 병렬적으로 코루틴이 실행됩니다)<br>

<hr>

**[ run_stt_stream ]**

```python
def run_stt_stream(audio_queue, response_queue, language_code):
    """
    별도 스레드에서 STT 스트리밍을 실행하는 함수

    Args:
        audio_queue (queue.Queue): 오디오 데이터를 받는 큐
        response_queue (asyncio.Queue): 처리 결과를 전달할 큐
        language_code (str): 인식 언어 코드
    """
    try:
        client = SpeechClient()

        # 음성 인식 설정
        recognition_config = cloud_speech_types.RecognitionConfig(
            explicit_decoding_config=cloud_speech_types.ExplicitDecodingConfig(
                encoding=cloud_speech_types.ExplicitDecodingConfig.AudioEncoding.LINEAR16,
                sample_rate_hertz=16000,
                audio_channel_count=1,
            ),
            language_codes=[language_code],
            model="long",
        )

        streaming_features = cloud_speech_types.StreamingRecognitionFeatures(
            interim_results=True,
        )

        streaming_config = cloud_speech_types.StreamingRecognitionConfig(
            config=recognition_config,
            streaming_features=streaming_features,
        )

        config_request = cloud_speech_types.StreamingRecognizeRequest(
            recognizer=f"projects/{PROJECT_ID}/locations/global/recognizers/_",
            streaming_config=streaming_config,
        )

        # 동기 제너레이터 - STT API 호출에 사용
        def request_generator():
            # 설정 요청 먼저 보내기
            yield config_request

            while True:
                # 큐에서 오디오 데이터 가져오기 (blocking)
                chunk = audio_queue.get()

                # None은 스트림 종료 신호
                if chunk is None:
                    break

                # 오디오 데이터 요청 생성 및 전송
                yield cloud_speech_types.StreamingRecognizeRequest(audio=chunk)

        # Google STT API 호출
        responses = client.streaming_recognize(requests=request_generator())

        # 응답 처리 및 결과 큐에 추가
        for response in responses:
            # 결과가 없는 경우 스킵
            if not response.results:
                continue

            # 마지막 결과만 처리 (일반적으로 가장 최신, 가장 정확한 결과)
            result = response.results[-1]

            if result.alternatives:
                transcript = result.alternatives[0].transcript
                is_final = result.is_final
                confidence = result.alternatives[0].confidence if hasattr(result.alternatives[0], 'confidence') else 0.0

                # 메인 스레드에 응답 전달
                def send_response():
                    asyncio.create_task(response_queue.put({
                        "transcript": transcript,
                        "is_final": is_final,
                        "confidence": confidence,
                    }))

                loop.call_soon_threadsafe(send_response)

    except Exception as e:
        logging.error(f"STT 스트리밍 오류: {str(e)}")
        # 오류 정보 전달
        try:
            asyncio.run(response_queue.put({
                "error": str(e)
            }))
        except:
            pass
```

다음은 실제로 구글 API를 호출하는 함수입니다.<br>

이전 글에서 보셨던 것과 크게 다르지 않습니다!<br>
다만 음성을 `audio_queue` 에서 꺼내고, 변환 결과를 `response_queue` 에 저장한다는 것만 다릅니다<br>

`interim_results` 를 `True` 로 설정해두면 중간 결과도 받아볼 수 있습니다.<br>

구글 API 관련 설정을 마친 후에 `reqeust_generator` 를 통해 `audio_queue` 에서 음성 바이트를 가져오고, 음성 데이터를 흘려줍니다.<br>

응답을 비동기적으로 처리하기 위해 main에서 선언해두었던 이벤트 루프를 이용합니다.<br>
이 방식을 이용하면 동기 함수 내부이지만 코루틴을 실행할 수 있게 됩니다.<br>

**[ process_responses.py ]**

```python
async def process_responses(response_queue, websocket):
    """
    STT 응답을 처리하고 WebSocket으로 전송하는 함수
    중간 결과는 'interim' 타입으로, 최종 결과는 'final' 타입으로 전송합니다.

    Args:
        response_queue (asyncio.Queue): STT 결과를 받는 큐
        websocket (WebSocket): 결과를 전송할 WebSocket
    """
    # 현재 상태 관리
    last_interim_text = ""  # 마지막으로 전송한 중간 텍스트
    last_final_text = ""    # 마지막으로 전송한 최종 텍스트

    try:
        while True:
            # 큐에서 응답 가져오기
            response = await response_queue.get()

            # 오류 발생 시
            if "error" in response:
                error_response = {
                    "type": "error",
                    "message": f"음성 인식 오류: {response['error']}"
                }
                await websocket.send_text(json.dumps(error_response, ensure_ascii=False))
                continue

            # 정상 응답 처리
            transcript = response["transcript"].strip()
            is_final = response["is_final"]
            # confidence = response.get("confidence", 0.0)

            # 텍스트가 비어있으면 무시
            if not transcript:
                continue

            # 결과 유형에 따라 처리
            if is_final:
                # 최종 결과가 이전 최종 결과와 다를 경우에만 전송
                if transcript != last_final_text:
                    json_response = {
                        "type": "final",
                        "text": transcript,
                        # "confidence": confidence
                    }
                    await websocket.send_text(json.dumps(json_response, ensure_ascii=False))
                    last_final_text = transcript

                # 중간 결과 초기화
                last_interim_text = ""

            else:
                # 중간 결과가 이전 중간 결과와 다를 경우에만 전송
                if transcript != last_interim_text:
                    json_response = {
                        "type": "interim",
                        "text": transcript,
                        # "confidence": confidence
                    }
                    await websocket.send_text(json.dumps(json_response, ensure_ascii=False))
                    last_interim_text = transcript

    except asyncio.CancelledError:
        # 태스크 취소
        return
    except Exception as e:
        logging.error(f"응답 처리 오류: {str(e)}")
        error_response = {
            "type": "error",
            "message": f"응답 처리 오류: {str(e)}"
        }
        await websocket.send_text(json.dumps(error_response, ensure_ascii=False))
```

`handle_websocket_connection` 에서 호출되었던 응답 처리 태스크입니다.<br>

중복된 텍스트를 전송하지 않기 위해 마지막으로 전송한 중간, 최종 텍스트를 저장할 변수를 선언합니다.<br>

계속해서 응답 큐에서 응답을 가져옵니다.<br>

> await가 있기 때문에 무작정 기다리지 않고, 응답이 오지 않으면 다른 함수에게 실행 권한을 넘깁니다.

응답 큐에 응답이 있다면 웹소켓으로 계속해서 내용을 전달합니다.<br>

중간 결과는 interim, 최종 결과는 final로 type 변수를 지정해 전송합니다.<br>
클라이언트에서 final이 전달 된 순간 번역 API를 호출하기 위해 이렇게 전송하게 되었습니다.<br>

<hr>

## 마치며

휴. 구현하면서 꽤 헷갈리고,,머가 잘 안되고,,했지만,,,<br>

결국 진짜 실시간 STT 변환 기능을 구현했습니다!<br>
번역 기능은 저번 SSE 방식을 그대로 사용하고 있습니다 ㅎ.ㅎ<br>
이전 글을 참고해주세요!<br>

🥵

<hr>
<br>

> 참고 자료

클로드
