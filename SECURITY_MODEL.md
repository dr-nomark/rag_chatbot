# rag_chatbot 보안 모델

이 문서는 `rag_chatbot`의 위협 모델입니다. 보안 스캔은 이 파일을 출발점으로 삼습니다. 배포 방식이나 감수하는 위험이 바뀌면 이 문서도 함께 고쳐 주십시오.

> 아래 내용 가운데 배포 형태, 문서 민감도, 공격자 범위는 코드만 보고 세운 가정이며 아직 확정되지 않았습니다(8장 참고). 레포에는 인프라나 배포 설정이 없습니다.

## 1. 시스템 개요와 신뢰 경계

단일 파일 Streamlit 앱(`demo.py`)입니다. 사용 흐름은 다음과 같습니다.

1. 사이드바에서 OpenAI API 키를 입력합니다(`st.sidebar.text_input`). 키는 `os.environ["OPENAI_API_KEY"]`에 저장됩니다.
2. PDF 또는 HWP 파일을 업로드합니다(`st.file_uploader`, `type=['hwp','pdf']`).
3. 서버가 텍스트를 추출합니다. PDF는 `get_pdf_text`(pdfminer)로, HWP는 `get_hwp_text`(olefile + zlib + struct 직접 파싱)로 처리합니다. 추출한 텍스트를 1000자 단위로 나눈 뒤 OpenAI 임베딩으로 메모리 안에 FAISS 인덱스를 만듭니다.
4. 채팅에 "요약"을 입력하면 문서 전체를 gpt-3.5-turbo로 요약합니다. 그 밖의 질문에는 가장 비슷한 청크 3개를 프롬프트에 넣어 답합니다. 응답은 `st.markdown`으로 스트리밍됩니다.

신뢰 경계:

- **브라우저 ↔ Streamlit 서버**: 업로드 파일, 채팅 입력, API 키는 모두 신뢰할 수 없는 입력입니다. 인증은 없습니다.
- **Streamlit 서버 ↔ OpenAI API**: 키를 사용해 문서 내용과 질문을 OpenAI로 보냅니다.
- **세션 사이의 경계**: `st.session_state`는 세션마다 분리되지만, `os.environ`은 프로세스 안의 모든 세션이 함께 씁니다.

## 2. 컴포넌트

| 이름 | 범위 |
|---|---|
| **Web UI** | `demo.py`의 Streamlit 페이지 코드(키 입력, 파일 업로드, 채팅 렌더링, `StreamHandler`) |
| **Document Parsing** | `demo.py`의 `get_hwp_text`, `get_pdf_text`, `process_uploaded_file` |
| **RAG / LLM** | `demo.py`의 `generate_response`, `generate_summarize`, FAISS·임베딩 호출, 프롬프트 구성 |
| **Secrets Handling** | `demo.py`의 OpenAI 키 처리(`os.environ`에 저장) |
| **Dependencies** | `requirements.txt`(langchain 0.0.346, streamlit 1.29.0, aiohttp 3.9.1, Pillow 10.1.0, pdfminer.six 20221105 등, 2023년 말 버전으로 고정) |

## 3. 진입점과 접근 주체

- **웹 페이지 전체**: 앱에 접속할 수 있는 사람이면 누구나 쓸 수 있습니다(인증 없음).
- **파일 업로드**: 임의의 PDF/HWP를 올릴 수 있습니다. 파서는 MIME 타입만 보고 고릅니다(`application/octet-stream`이면 HWP 파서로 보냅니다).
- **채팅 입력**: 임의의 문자열을 입력할 수 있습니다. 입력은 문서 청크와 함께 LLM으로 전달됩니다.
- **API 키 입력**: 10자를 넘으면 프로세스 환경변수에 기록됩니다.

## 4. 자산과 민감도 (가정)

- **OpenAI API 키**: 높음. 다른 사람이 쓰면 키 주인에게 요금이 청구됩니다.
- **업로드 문서 내용**: 기밀일 수 있다고 가정합니다.
- **서버 가용성**: 중간(데모 앱으로 가정).

## 5. 공격자 프로필 (가정)

- **앱에 접속할 수 있는 익명 방문자**: 악성 파일을 올리거나, 다른 사용자의 키나 문서를 노립니다.
- **악성 문서 작성자**: 문서 안에 프롬프트 인젝션을 심습니다. 이 경우 문서를 올린 사용자가 피해자가 될 수 있습니다.
- **공급망 / 공개 CVE 공격자**: 고정된 오래된 의존성의 알려진 취약점을 노립니다.

## 6. 기존 통제

- 업로더가 확장자를 `hwp`/`pdf`로 제한합니다.
- HWP 파일은 OLE 구조(`FileHeader`, `HwpSummaryInformation`)만 최소한으로 검사합니다.
- 키 입력 필드는 `type="password"`입니다.
- 대화 기록과 벡터스토어는 세션별 `st.session_state`에 저장됩니다.
- 크기 제한(Streamlit 기본 업로드 한도 200MB는 예외), 인증, 레이트 리밋, 출력 정제는 코드에서 확인되지 않았습니다.

## 7. 범위 밖 / 감수하는 위험

- 아직 지정된 항목이 없습니다.

## 8. 미해결 질문

- 배포 위치와 접근 범위는 어디까지입니까? (공개 / 사내 / 로컬)
- 사용자 개인 키를 씁니까, 공용 키를 씁니까?
- 업로드되는 문서는 얼마나 민감합니까?
- 서비스 거부, 프롬프트 인젝션, 의존성 CVE, 비용 남용 중 감수할 위험이 있습니까?
