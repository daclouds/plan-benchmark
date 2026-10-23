# 검증 기록

2026-10-24(Asia/Seoul) 기준으로 확인했습니다.

- Python 3.12.14, Node.js 24.19.0, Promptfoo 0.124.1.
- `python -m unittest -q`: 11개 테스트 통과.
- Python 단독 mock: 6문제 × A/B/C × 2회 = 36개 생성·채점·집계 확인.
- 실제 Promptfoo + localhost bridge mock: 같은 36개 실행 모두 통과, 오류 0개.
- Promptfoo 결과의 반복 번호, 군별 호출 수, 총점과 ledger 집계를 대조.
- 비용 테스트: 캐시 읽기/쓰기, reasoning 토큰의 중복 계산 방지, 잘린 출력의 사용 비용 보존, 사용량 미반환 시 비용 미상 처리.
- 실패 테스트: 병렬 일부 실패 시 완료된 호출 비용 보존, judge 실패와 품질 0점 분리, 형식·의존성 오류, 치명적 오류가 있는 높은 점수의 불합격 처리.

실제 유료 LLM API는 호출하지 않았습니다. 모델 접근권한, 실제 API 응답 호환성, 실제 출력의 JSON 준수율과 품질은 키를 설정한 뒤 실험으로 확인해야 합니다. mock 점수는 모두 고정된 합성 점수이므로 모델 성능 비교 근거로 사용할 수 없습니다.

기본 Python 직접 연결 방식은 이 Codex 샌드박스가 Node의 자식 프로세스 실행을 제한하여 종단 검증하지 못했습니다. 검증된 `--bridge` 방식은 동일한 Python 생성·채점 함수를 localhost로 연결합니다.

`example-mock/`에는 최종 Promptfoo mock 실행의 집계 보고서와 CSV를 넣었습니다. ZIP에는 패키지 다운로드 캐시, API 키, 실패한 개발 실행, node_modules를 포함하지 않았습니다.
