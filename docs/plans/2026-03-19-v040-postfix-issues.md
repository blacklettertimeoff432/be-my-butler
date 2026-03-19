# BMB v0.4.0 Post-Fix Issues

> **작업 완료 후 이 파일을 삭제하세요**: `rm docs/plans/2026-03-19-v040-postfix-issues.md && git add -A && git commit -m "chore: remove resolved postfix issues doc"`

Generated: 2026-03-19
Source: v0.4.0 integrity + security audit

---

## Pre-existing Bugs (v0.4.0에서 발견, v0.3.x부터 존재)

### 1. Architect timeout 변수 오류
- **위치**: `skills/bmb/bmb.md` Step 4, line ~380
- **문제**: Architect는 Claude 에이전트인데 `$CROSS_TIMEOUT` 사용 → `$CLAUDE_TIMEOUT`이어야 함
- **영향**: Monitor에 잘못된 타임아웃 전달 → Consultant에 틀린 남은시간 안내
- **수정**: `"timeout_sec":$CROSS_TIMEOUT` → `"timeout_sec":$CLAUDE_TIMEOUT`

### 2. Architect agent Write tool 누락
- **위치**: `agents/bmb-architect.md:5`
- **문제**: frontmatter `tools: Read, Glob, Grep, Bash, Task` — Write 없음
- **실제 동작**: 프로세스에서 councils/, handoffs/ 에 파일 생성 (Write 필요)
- **수정**: `tools: Read, Write, Glob, Grep, Bash, Task`

### 3. Telegram token URL 노출
- **위치**: `skills/bmb/bmb.md` TELEGRAM PROTOCOL 섹션, line ~82
- **문제**: `curl "https://api.telegram.org/bot${BMB_TG_TOKEN}/sendMessage"` — 토큰이 URL에 포함되어 `ps aux`에 노출
- **수정**: POST body로 전달하거나 환경변수 경고 주석 추가
- **심각도**: 싱글유저 환경에서는 낮음, 멀티유저에서는 중

## v0.4.0 미완성 항목 (의도적 — 다음 단계)

### 4. SESSION_MODE 실제 파이프라인 라우팅 미구현
- **위치**: `skills/bmb/bmb.md` Step 1
- **현황**: SESSION_MODE 프레임워크(변수, 주석 가이드, env 파일) 도입 완료. 실제 분기 로직(sub 모드 스킵, consolidation 스킵)은 미구현
- **다음 단계**: Step별 `if [ "$SESSION_MODE" = "sub" ]` 분기 추가
- **참고**: 설계 문서 `docs/plans/2026-03-19-bmb-v040-design.md` Feature #5 참조

### 5. Monitor config 문서화 누락
- **위치**: `docs/configuration.md`
- **문제**: `monitor.enabled`, `monitor.interval`, `monitor.idle_stall_sec` 키가 bmb.md에서 참조되지만 configuration.md에 미문서화
- **수정**: configuration.md에 monitor 섹션 추가 + 기본값 명시

## 권장 수정 순서

1. **#1 + #2** (즉시) — Architect 타임아웃 + Write tool → 1분 수정
2. **#5** (즉시) — Monitor config 문서화 → 5분 수정
3. **#3** (선택) — Telegram token → 보안 강화
4. **#4** (다음 버전) — SESSION_MODE 라우팅 → v0.4.1 scope
