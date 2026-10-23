# Plan 품질 A·B·C 실험

저비용 모델 여러 개가 고비용 모델 하나의 **최종 실행 계획 품질**을 따라잡는지 비교하는 Promptfoo + Python 실험 키트입니다.

| 군 | 계획 생성 | 생성 호출 수 / 문제 / 반복 |
|---|---|---:|
| A | high 모델 단독 | 1 |
| B | low 모델 단독 | 1 |
| C | low 모델의 독립 후보 3개를 병렬 생성 → low 모델로 통합 | 4 |

C는 코드가 정해진 호출을 조율합니다. 고비용 메인 모델의 숨은 조율·통합 호출은 없습니다. Codex 앱의 대화형 서브에이전트를 직접 실행하는 방식이 아니라, **같은 병렬 후보 생성·통합 구조를 재현 가능한 API 호출로 구현**했습니다. 도구 사용·에이전트 메모리·자율 위임 성능은 이 실험에 포함되지 않습니다.

## 빠른 실행 — PowerShell

필수: Python 3.11 이상, Node.js 22.22 이상(24 권장), npx 또는 설치된 Promptfoo. Python 코드는 표준 라이브러리만 사용하므로 pip 설치가 필요 없습니다.

이 README가 있는 폴더에서 실행하세요.

```powershell
# 선택 사항: 검증된 Promptfoo 버전을 프로젝트에 설치
npm install

# API 없이 Python 파이프라인 점검
python run.py --mock --engine python --limit 2 --repeat 1

# Promptfoo 설치와 연결 점검: 첫 실행에는 패키지 다운로드가 필요할 수 있음
python run.py --mock --limit 2 --repeat 1

# Node의 Python 자식 프로세스 실행이 제한된 환경에서는 로컬 연결 방식 사용
python run.py --mock --bridge --limit 2 --repeat 1
```

실제 실험 전 `experiment.toml`에서 모델 접근 가능 여부와 단가를 확인하세요. API 키는 현재 PowerShell 세션의 환경변수에 설정합니다. 파일이나 명령 기록에 키를 남기지 않으려면 다음처럼 입력하세요.

```powershell
$planKey = Read-Host 'OpenAI API key' -AsSecureString
$env:OPENAI_API_KEY = [System.Net.NetworkCredential]::new('', $planKey).Password

# 6개 예제 × 3개 군 × 1회: 생성 36회 + 정상 출력 채점 최대 18회
python run.py --repeat 1

# 각 문제를 3회 반복: 생성 108회 + 정상 출력 채점 최대 54회
python run.py --repeat 3

# 로컬 연결 방식의 실제 실험: 같은 생성·채점 로직, 같은 비용 집계
python run.py --bridge --repeat 3

# Promptfoo가 없어도 동일 생성·채점·비용 집계를 수행할 수 있음
python run.py --engine python --repeat 3
```

`run.py`는 매번 새 `runs/real_...` 또는 `runs/mock_...` 폴더를 만듭니다. Promptfoo 캐시는 비활성화합니다. 표준 Promptfoo 설정은 `promptfooconfig.yaml`이며, 실행기는 선택한 케이스와 절대 경로가 들어간 설정을 각 실행 폴더에 생성합니다. 키가 없으면 실제 실행 전에 종료합니다. 실제 API 호출은 과금됩니다. mock 결과는 항상 mock으로 표시되며 실제 비용은 0입니다.

`package.json`은 연결 검증한 Promptfoo 0.124.1을 고정합니다. 실행기는 프로젝트의 `node_modules` 설치를 우선 사용하고, 없으면 PATH의 `promptfoo`, 그다음 `npx --yes promptfoo@0.124.1`(또는 pnpm dlx)을 사용합니다. 실제 사용 버전은 실행 폴더의 `promptfoo-version.txt`에 기록됩니다. `npm install`이 생성한 lockfile도 보관하면 다음 실행에서 `npm ci`로 의존성을 고정할 수 있습니다.

`--bridge`는 Python 실행기 안에 임의 포트의 localhost HTTP 서버를 잠시 열고 Promptfoo의 JavaScript 어댑터를 연결합니다. 서버는 127.0.0.1에만 바인딩하며 실행마다 임의 토큰을 사용하고 실험 후 닫습니다. 생성·채점 모델 호출과 비용 계산은 기본 Python provider 방식과 동일합니다. 이 키트의 Promptfoo 0.124.1 연결 검증은 `--bridge` mock 모드로 수행했습니다. 기본 Python 직접 연결은 이 Codex 샌드박스의 Node 자식 프로세스 실행 제한으로 종단 검증하지 못했습니다. 일반 환경에서는 기본 모드부터 사용하고 Python 인식/실행 오류가 나면 `--bridge`를 사용하세요.

## 모델·엔드포인트 교체

`experiment.toml`의 세 역할을 수정하세요.

```toml
[models.low]
model = "원하는-모델-ID"
base_url = "https://provider.example/v1"
api_key_env = "OTHER_API_KEY"
input_per_million = 0.10
cached_input_per_million = 0.01
cache_write_per_million = 0.125
output_per_million = 0.50
max_completion_tokens = 6000
reasoning_effort = "medium"
```

위 단가는 설정 예시이며 다른 모델에 그대로 적용하면 안 됩니다. `high`는 A, `low`는 B와 C 전체, `judge`는 모든 군에 동일하게 적용합니다. 최초 설정은 `gpt-6.1-sol` / `gpt-6-luna` / `gpt-6.1-sol`입니다. 계정의 사용 가능한 API 모델로 교체하세요. Codex 앱 구독의 모델 접근권한과 별개입니다.

현재 어댑터는 **OpenAI 호환 `/chat/completions` 텍스트 API**를 사용합니다. Anthropic/Gemini 네이티브 API를 직접 지원한다는 뜻은 아닙니다. 호환 엔드포인트는 `messages`, `max_completion_tokens`, `usage.prompt_tokens`, `usage.completion_tokens`를 지원해야 합니다. 지원하지 않는 모델에서는 `reasoning_effort`를 삭제하세요. `temperature`, `json_mode = true`는 해당 모델이 지원할 때만 추가하세요. 토큰 제한은 reasoning 토큰도 포함하므로 JSON 출력이 잘리면 늘리세요. 자동 토큰 제한 확대나 재시도는 없습니다.

`workers`로 C의 후보 수를 바꾸면 C의 생성 호출 수는 `workers + 1`이 됩니다. 후보 수를 바꾼 구성은 별도 실험으로 취급하세요. 공통 `max_plan_chars`, `max_steps`가 최종 결과의 분량을 제한합니다. 초과 출력은 몰래 잘라내지 않고 형식 실패로 기록합니다.

## 비용 계산

API가 반환한 토큰 사용량과 설정 단가로 USD를 추정합니다.

```text
호출 비용 = [(입력 - 캐시 읽기 - 캐시 쓰기) × 입력 단가
             + 캐시 읽기 × 캐시 입력 단가
             + 캐시 쓰기 × 캐시 쓰기 단가
             + 전체 출력 토큰 × 출력 단가] / 1,000,000

생성 비용 = A/B의 단독 호출 또는 C의 후보 전체 + 통합 호출
채점 비용 = judge 호출 비용
실험 전체 비용 = 생성 비용 + 채점 비용
```

`completion_tokens`에 포함된 reasoning 토큰을 다시 더하지 않습니다. 캐시 쓰기는 API의 `prompt_tokens_details.cache_write_tokens`가 보고될 때 분리 계산합니다. 이 필드를 보고하지 않는 호환 서비스의 추가 요금은 반영할 수 없습니다. 실제 청구액이 아니라 **설정 단가에 근거한 추정치**입니다. 기본값은 공식 모델 페이지의 2026-10-24 표준·짧은 컨텍스트 텍스트 단가를 사용합니다. 장문 컨텍스트 할증, 지역 할증, Fast/Batch/Flex, 외부 도구·인프라·구독 비용은 자동 계산하지 않습니다. 사용 조건에 맞게 단가를 변경하세요.

반환된 사용량이 있는 호출은 JSON 파싱 실패나 출력 잘림이 있어도 비용을 보존합니다. 병렬 후보 하나가 실패해도 나머지 호출이 끝날 때까지 기다려 이미 사용한 비용을 기록합니다. 사용량이 없는 타임아웃·HTTP 실패는 무료로 단정하지 않고 `unknown_cost_calls`로 표시합니다. 이 값이 0보다 크면 표의 전체 비용은 알려진 비용의 하한입니다. 공급자의 청구 내역과 대조하세요.

## 평가 방식

모든 군은 동일 과제와 동일 출력 형식을 받습니다. 숨겨진 `eval_reference`는 생성/통합 모델에 전달되지 않습니다. 심사 모델에는 원문 과제, 과제별 참조 체크리스트, 최종 Plan만 전달합니다. 실험군 이름·가격·후보 계획은 전달하지 않습니다. 출력 내용 자체로 모델 특성이 드러날 가능성까지 제거한 완전한 익명화는 아닙니다.

코드는 먼저 JSON 형식, 필수 항목, 중복 단계, 알 수 없는/자기/미래 의존성, 분량 제한을 검사합니다. 실패하면 **0점**이며 추가 채점 API를 호출하지 않습니다. 이 검사는 의미적 타당성의 증명이 아닙니다.

정상 형식의 Plan은 동일 judge 모델이 항목별 0~4점과 근거를 반환합니다.

| 항목 | 가중치 |
|---|---:|
| 요구사항 충족 | 30 |
| 실행 가능성 | 25 |
| 순서·의존성 | 20 |
| 검증 방법 | 15 |
| 실패 대응 | 10 |

0=없음/근본 오류, 1=중대한 누락, 2=실질적 수정 필요, 3=경미한 누락, 4=구체적·일관된 충족입니다. 항목 점수를 4로 나눈 뒤 가중합하여 **100점 척도**로 보고합니다. 합격은 기본 75점 이상이며 치명적 오류가 없어야 합니다. 치명적 오류가 있으면 점수가 높아도 불합격입니다.

생성 API 실패와 judge 실패는 품질 0점으로 섞지 않고 별도로 기록하며 평균 품질에서 제외합니다. 따라서 평균만 보지 말고 시도 수, 채점 수, 오류 수도 함께 확인하세요. judge는 절대적인 정답이 아닙니다. A와 judge의 기본 모델이 같아 발생할 수 있는 선호 편향을 점검하려면 judge 교체 실험과 사람 평가 표본을 추가하세요.

## 결과 파일

| 파일 | 내용 |
|---|---|
| `summary.md` | 군별 Plan 평균, 합격률, 생성·채점·전체 비용 |
| `summary.json` | 오류·치명적 오류율·지연시간·누락 실행·차이의 신뢰구간 등 전체 집계 |
| `samples.csv` | 개별 문제/반복/군의 점수와 비용; Excel에서 열 수 있는 UTF-8 BOM |
| `records/*.json` | 최종 Plan, C 후보, 항목별 근거, API별 모델/사용량/단가/비용/실패 기록 |
| `manifest.json` | 모델·단가 설정, 데이터셋 및 Python 코드 해시; API 키는 저장하지 않음 |
| `cases.json` | 해당 실행에 사용한 과제와 평가 체크리스트 |
| `promptfoo-results.json` | Promptfoo로 실행한 경우의 UI/평가 결과 원본 |

Promptfoo의 provider `cost`에는 **생성 비용만** 넣습니다. Python 채점기가 사용하는 비용까지 합산한 실제 실험 비교는 반드시 `summary.*`를 보세요. 실행 폴더에는 과제·계획 원문이 저장됩니다.

```powershell
python summarize.py 'runs/실행폴더이름'
# Promptfoo 기록 저장 위치를 해당 실행 폴더로 맞춘 뒤 열기
$env:PROMPTFOO_CONFIG_DIR = (Resolve-Path 'runs/실행폴더이름/promptfoo-home').Path
npx promptfoo@0.124.1 view
python -m unittest -v
```

## 실험 해석과 데이터 확장

예제 6개는 작동 확인용 소규모 데이터셋입니다. 본 실험에서는 실제 사용 분야의 독립된 과제를 충분히 추가하고 사람이 체크리스트를 검토하세요. `cases.json`의 항목 형식을 복사해 `task_id`, `task`, `eval_reference`를 추가하면 됩니다. task_id는 유일해야 합니다. 채점 기준을 조정하는 개발용 과제와 최종 비교용 과제는 분리하세요.

보고서는 문제별 반복 평균을 먼저 계산한 뒤 A와 B/C의 차이를 구하고, 문제 단위 부트스트랩 95% 구간을 계산합니다. 반복을 독립 문제처럼 취급하지 않습니다. 예정된 반복이 모두 채점된 문제만 해당 쌍 비교에 포함합니다. 실패 때문에 제외되는 문제가 있으면 선택 편향이 생길 수 있으므로 오류를 해결하거나 별도 실패율로 보고하세요.

`noninferiority_margin = 5`는 사전에 정할 허용 품질 차이입니다. C-A 구간의 하한이 -5보다 높으면 이 데이터에서 품질 비열등성의 탐색적 신호로 표시합니다. mock이나 소수 과제의 구간으로 성능 결론을 내리면 안 됩니다. 품질 차이, 치명적 오류율, 실패율, 생성 비용을 함께 검토해야 저렴하게 따라잡았다고 주장할 수 있습니다.

현재 구성은 **정해진 호출 구조의 달성 품질과 비용**을 측정하며 동일 달러 예산을 강제하지 않습니다. 예산 동등 비교가 필요하면 후보 수·토큰 한도를 별도 개발셋에서 정하고 고정한 다음 최종셋을 평가하세요. 마지막에 좋은 결과만 골라 보고하면 안 됩니다. 최종 작업을 실행해 목표 달성 여부까지 검증하는 실험도 별도로 추가할 수 있습니다.

Python 단독 실행은 전체 군/문제/반복 순서를 seed로 섞고, Promptfoo 실행은 provider 순서와 문제 순서를 섞습니다. 모델 샘플링의 완전한 결정성은 보장하지 않습니다. 장기 실험은 접근 가능한 모델 스냅샷과 Promptfoo 버전을 고정하세요.

## 공식 참고 문서

- [Promptfoo Python provider](https://www.promptfoo.dev/docs/providers/python/)
- [Promptfoo Python assertions](https://www.promptfoo.dev/docs/configuration/expected-outputs/python/)
- [Promptfoo 결과 내보내기](https://www.promptfoo.dev/docs/configuration/outputs/)
- [OpenAI Chat Completions API](https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create)
- [GPT-6.1 Sol 모델·단가](https://developers.openai.com/api/docs/models/gpt-6.1-sol)
- [GPT-6 Luna 모델·단가](https://developers.openai.com/api/docs/models/gpt-6-luna)
