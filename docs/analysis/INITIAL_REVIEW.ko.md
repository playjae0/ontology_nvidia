# NVIDIA Auto-Ontology 초기 코드 분석

## 가져온 범위

- 원본: https://github.com/NVIDIA/auto-ontology
- 기준 커밋: `c447003f72a822301e927196a4ecb5b597c308d5`
- 원본 이력 없이 소스 스냅샷을 복사했다. 라이선스와 서드파티 고지는 유지한다.
- 이 문서는 코드와 구성 파일을 읽은 초기 분석이다. 실제 데이터 연결과 모델 응답 품질을 검증한 결과는 아니다.

## 무엇을 만드는 코드인가

관계형 DB 위에 업무 용어, 컬럼 의미, 관계, SQL 속성을 담는 의미 계층을 만들고 자연어 질문에 SQL과 결과를 제공하는 전체 애플리케이션이다. 단순한 온톨로지 파일 생성기보다 데이터 카탈로그·의미 계층·질의 에이전트·관리 UI를 함께 제공하는 플랫폼에 가깝다. 저장소는 Postgres이며 pgvector로 벡터 검색을 지원한다. 별도 RDF 트리플스토어나 OWL 추론기를 기본 구성으로 가정해서는 안 된다.

## 구성과 읽을 위치

| 영역 | 위치 | 역할 |
| --- | --- | --- |
| 웹 UI·인증·게이트웨이 | `frontend/` | Next.js 16, React 19, Better Auth, Prisma |
| API 서버 | `auto_ontology/server/__main__.py` | FastAPI 앱 팩토리 `create_app` |
| DB 연결 | `auto_ontology/connectors/` | DB별 메타데이터와 쿼리 접근 |
| 카탈로그 | `auto_ontology/catalog/` | 메타데이터 추출·정규화·저장 |
| 수집 워커 | `auto_ontology/ingestion_service/` | 수집 작업과 상태 관리 |
| 의미 계층 생성 | `auto_ontology/semantic/compile.py`, `pipeline.py`, `visit_enter.py` | 테이블 처리, 용어 추출, 의미 관계, 임베딩 |
| 자연어 질의 | `auto_ontology/retrieval/text_to_sql/main.py`, `text_to_sql_graph.py` | LangGraph 기반 질의 처리와 검증·재시도 경로 |
| 데이터 계층 | `auto_ontology/dal/schema.py`, `session.py` | SQLAlchemy Core 스키마·연결 관리 |
| 스키마 마이그레이션 | `alembic/` | Postgres 스키마 버전 관리 |
| MCP | `mcp/` | 별도 Python 패키지, 질의·의미 계층 읽기 |

## 처리 흐름

1. 커넥터가 대상 DB의 테이블·컬럼·키 등 메타데이터를 읽는다.
2. 카탈로그와 수집 서비스가 정보를 저장하고 검색에 필요한 데이터를 준비한다.
3. Semantic compilation이 테이블을 처리하며 업무 용어·컬럼 의미·관계를 생성한다. `pipeline.py`는 병렬 처리와 순서 있는 커밋을 조율한다.
4. 질의 파이프라인이 질문과 의미 계층에서 관련 문맥을 찾고 SQL 생성·검증·실행 경로를 거친다.
5. UI 또는 MCP가 SQL·결과·관련 문맥을 사용자나 에이전트에 제공한다.

자동 생성된 용어와 관계가 업무상 정답이라는 보장은 없다. 초기 평가는 알려진 질문과 정답 SQL을 가진 작은 데이터셋으로 진행하는 편이 좋다.

## 실행에 필요한 것

- Python `>=3.12,<3.14`, `uv`; 프런트엔드는 pnpm 기반.
- Docker Compose 또는 공식 Helm 배포 경로. 상세 절차는 루트 `README.md`와 `DEPLOYMENT.md`가 기준이다.
- Postgres + pgvector, 분석 대상 DB, 모델 엔드포인트와 API 키.
- 기본 모델 구성은 NVIDIA NIM의 추론·임베딩·리랭킹이다. `.env.example`의 역할별 설정과 기본값 상속을 확인해야 한다.
- 실제 `.env`와 비밀번호·API 키는 Git에 올리지 않는다.
- README의 샘플 데이터 실습은 별도 `NVIDIA/auto-ontology-eval` 저장소와 시드 작업도 필요하다.
- MCP는 이미 실행 중인 Auto-Ontology에 연결한다. MCP만 실행한다고 전체 플랫폼이 설치되지는 않는다. MCP 쓰기는 지원하지 않으며 변경은 앱 API 경로를 따른다.

## 이 저장소에서 주의할 지점

- 원본 GitHub Actions에는 NVIDIA의 릴리스·스테이징·NVSkills 연동도 포함돼 있다. 개인 레포용 CI와 원본 조직 전용 배포 흐름은 별도로 검토할 필요가 있다.
- 모델 API 호출 비용과 데이터 전달 범위는 실제 실행 전에 사용할 환경에 맞춰 정한다. 이번 작업에서는 모델 호출이나 배포를 하지 않았다.
- Kumo 선택 기능의 바이너리 휠이 `vendor/wheels/`에 포함된다. 플랫폼·Python 버전 및 관련 라이선스를 함께 확인한다.
- 로컬 의존성 설치 시 `twofish` 빌드가 기본 `clang` 부재로 실패했다. 설치된 GCC를 `CC=gcc CXX=g++`로 지정해 재시도했다.

## 다음 분석 순서

1. `dal/schema.py`에서 용어·속성·관계가 어떤 데이터 모델로 저장되는지 정리한다.
2. `semantic/visit_enter.py`와 프롬프트에서 LLM이 만드는 값과 결정적 처리로 만드는 값을 구분한다.
3. 알려진 질문 하나를 `text_to_sql_graph.py`에서 따라가며 검색·생성·검증·재시도를 추적한다.
4. 작은 샘플 DB로 수집 → 의미 계층 생성 → 질문 → SQL·행 검증까지 실행한다.
5. 기존 온톨로지 플랫폼과 데이터 모델, 관리 API, MCP 연동 범위를 비교한다.
