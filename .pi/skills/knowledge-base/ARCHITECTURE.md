# Knowledge Base Skill 아키텍처 및 구현 지침

본 문서는 Pi 에이전트의 지식베이스(Knowledge Base) 스킬 확장을 위한 폴더 구조 및 핵심 아키텍처 결정 사항을 정의합니다. 에이전트는 해당 스킬을 구현하고 유지보수할 때 이 구조를 준수해야 합니다.

## 1. 디렉토리 구조 (Directory Structure)

지식베이스 스킬은 터미널에서 독립적으로 실행 가능한 파이썬 스크립트 모음으로 구성됩니다.

```text
.pi/
└── skills/
    └── knowledge-base/
        ├── SKILL.md                 # 스킬 명세서: 에이전트가 읽고 동작 방식을 이해하는 문서
        ├── ARCHITECTURE.md          # 현재 문서 (구조 및 의사결정 기록)
        ├── requirements.txt         # 의존성: sqlite-vec, tree-sitter, llama-cpp-python 등
        ├── models/                  # (선택) 오프라인 GGUF 임베딩 모델 저장 폴더
        └── scripts/                 # 에이전트가 터미널에서 직접 실행할 액션 스크립트
            ├── init_db.py           # [초기화] DB 파일 생성 및 sqlite-vec 테이블 셋업
            ├── create.py            # [C] 인자: <file_path> | 파싱, 임베딩, DB 삽입
            ├── read.py              # [R] 인자: <query> | 쿼리 임베딩 및 하이브리드 검색
            ├── update.py            # [U] 인자: <file_path> | 기존 청크 삭제 후 재생성
            ├── delete.py            # [D] 인자: <file_path> | 특정 파일/경로의 데이터 삭제
            └── core/                # 공통 모듈 (DRY 원칙 준수)
                ├── db_client.py     # SQLite 커넥션 및 sqlite-vec 확장 로드
                ├── chunker.py       # tree-sitter 기반 AST 파싱/청킹 로직
                └── embedder.py      # llama-cpp-python 임베딩 추출 로직
```

## 2. 구성 요소 상세 (Component Details)

*   **`SKILL.md` (제어 타워)**: 에이전트의 진입점입니다. 에이전트는 사용자의 CRUD 요청을 분석한 뒤, `scripts/` 디렉토리 내의 적절한 파이썬 스크립트를 인자와 함께 호출하도록 설계되었습니다.
*   **`scripts/` (액션 모듈)**: 에이전트가 동적으로 코드를 생성하여 실행하는 대신, 사전 정의된 스크립트를 CLI 명령어로 호출합니다. 각 스크립트는 실행 결과를 표준 출력(`stdout`)으로 반환하여 에이전트의 컨텍스트로 제공합니다.
*   **`scripts/core/` (공통 모듈)**: 데이터베이스 커넥션 생성이나 무거운 임베딩 모델 로딩과 같은 반복 작업을 모듈화하여 성능과 유지보수성을 확보합니다.

## 3. 아키텍처 결정 사항 (Architectural Decisions)

구현을 진행하기 전 또는 진행 중에 확정해야 할 주요 설계 의사결정 사항입니다.

1.  **스토리지 격리 수준 (Storage Scope)**
    *   `옵션 A`: 운영체제 전역 공유 (예: `~/.pi/knowledge.db`) - 모든 프로젝트가 하나의 지식베이스를 공유.
    *   `옵션 B`: 프로젝트별 독립 격리 (예: `./.pi/knowledge.db`) - 각 프로젝트 워크스페이스마다 별도의 벡터 DB 유지.
2.  **CRUD 실행 인터페이스 (Execution Interface)**
    *   에이전트가 파이썬 코드를 즉석에서 작성하여 실행하는 방식을 배제하고, `scripts/` 폴더 내의 사전 구현된 파이썬 스크립트에 파라미터를 넘겨 실행하는 방식을 표준으로 채택합니다.
3.  **임베딩 모델 관리 (Embedding Model Management)**
    *   `옵션 A`: 사용자가 사전에 모델 파일(예: Nomic GGUF)을 `models/` 폴더에 다운로드해 두는 것을 전제로 동작.
    *   `옵션 B`: 에이전트가 `init_db.py` 또는 로드 시점에 모델 파일의 존재 여부를 확인하고, 없을 경우 Hugging Face 등에서 자동 다운로드하는 로직 포함.
