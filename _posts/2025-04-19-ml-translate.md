---
title: "생성형 AI 미니 프로젝트(3): Open AI 실시간 응답 구현하기 (feat. SSE)"
categories: [Fast API]
tags: [ML, Fast API, SSE, Open AI API, SKALA, SKALA1기, SK]
---

> Skala 과정에서 생성형 AI 대해 새롭게 알게 되었습니다.<br>
> 생성형 AI 수업 중 진행하게 된 미니 프로젝트 구현 과정을 기술합니다.

[처음 글 보러가기](<https://sermadl.github.io/posts/ml-project(1)/>)

Open AI의 응답을 실시간으로 클라이언트에 전달하고 싶어서<br>
Fast API의 StreaminResponse를 사용한 SSE를 통해 해당 기능을 구현했습니다!<br>

<hr>

## 실시간 번역 기능 구현하기

### openai_router.py

```python
from typing import Union
from fastapi import APIRouter, Header, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.db.database import get_db

from app.services.openai_service import get_streaming_message_from_openai
from app.services.log_script_service import logger
from fastapi.responses import StreamingResponse
from app.dto.message_request import MessageRequest


router = APIRouter()

@router.post(
    "/translate",
    summary="Generate messages using OpenAI (streaming)",
)
async def get_generated_messages_with_header(
    data: MessageRequest,
    x_script_id: Union[str, None] = Header(default=None),
    db: AsyncSession = Depends(get_db)
):
    """
    ## Open AI에 번역 요청
    - Header: "x-script-id" 포함되어야 함!
    - data: MessageRequest
        - lang: 언어 코드 (e.g., "en-US", "ko-KR")
        - message: 번역할 메시지
    """

    await logger(db, data, x_script_id)

    return StreamingResponse(
        get_streaming_message_from_openai(data),
        media_type="text/event-stream"
    )

    .
    .
    .
```

StreamingResponse를 사용해서 SSE를 구현합니다.<br>

<hr>

### openai_service.py

```python
import logging
import os
from fastapi import HTTPException
from langchain_openai import ChatOpenAI
from langchain_core.output_parsers import StrOutputParser
from langchain_core.prompts import PromptTemplate
from langchain_teddynote.messages import stream_response
from app.dto.message_request import MessageRequest

OPENAI_API_KEY=os.getenv("OPENAI_API_KEY")

model = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0.5,
    openai_api_key=OPENAI_API_KEY,
    streaming=True
)

templete='{text}을 {lang}로 번역해주세요. 번역 된 문장만 출력해주세요.'
prompt = PromptTemplate.from_template(templete)
output_parser = StrOutputParser()
chain = prompt | model | output_parser

async def get_streaming_message_from_openai(data: MessageRequest):
    try:
        # 스트리밍 응답 생성
        response = chain.astream({
            "text": data.message,
            "lang": data.lang
        })

        async for token in response:
            content = f"data: {token}\n\n"
            yield content
        yield "data: [DONE]\n\n"
    except Exception as e:
        # 에러 발생 시 스트리밍 방식으로 에러 메시지 반환
        logging.error(f"Error occurred: {str(e)}")
        raise HTTPException(status_code=500, detail=f"OpenAI Error: {str(e)}")
```

수업 시간에 배웠던 `chain` 과 `StrOutputParser` 을 사용해서 프롬프트와 모델을 잇고, 응답을 예쁘게(원하는 정보만) 받을 수 있게 설정해주었습니다.<br>

`streaming` 옵션을 True로 설정해서 응답을 스트리밍 방식으로 하도록 해주고 `astream` 으로 응답을 받아왔습니다.<br>

> `stream` 은 동기 방식으로 스트리밍 응답을 주는 함수이고, `astream` 이 비동기 방식으로 스트리밍 응답을 주는 함수이기 때문에 `astream` 을 사용했습니다!

`aysnc for` 을 사용해서 응답이 올 때마다 SSE로 결과를 전송할 수 있도록 해주었습니다.<br>

<hr>

## 마치며

이전 STT에 비해 굉장히 싱겁게(?) 끝났지만,,<br>
옵션과 함수를 사용하기 위해 `ChatOpenAI` 객체를 뜯어봤습니다,,<br>

쪼금 오래걸렸어용,,<br>

참고하시고 행복한 코딩 생활 하세욥 😀

<hr>
<br>

> 참고 자료

[하나](https://python.langchain.com/api_reference/openai/llms/langchain_openai.llms.base.OpenAI.html)
