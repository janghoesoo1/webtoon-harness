---
name: webtoon-orchestrator
description: "Codex에서 웹툰 제작 하네스를 실행하는 진입점. 기존 Claude Code용 27개 에이전트와 6개 스킬 정의를 읽어 Codex 작업 루프로 변환하고, 트렌드 조사부터 시나리오, 레퍼런스, 50+ 패널 렌더, 검증, 세로 스크롤 RELEASE까지 수행한다. 트리거: 웹툰 만들어, 웹툰 한 화 제작, 다음 화 만들어, 패널 다시 그려, 반전 더 강하게, 웹툰 하네스 실행."
---

# Webtoon Orchestrator for Codex

이 스킬은 Claude Code용 하네스를 Codex에서 실행하기 위한 어댑터입니다. 원본 역할과 방법론을 새로 복제하지 않고, 저장소의 `.claude/agents`와 `.claude/skills`를 읽어 같은 산출물 계약으로 진행합니다.

## 먼저 읽을 파일

요청 유형에 따라 필요한 파일만 읽습니다.

- 전체 회차 제작: `.claude/skills/webtoon-orchestrator/SKILL.md` 전체와 각 Phase의 관련 agent 파일.
- 리서치: `trend-scout`, `platform-ranker`, `audience-analyst`, `hook-analyst`, `trend-synthesizer`.
- 시나리오: `concept-architect`, `worldbuilder`, `character-designer`, `series-plotter`, `twist-master`, `tension-engineer`, `episode-outliner`, `dialogue-writer`, `script-editor`.
- 비주얼: `art-director`, `ref-sheet-artist`, `panel-director`, `letterer`, `prompt-smith`, `panel-artist-a`, `panel-artist-b`, `panel-artist-c`, `panel-validator`.
- 조립검수: `episode-compositor`, `quality-reviewer`, `continuity-manager`, `showrunner`.

## Codex 변환 규칙

Claude Code 원본의 `TeamCreate`, `TaskCreate`, `SendMessage`, custom agent 호출은 Codex에서 다음 방식으로 해석합니다.

- 팀 생성은 해당 Phase의 역할 파일을 읽고 산출물 목록을 확정하는 것으로 대체합니다.
- 역할별 작업은 `_workspace/` 안의 명시 파일 작성으로 대체합니다.
- 역할 간 메시지는 다음 역할 산출물에 "입력 요약"과 "결정 근거"를 남기는 방식으로 대체합니다.
- 검증 담당 역할은 별도 검증 문서에 PASS, FIX, REDO, ACCEPT-FLAG를 기록합니다.
- 후속 요청은 기존 산출물을 먼저 읽고 영향받는 단계만 다시 실행합니다.

## 표준 Phase

1. Phase 0, 컨텍스트 확인.
   - `_workspace/` 존재 여부와 사용자 요청을 확인합니다.
   - 새 기획이면 기존 작업을 지우지 말고 별도 백업 또는 새 회차 경로를 사용합니다.

2. Phase 1, 준비.
   - `_workspace/00_input/brief.md`에 사용자 요청, 회차 번호, 제약을 기록합니다.
   - 필요한 하위 디렉토리를 만듭니다.

3. Phase 2, 리서치.
   - 4개 관점 조사 결과를 별도 파일로 쓰고 `trend-brief.md`로 종합합니다.
   - 최신 플랫폼 트렌드가 필요하면 웹 검색으로 검증합니다.

4. Phase 3, 시나리오.
   - 콘셉트, 세계관, 캐릭터, 시리즈 아크, 반전 계획, 긴장 곡선, 비트시트, 대본, 최종 대본을 순서대로 작성합니다.
   - `ep{NN}_script_final.md`는 대사 위주이고 50개 이상 패널로 분해 가능해야 합니다.

5. Phase 4, 비주얼.
   - `style-bible.md`와 `character-sheets.md`를 먼저 만듭니다.
   - 캐릭터 레퍼런스 시트를 먼저 렌더하고 `refs/INDEX.md`를 확정합니다.
   - 샷리스트, 레터링 명세, 패널 프롬프트를 작성합니다.
   - codex image generation은 동시에 5개 이하로 실행합니다.
   - `panel-validator` 기준으로 패널별 검증과 필요한 재생성을 수행합니다.

6. Phase 5, 조립검수.
   - 말풍선이 베이크된 패널만 사용해 오버레이 없는 세로 스크롤 `index.html`을 만듭니다.
   - QA와 continuity를 갱신하고, showrunner 기준으로 `RELEASE/ep{NN}/`를 패키징합니다.

7. Phase 6, 마무리.
   - `RELEASE/ep{NN}/index.html`을 열어 실제로 확인합니다.
   - 최종 응답에 회차 제목, 패널 수, 뷰어 경로, 검증 결과, 남은 리스크를 보고합니다.

## 후속 작업 라우팅

- "다음 화"는 기존 세계관, 스타일, 레퍼런스, continuity를 재사용하고 `03_episode` 이후를 새 회차로 생성합니다.
- "반전 더 강하게"는 시나리오 단계부터 영향받는 하위 단계만 다시 실행합니다.
- "패널 N번 다시"는 해당 패널의 프롬프트, 렌더, 검증, 조립만 다시 실행합니다.
- "대사 톤 수정"은 대본과 레터링, 영향을 받는 패널만 다시 실행합니다.

## 완료 기준

- 요청한 범위의 산출물이 `_workspace/`와 `RELEASE/`에 존재합니다.
- 렌더 이미지가 0바이트나 손상 파일이 아닙니다.
- 서로 다른 패널의 md5 중복이 없습니다.
- 검증 문서에 패널별 판정이 남아 있습니다.
- 최종 HTML을 실제로 열어 세로 스크롤 웹툰으로 확인했습니다.
