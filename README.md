# llm-wiki

개인용 LLM 기반 지식 위키를 구축하고 유지하는 Claude Code 스킬.

LLM이 원본 문서를 읽고 구조화된 마크다운 위키를 만들어 최신 상태로 유지합니다. 매 질문마다 처음부터 다시 추론하는 RAG 방식과 달리, 한 번 정리한 지식을 누적해 재사용합니다.

Karpathy의 [llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) 패턴 기반.

## 아키텍처

3개 레이어:

```
raw/          # 원본 문서 (불변, 여기에 파일을 떨어뜨림)
wiki/         # LLM이 생성·관리하는 마크다운 페이지
purpose.md    # 이 위키가 존재하는 이유
CLAUDE.md     # 스키마와 워크플로우
```

`wiki/`는 네임스페이스가 흩어지지 않도록 고정된 버킷으로 나뉩니다:

- `entities/` — 사람, 팀, 조직, 제품
- `concepts/` — 패턴, 방법론, 기술 용어, 이론
- `sources/` — 인제스트된 원본 소스별 요약 페이지
- `synthesis/` — 쿼리에서 도출된 교차 소스 페이지
- `synthesis/daily/` — 하루 단위 다이제스트
- `index.md`, `overview.md`, `log.md` — 카탈로그, 스냅샷, 히스토리

모든 페이지는 frontmatter에 `sources: [raw/...]`를 가집니다. 이 필드가 source-overlap 쿼리 시그널과 delete 캐스케이드를 가능하게 합니다.

## 서브커맨드

`/llm-wiki <subcommand> [args]` 형태로 호출:

| 서브커맨드 | 용도 |
|---|---|
| `build [scenario]` | 현재 디렉토리에 위키 구조 부트스트랩 |
| `ingest [path]` | `raw/` 안의 소스를 읽어 위키에 통합 (경로 생략 시 `raw/` 전체 처리) |
| `query <question>` | 위키 내용으로 답변 (인용 포함) |
| `daily [date]` | 하루치 raw 입력을 다이제스트로 합성 |
| `lint` | 모순·고아 페이지·누락 헬스체크 |
| `delete <path>` | 원본 소스 제거 및 위키 캐스케이드 정리 |

## 빠른 시작

1. 위키를 둘 디렉토리에서 `/llm-wiki build` 실행. 시나리오(research, reading, personal, business, general) 선택 후 `purpose.md` 채우기.
2. `raw/`에 소스 파일을 떨어뜨리기 (여러 개 가능).
3. `/llm-wiki ingest` 실행 — `raw/` 안의 모든 파일을 훑어 위키에 통합합니다. 특정 파일만 처리하려면 `/llm-wiki ingest raw/<file>`. SHA256 해시로 이미 인제스트된 파일은 자동 스킵하며, 각 파일마다 분석 → 3-5개 요점 논의 → 작성 순으로 진행됩니다.
4. `/llm-wiki query <question>`으로 질문. 답변에는 위키 페이지와 원본 파일이 인용됩니다.
5. 하루 끝에 `/llm-wiki daily`로 그 날의 raw를 하나의 다이제스트로 압축.
6. 주기적으로 `/llm-wiki lint`를 돌려 모순·고아 페이지·공백 영역을 점검.

## 설계 원칙

- **사용자가 큐레이션, LLM은 관리.** 소스 선택과 질문은 사람이, 정리·연결은 LLM이.
- **Raw는 불변.** `raw/`는 읽기 전용. `delete`는 위키 페이지만 정리하고 원본 파일은 건드리지 않음.
- **2단계 인제스트.** 먼저 분석 → 요점 논의 → 작성. 이 사이 멈춤이 모순을 잡아냄.
- **모든 주장에 출처.** 모든 위키 클레임은 원본 소스나 다른 위키 페이지로 추적 가능.
- **재유도 대신 누적.** 좋은 답변은 synthesis 페이지로 위키에 되돌려 저장.
- **해시로 중복 회피.** SHA256으로 raw 파일을 체크해 변경되지 않은 소스는 다시 읽지 않음.
- **조기 도구화 금지.** 200페이지 미만 규모에서는 index 파일 + 4가지 관련성 시그널만으로 충분. 벡터 검색은 실제 한계에 부딪힐 때까지 도입하지 않음.

## 쿼리 관련성 (마크다운만 사용)

`query`는 임베딩 없이 4가지 시그널로 후보 페이지를 확장합니다:

| 시그널 | 가중치 |
|---|---|
| Source overlap (`sources[]` 공유) | ×4.0 |
| 직접 `[[wikilink]]` | ×3.0 |
| 공통 이웃 (Adamic-Adar) | ×1.5 |
| 타입 친화도 | ×1.0 |

## 호환성

`wiki/` 디렉토리는 Obsidian 호환 `[[wikilink]]`와 YAML frontmatter를 가진 평문 마크다운입니다. Obsidian에서 열어 그래프 뷰로 보거나, git 저장소로 두고 인제스트마다 커밋해도 됩니다.

## 참고

- 전체 스킬 명세: [`.claude/llm-wiki/SKILL.md`](.claude/llm-wiki/SKILL.md)
- 다른 언어: [English](README_en.md) / [日本語](README_jp.md)
- 원본 gist: [karpathy/llm-wiki.md](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
