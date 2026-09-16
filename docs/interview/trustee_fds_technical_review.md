# trustee-fds 학습 및 기술면접 준비 가이드

> 이 문서는 저장소 코드를 읽고 정리한 학습용 자료다. 프로그램 동작을 바꾸지 않으며,
> 법률 의견이 아니다. 코드 위치(`파일:줄`)는 아래 기준 커밋을 따른다.

## 목차

0. [문서 안내](#0-문서-안내)
1. [프로젝트 목적](#1-프로젝트-목적)
2. [디렉터리 구조와 책임](#2-디렉터리-구조와-책임)
3. [프로그램 실행 흐름](#3-프로그램-실행-흐름)
4. [핵심 클래스와 함수](#4-핵심-클래스와-함수)
5. [주요 설계 결정과 이유](#5-주요-설계-결정과-이유)
6. [강점과 한계](#6-강점과-한계)
7. [예상 기술면접 질문과 모범답안](#7-예상-기술면접-질문과-모범답안)
8. [추가로 공부할 개념](#8-추가로-공부할-개념)
- [부록 A. 주요 코드 위치 빠른 참조](#부록-a-주요-코드-위치-빠른-참조)

---

## 0. 문서 안내

### 0.1 기준 정보

| 항목 | 값 |
|---|---|
| 저장소 | `trustee-fds` (git 저장소, `main` 브랜치) |
| 기준 커밋 | `57a903d` — fix: CI 테스트 모듈 경로 일관화 |
| 작성일 | 2026-09-16 |
| 확인 방법 | 소스·룰·테스트·CI 설정을 직접 읽음. 테스트는 이 문서 작성 중 **실행하지 않았다.** |

본문에서 코드로 확인한 사실과 추론은 구분해 적는다. 확인하지 못한 추론은
**(미확인)** 으로 표시한다.

### 0.2 읽는 순서

| 목적 | 추천 순서 |
|---|---|
| 처음 구조를 익힐 때 | 1 → 2 → 3 → 4 |
| 설계 논리를 설명할 준비 | 5 → 6 |
| 면접 직전 복습 | 7 → 6.2 → 부록 A |
| 약점 보완 학습 | 8 |

### 0.3 용어 정리

| 용어 | 뜻 |
|---|---|
| 부인권 | 파산 전 채무자가 한 재산 감소 행위의 효력을 파산관재인이 부정해 재산을 되찾는 권리 |
| 무상부인 | 채무자회생법 제391조 제4호. 지급정지·파산신청 전후 일정 기간의 무상행위 및 이와 같이 볼 유상행위가 대상 |
| 소급기간 | 기준일로부터 거슬러 올라가는 기간. 이 프로젝트는 일반 6개월, 특수관계인 상대 12개월(제392조 제3항) |
| 지급정지일 | 채무자가 채무를 일반적·계속적으로 갚을 수 없다고 외부에 드러낸 날. 다투어지는 값이라 **시나리오 변수**로 다룬다 |
| 법률행위 상대방 | `legal_counterparty`. 소급기간 분기는 이 역할의 관계만 본다(대법원 2008다48117) |
| Value Out | 채무자에게서 나간 가치: 자산 공정가치 + 증가한 채무 + 포기한 권리 |
| Value In | 채무자에게 들어온 가치: 지급이 확인된 대가 + 실제 채무 감소 + 실현된 직접 이익 |
| 대가비율 | `Value In / Value Out`. 법정 기준이 아니라 **조사 우선순위 척도** |
| 순출연 추정액 | `Value Out − Value In`. 회수 기대액이 아니다 |
| 신호(signal) | 조사 우선순위를 높이는 사실. 예: 상대방이 특수관계인, 기준일 이후 행위 |
| 플래그(flag) | 판정 과정에서 붙는 주의 표시. 예: `boundary_case`, `relation_unverified` |

---

## 1. 프로젝트 목적

### 1.1 해결하려는 문제

파산관재인은 수십~수천 건의 거래 중 부인권을 검토할 거래를 골라야 한다. 이때
놓치기 쉬운 거래를 찾고, **왜 그 거래를 봐야 하는지**를 법률요건별 근거와 함께
제시하는 것이 목적이다. 프로젝트 스스로 정의한 성공 기준은 적발률이 아니라
"관재인이 더 짧은 시간에 누락 없이 증거를 확인하고, 판단 근거를 재현할 수 있는가"다.

현재 범위(Phase 1)는 **제391조 제4호(무상부인) 하나**다. 제1~3호는 수익자의 악의를
요건으로 하므로 거래 데이터만으로 검토할 수 없어 제외했다.

### 1.2 하는 일과 하지 않는 일

| 한다 | 하지 않는다 |
|---|---|
| 법정 시간 창과 행위 유형으로 후보 선별 | 부인권 성립 여부의 자동 판정 |
| Value Out ↔ Value In 비대칭 산출 | 대가비율을 법정 기준으로 선언 |
| 순출연 추정액 기준 우선순위 정렬 | 회수확률을 반영한 기대값 제시 |
| 확인된 사실과 미확인 사실의 구분 | 증거 부재를 무상성으로 단정 |
| 평가액 미상 거래의 별도 구획 표시 | 결측값을 0으로 대체 |
| 입력 무결성 검사 후 판정 | 공정가치·지급정지일의 확정 |

### 1.3 은행 FDS와의 차이

저장소 이름에 "fds"가 들어가지만 실시간 이상거래 차단 시스템이 아니다.

| | 은행 FDS | trustee-fds |
|---|---|---|
| 시간 방향 | 발생 시점 예측 | 파산선고 후 소급 추론 |
| 판단 기준의 출처 | 데이터에서 학습 | 조문·판례에 이미 열거됨 |
| 우선 지표 | 정밀도 | 금액가중 재현율(놓치지 않기) |
| 산출물 | 차단 결정 | 조사 후보와 법정 논거 |
| 소비자 | 기계 | 사람(관재인·법원) |
| 방식 | 주로 ML | 룰 기반 |

### 1.4 사용 제한

- README와 룰 파일은 연구·포트폴리오용이며 **실제 사건에 쓰면 안 된다**고 명시한다.
- 룰 상태는 `art391-4-gratuitous`가 `draft`, `related-party-classification`이
  `spec_verified`이며, 둘 다 법률 전문가 검토 전이다.
- 샘플 데이터는 합성 자료다(거래 34건, 사건 2건).

---

## 2. 디렉터리 구조와 책임

### 2.1 트리

```
trustee-fds/
├── src/                         # 파이썬 소스 (__init__.py 없음 → 네임스페이스 패키지)
│   ├── engine.py                # 판정 엔진 (1473줄)
│   ├── cli.py                   # CLI 리포트와 골든 파일 대조 (318줄)
│   ├── ui_service.py            # UI와 엔진 사이의 경계 계층 (75줄)
│   └── onprem.py                # 로컬 전용 Streamlit 실행기 (43줄)
├── streamlit_app.py             # Streamlit 화면 (99줄)
├── rules/
│   ├── art391_4_gratuitous.yaml # 무상부인 룰: 시간 창, 행위 유형, 경제 실질, 신호, 정렬
│   └── related_party.yaml       # 특수관계인 분류 룰 (시행령 제4조)
├── data/sample/
│   ├── cases.csv                # 사건 (2건)
│   ├── transactions.csv         # 거래 (34건)
│   ├── parties.csv              # 당사자와 관계 사실 (23건)
│   ├── transaction_parties.csv  # 거래-당사자 다대다 연결과 역할 (64건)
│   ├── transaction_links.csv    # 거래 간 자금흐름 연결 (2건, 선택 파일)
│   └── expected_results.csv     # 골든 파일: 거래별 기대 판정 (34건)
├── tests/
│   ├── test_engine.py           # 엔진 단위 테스트
│   ├── test_cli.py              # 대조 로직·CLI 출력 테스트
│   ├── test_ui_service.py       # UI 경계 통합 테스트 (unittest 스타일)
│   └── test_streamlit_app.py    # Streamlit AppTest
├── docs/                        # 온프레미스 안내, 설계안, 법리 연구, 재검증 리포트
├── scripts/build_usb.py         # 오프라인 배포 ZIP 생성
├── start-macos.command          # macOS 더블클릭 실행
├── start-windows.cmd            # Windows 더블클릭 실행
├── .github/workflows/ci.yml     # CI
├── requirements.txt             # 런타임: PyYAML, python-dateutil
├── requirements-dev.txt         # + pytest
└── requirements-ui.txt          # + streamlit
```

### 2.2 디렉터리·파일별 책임

| 경로 | 책임 | 의존 방향 |
|---|---|---|
| `src/engine.py` | 데이터 로드, 무결성 검사, 관계 판정, 4단계 판정, 정렬 | `rules/`, `data/` 읽기. 다른 모듈에 의존하지 않음 |
| `src/cli.py` | 텍스트 리포트, `expected_results.csv` 대조, 종료 코드 | `engine` |
| `src/ui_service.py` | 업로드 CSV 형식 검증, 임시 디렉터리 구성, 엔진 호출, JSON 직렬화 | `engine` |
| `src/onprem.py` | 사전 점검 후 loopback 전용 Streamlit 서버 실행 | `ui_service` |
| `streamlit_app.py` | 입력 선택, 결과 표시, 세션 상태, JSON 다운로드 | `ui_service`, `engine.partition` |
| `rules/` | 조문에서 나온 값과 판정 명세. 법률 전문가가 코드 없이 검토하는 대상 | 엔진이 읽음 |
| `data/sample/` | 합성 입력과 회귀 계약(골든 파일) | 엔진·CLI가 읽음 |
| `tests/` | 단위·통합·UI 테스트 | 모든 계층 |
| `scripts/` | 배포 산출물 생성 (`dist/`에 ZIP) | 저장소 파일 복사, `pip download` |
| `docs/` | 설계·법리 근거 문서 | 없음 |
| 런처 스크립트 | `.venv` 확인 후 `python -m src.onprem` 실행 | `onprem` |
| `.github/workflows/ci.yml` | 테스트, 골든 대조, 3개 OS에서 UI 실행기 점검 | 전체 |

의존 방향은 한쪽으로만 흐른다.

```
streamlit_app.py ─┐
onprem.py ────────┼─▶ ui_service.py ─┐
                  │                  ├─▶ engine.py ─▶ rules/*.yaml, data/sample/*.csv
cli.py ───────────┴──────────────────┘
```

### 2.3 입력 데이터 모델

다섯 개 테이블로 정규화되어 있다.

```
cases ──< transactions ──< transaction_parties >── parties
                 └──────< transaction_links (from → to)
```

| 테이블 | 기본키 | 핵심 필드 | 설계 포인트 |
|---|---|---|---|
| `cases` | `case_id` | `petition_date`, `suspension_of_payment_date`, `suspension_date_confirmed`, `adjudication_date`, `debtor_type` | 지급정지일은 확정값이 아닌 시나리오 변수 |
| `transactions` | `transaction_id` | `action_type`, `asset_fair_value`, `consideration_paid`, `payment_verified`, `liability_reduction` | `action_type`에는 사실(`sale`)만, 결론(`undervalue_sale`)은 금지 |
| `parties` | (`case_id`, `party_id`) | `relation_type`, `relation_valid_from/to`, `kinship_degree`, `officer_of`, `affiliate_status_verified` | 관계는 사실로만 기록, 해당 여부는 룰이 판정. 채무자 본인은 `self`로 등재 |
| `transaction_parties` | (`transaction_id`, `party_id`, `role`) | `role`, `share_ratio` | 역할: `legal_counterparty` / `principal_debtor` / `ultimate_beneficiary` |
| `transaction_links` | `link_id` | `from/to_transaction_id`, `link_type`, `confidence` | `alleged` 링크는 신호로 인정하지 않음 |

### 2.4 git 추적 대상이 아닌 폴더

| 폴더 | 정체 | 비고 |
|---|---|---|
| `.venv/` | 가상환경 | `.gitignore` 대상 |
| `.idea/` | PyCharm 설정 | `.gitignore` 대상 |
| `.pytest_cache/`, `__pycache__/` | 캐시 | `.gitignore` 대상 |
| `.ipynb_checkpoints/` | Jupyter 자동 저장 | `.gitignore`에 없어 `git status`에 미추적으로 표시됨 |
| `dist/` | `build_usb.py` 산출물 | `.gitignore` 대상 |

---

## 3. 프로그램 실행 흐름

### 3.1 공통 코어

CLI와 UI는 같은 엔진 함수를 같은 순서로 호출한다.

```python
rules = load_rules(ROOT)                                    # 1. 룰 로드·검증
unilateral = rules["art391-4-gratuitous"]["action_filter"].get("unilateral_acts", [])
ds = Dataset.load(ROOT, unilateral_acts=unilateral)         # 2. CSV 로드 + 무결성 검사
if any(i.severity == IssueSeverity.CRITICAL for i in ds.integrity_issues):
    ...                                                     # 3. 호출자가 중단
results = run(ds, rules)                                    # 4. 거래별 판정
unvalued, valued = partition(results, rules["art391-4-gratuitous"])  # 5. 구획·정렬
```

룰을 데이터보다 **먼저** 읽는 이유: 무결성 검사 중 `legal_counterparty` 누락 검사가
상대방 없는 단독행위(`unilateral_acts`, 예: 상속포기)를 예외로 두어야 하는데,
그 목록이 룰 YAML에 있기 때문이다.

### 3.2 CLI 경로

**시작 지점:** `src/cli.py`의 `main()` (`src/cli.py:284`)

```bash
python src/cli.py        # 또는 python -m src.cli
```

- 파일 상단 `if __package__:` 분기(`src/cli.py:9`)가 두 실행 방식에 맞춰
  `from .engine` / `from engine`을 고른다.
- Windows 콘솔(cp949)에서 `—`, `⚠` 출력 오류를 막기 위해 stdout을 UTF-8로
  재설정한다(`src/cli.py:19`).
- 명령줄 인자는 없고, 입력은 `data/sample/`로 고정이다.

| 순서 | 호출 | 하는 일 |
|---|---|---|
| 1 | `load_rules(ROOT)` | YAML 로드, 필수 메타데이터·중복·의존 룰 검증 |
| 2 | `report_rule_status(rules)` | `production`이 아닌 룰이 있으면 경고 |
| 3 | `Dataset.load(ROOT, ...)` | CSV 로드 직후 13개 무결성 검사 |
| 4 | `report_provisional_anchors(ds)` | 미확정 지급정지일을 기준점으로 쓰는 사건 경고 |
| 5 | `report_integrity(ds)` | 무결성 문제를 심각도·카테고리별로 출력 |
| 6 | CRITICAL 확인 | 있으면 `return 2` (`src/cli.py:302`) |
| 7 | `run(ds, rules)` | 판정 |
| 8 | `report(ds, rules, results)` | 평가 필요 / 평가 완료 구획, 과다변제 산식, 합계 출력 |
| 9 | `validate(results)` | 골든 파일 대조, `target_stage` 미도달 집계 |
| 10 | 종료 코드 | `return 1 if (m or u) else 0` (`src/cli.py:314`) |

**종료 코드 계약**

| 코드 | 의미 |
|---|---|
| 0 | 정상, 골든 일치, 모든 케이스가 검증 단계에 도달 |
| 1 | 골든 불일치 또는 `target_stage` 미도달 |
| 2 | CRITICAL 입력 무결성 문제로 판정 전 중단 |

### 3.3 Streamlit 경로

**시작 지점은 세 겹이다.**

```
start-macos.command / start-windows.cmd
 └─ .venv/bin/python -m src.onprem                 (.venv 없으면 안내 후 exit 2)
     └─ src/onprem.py main()                       src/onprem.py:12
         ├─ streamlit 설치 확인 (없으면 exit 2)
         ├─ analyze(sample_files()) 사전 점검 (CRITICAL이면 exit 2)
         ├─ --check-only 이면 "OK: transactions=34, candidates=20" 출력 후 exit 0
         └─ subprocess: python -m streamlit run streamlit_app.py
                          --server.address=127.0.0.1 --server.headless=true
                          --browser.gatherUsageStats=false ...
             └─ streamlit_app.py
```

`streamlit_app.py`에는 `main()`이 없다. Streamlit은 사용자가 위젯을 조작할 때마다
**스크립트 전체를 위에서 아래로 다시 실행**한다. 그래서 결과를
`st.session_state`에 보관하고, 입력이 바뀌었는지 해시로 판단한다.

```
사이드바: 입력 방식 선택
  ├─ "합성 샘플"  → files = sample_files()
  └─ "CSV 업로드" → file_uploader 5개 → files[name] = bytes
필수 4개 파일이 모두 있어야 "분석 실행" 버튼 활성화
current = fingerprint(files)                              # SHA-256
session_state의 key (mode, current)가 다르면 이전 결과 폐기
"분석 실행" 클릭
  └─ analyze(files)                                       # src/ui_service.py:31
       ├─ 파일 이름·필수 파일 검사
       ├─ 임시 디렉터리/data/sample/ 에 CSV 기록
       │    UTF-8 BOM 허용, 샘플 헤더 포함, 행별 열 개수, 빈 cases/transactions 거부
       ├─ Dataset.load(임시 root)
       └─ CRITICAL이면 results=[], blocked=True / 아니면 run()
  └─ 예외 (ValueError, KeyError, OSError, UnicodeError, csv.Error) → st.error
결과 표시: 룰 상태, 경고, 무결성 문제 → blocked이면 st.stop()
사건 필터 → partition() → 지표 4개 → 표 3개 → 거래 상세
"JSON 저장" 다운로드 버튼 → export_json()
```

`analyze()`가 임시 디렉터리에 `data/sample/` 구조를 만드는 이유는
`Dataset.load()`가 `root / "data" / "sample"` 경로만 읽기 때문이다. 엔진을 고치지
않고 UI를 붙이기 위한 우회다(6.2 참조).

### 3.4 판정 4단계 파이프라인

`run()` (`src/engine.py:1357`)은 거래마다 아래 단계를 거치고, 탈락하면 그
자리에서 `drop_reason`을 남기고 다음 거래로 넘어간다.

```
[사전] CRITICAL 무결성 문제 → ValueError
[사전] check_predicate_coverage(rule) → UnimplementedPredicate
for tx in ds.transactions:
  ① temporal  상대방 관계 판정 → 소급기간 → 시간 창
  ② action    route_out 조건 평가 → 후보 행위 유형 확인
  ③ economic  Value Out / In, 대가비율 → 염가 판정
  ④ ranking   자금흐름 추적 → 신호 평가 → 우선순위
```

| 단계 | 핵심 함수 | 탈락 조건 | 결과에 기록되는 것 |
|---|---|---|---|
| ① temporal | `classify_relation`, `resolve_lookback`, `temporal_window` | 거래일 < 창 시작일 | `lookback_months`, `boundary_case`, `post_adjudication_soft` |
| ② action | `_debt_repayment_within_tolerance` | `route_out` 해당, 또는 후보 행위 유형 아님 | `drop_reason`(라우팅 목적지 포함) |
| ③ economic | `economics`, `undervalue_verdict` | 대가비율 > 0.50 (염가 판정 대상 행위만) | `value_out`, `value_in`, `ratio` |
| ④ ranking | `evaluate_related_party_fund_flow`, `PREDICATES`, `triage_priority` | 없음 (후보 확정) | `candidate=True`, `signals`, `priority` |

주요 규칙:

- **소급기간:** 상대방 중 한 명이라도 `related`면 12개월, `verify`가 있으면
  누락 방지를 위해 12개월, 모두 `not_related`면 6개월.
- **시간 창:** 기준일(지급정지일, 파산신청일) 각각에서 소급기간을 빼고 그중 가장
  이른 날을 창 시작일로 삼는다(합집합). 파산선고일 이후 거래는 배제하지 않고
  `post_adjudication_soft` 플래그만 붙인다.
- **변제:** 실지급액과 실제 채무 감소액 차이가 2% 이내면 편파부인 검토로
  라우팅하고, 초과하면 초과분만 Value Out으로 분리한다.
- **미검증 대가:** `payment_verified != "Y"`면 지급 대가를 Value In에 넣지 않는다.
- **우선순위:** 대가비율이 없으면 `unresolved`. 보증처럼 비율이 척도로 맞지 않는
  행위는 신호 개수로 우선순위를 매긴다.

### 3.5 결과 저장 방식

| 경로 | 저장 | 형식 |
|---|---|---|
| CLI | **파일 저장 없음.** 모두 stdout 출력 | 텍스트 |
| Streamlit | 세션(`st.session_state`)에만 유지. 다운로드 버튼으로만 파일 생성 | `trustee-fds-review.json` |

JSON(`export_json`, `src/ui_service.py:63`)에는 입력 해시, 룰 상태, 면책 문구, 원본
입력 5개 테이블, 무결성 문제, 판정 결과 전체가 들어간다. 사건 필터와 관계없이
전체를 저장한다. DB 저장이나 세션 복원 기능은 없다.

### 3.6 CLI와 UI 비교

| | CLI | Streamlit |
|---|---|---|
| 시작 지점 | `src/cli.py` `main()` | 런처 → `src/onprem.py` `main()` → `streamlit_app.py` |
| 입력 | `data/sample/` 고정 | 합성 샘플 또는 업로드 |
| 입력 검증 | 무결성 검사 | 헤더·열 개수 검사 + 무결성 검사 |
| CRITICAL 처리 | exit 2 | 화면 오류 후 `st.stop()` |
| 골든 대조 | 함 | 안 함 |
| 결과 | stdout | 화면 + JSON 다운로드 |
| 테스트 | `test_cli.py` | `test_ui_service.py`, `test_streamlit_app.py` |

---

## 4. 핵심 클래스와 함수

### 4.1 데이터 구조

| 이름 | 위치 | 종류 | 역할 |
|---|---|---|---|
| `Dataset` | `src/engine.py:94` | `@dataclass` | 입력 5개 테이블 보관, 조회 메서드, 무결성 검사 |
| `Result` | `src/engine.py:1342` | `@dataclass` | 거래 1건의 판정 결과 |
| `IntegrityIssue` | `src/engine.py:61` | `@dataclass` | 무결성 문제 1건 (`category`, `severity`, `message`, `details`) |

**`Dataset` 주요 필드와 메서드**

```python
cases: dict            # case_id → row
transactions: list     # row[]
parties: dict          # (case_id, party_id) → row   ← 사건 간 party_id 충돌 방지
links: list            # transaction_parties row[]
tx_links: list         # transaction_links row[]
integrity_issues: list[IntegrityIssue]
unilateral_acts: frozenset
_raw_cases, _raw_parties   # dict로 접히기 전 원본 — 중복 키 검사용
```

| 메서드 | 역할 |
|---|---|
| `load(root, unilateral_acts)` (클래스 메서드) | CSV 읽기 → 인스턴스 생성 → `_check_all_integrity()` |
| `parties_of(case_id, tx_id, role)` | 특정 거래·역할의 당사자 목록. `case_id`를 강제해 교차 사건 오염 방지 |
| `inbound_fund_flows(case_id, tx_id, link_types)` | 이 거래로 들어오는 자금흐름 링크 |
| `_check_*()` 13개 | 형식, 참조, 중복 키, 커버리지, 지분 합계, 링크 시간 순서·자기참조·순환 |

**`Result` 필드**

| 필드 | 의미 |
|---|---|
| `tx_id` | 거래 ID |
| `reached` | 도달한 최종 단계 (`temporal`/`action`/`economic`/`ranking`) |
| `candidate` | 후보 여부 |
| `lookback_months`, `lookback_key` | 소급기간과 그 근거 판정 |
| `priority` | `high`/`medium`/`low`/`unresolved` |
| `value_out`, `value_in`, `ratio` | 경제 실질 |
| `signals`, `flags` | 신호 id 목록, 플래그 목록(중복 제거, 순서 유지) |
| `drop_reason` | 탈락 사유 |

### 4.2 오류 분류 타입

| 이름 | 위치 | 값 | 제어 흐름에 미치는 영향 |
|---|---|---|---|
| `IssueSeverity` | `src/engine.py:50` | `CRITICAL=1`, `WARNING=2`, `INFO=3` | **결정적.** CRITICAL이면 판정 중단 |
| `IssueCategory` | `src/engine.py:44` | `REFERENTIAL="참조 무결성"`, `VALUE_CONSTRAINT="값 제약"` | 없음. 보고·그룹화 전용 |
| `UnimplementedPredicate` | `src/engine.py:38` | `Exception` 하위 클래스 | 즉시 예외로 판정 중단 |

심각도 기준:

| 심각도 | 의미 | 예 |
|---|---|---|
| CRITICAL | 조용히 판정 오류로 이어짐 | 날짜·숫자 형식 오류, `legal_counterparty` 누락, 중복 기본키, 링크 시간 역행·순환 |
| WARNING | 조용히 데이터가 소실됨 | `ultimate_beneficiary` 누락, 고아 링크, `share_ratio` 합 ≠ 1.0 |
| INFO | 판정 영향 없음, 진단 개선 | 어떤 거래에서도 참조되지 않는 당사자 |

#### `UnimplementedPredicate` 상세

| 구분 | 내용 |
|---|---|
| 역할 | YAML `priority_signals`의 id에 대응하는 파이썬 술어가 없을 때 판정 시작 전에 중단 |
| 발생 위치 | `check_predicate_coverage(rule)` (`src/engine.py:1332`), `run()` 안 CRITICAL 검사 직후(`src/engine.py:1364`) |
| 입력 | `rule["priority_signals"]`, 모듈 전역 `PREDICATES` |
| 출력 | 정상이면 `None`. 누락 시 모든 누락 id를 담은 메시지로 예외 |
| 부작용 | 없음. 거래 루프 전에 멈추므로 부분 결과가 생기지 않음 |
| 실패 조건 | YAML에 id 추가 후 구현 누락, id 오타 |
| 잡지 못하는 것 | 코드에만 있는 술어(한 방향 검사), `when` 설명과 구현의 의미 불일치 |
| 호출자 처리 | CLI·UI·`onprem` 어디서도 잡지 않음 (6.2 참조) |

#### `IssueCategory` 상세

| 구분 | 내용 |
|---|---|
| 역할 | 무결성 문제의 종류 표시. 심각도와 직교하는 분류 |
| 입력 | 없음 (상수 열거형). 각 `_check_*()`가 선택해 `IntegrityIssue`에 넣음 |
| 출력(사용처) | CLI `report_integrity()` 머리글, `export_json()`의 `category` 문자열. Streamlit 화면은 사용하지 않음 |
| 부작용 | 없음 (불변) |
| 실패 조건 | 없는 멤버 참조 시 `AttributeError` |
| 잠재 위험 | CLI가 두 멤버를 직접 나열해 순회 → 새 카테고리는 CLI에 출력되지 않음 |

### 4.3 판정 함수

| 함수 | 위치 | 입력 → 출력 | 핵심 규칙 |
|---|---|---|---|
| `load_rules(root)` | `engine.py:815` | 경로 → `{rule_id: dict}` | 필수 메타데이터, 알 수 없는 `status`, 중복 `rule_id`, 의존 룰 누락 시 예외 |
| `classify_relation(party, tx_date, rp_rules, ...)` | `engine.py:989` | 당사자 → `(verdict, flags)` | 유형별 동적 판정 후 관계 유효기간 적용 |
| `resolve_lookback(verdicts, rule)` | `engine.py:1026` | 상대방 판정들 → `(months, basis, key)` | `related` 우선, 그다음 `verify` |
| `temporal_window(case_row, months, rule)` | `engine.py:1035` | 사건 → `(start, anchors, upper)` | 기준일이 하나도 없으면 `ValueError` |
| `_debt_repayment_within_tolerance(tx, route_rule)` | `engine.py:1049` | 거래 → `bool` | 데이터 결측이면 `False`(근거 없이 배제하지 않음) |
| `economics(tx, rule)` | `engine.py:1070` | 거래 → `(out, in, ratio, notes)` | 미검증 대가·미실현 이익 제외, 이중계상 방지, 0으로 나누지 않음 |
| `undervalue_verdict(tx, ratio, rule)` | `engine.py:1152` | → `candidate`/`unresolved`/`not_candidate` | 대상 행위만 `threshold`(0.50) 적용 |
| `triage_priority(action_type, ratio, n, rule)` | `engine.py:1134` | → 우선순위 | 비율 없음 → `unresolved`, 적용 범위 밖 행위 → 신호 개수 기준 |
| `partition(results, rule)` | `engine.py:1466` | 결과 → `(unvalued, valued)` | 평가 필요는 신호 수, 평가 완료는 순출연 추정액 내림차순 |

**관계 판정 분기** (`classify_relation`)

| `relation_type` | 판정 방식 |
|---|---|
| `former_spouse` | 혼인 기간 중 `related`, 이혼 후 `post_divorce_dependency`로 판정 |
| `affiliate` | 법인 채무자: `affiliate_status_verified`. 개인 채무자: 합산 지분 30% 이상 또는 사실상 지배 |
| `officer` | `officer_of`가 가리키는 대상(채무자 법인 / 계열회사 / 무관 법인)에 따라 |
| 혈족·인척 | `kinship_degree`가 한도(혈족 8, 인척 4) 이내인지. 결측이면 `verify` |
| `employee` | 직함이 아니라 생계 의존·공동생계 사실로 |
| 그 외 | `related_party.yaml` `classification`의 정적 판정 |

모든 경로에서 **사실이 없으면 `verify`** 로 남기고, `related`로 추정하지 않는다.

### 4.4 술어 레지스트리

```python
PREDICATES = {}                              # src/engine.py:1261

def predicate(pid):
    def deco(fn):
        PREDICATES[pid] = fn                 # import 시점에 등록
        return fn
    return deco

@predicate("post_anchor")
def _p9(c): return c["days_to_anchor"] is not None and c["days_to_anchor"] <= 0
```

- YAML의 `when`은 사람이 읽는 명세이고, 실행은 같은 id의 함수가 한다.
- 술어는 모두 하나의 문맥 딕셔너리 `ctx`를 받는다: `tx`, `value_in`, `value_out`,
  `cp_verdicts`, `pd_verdicts`, `counterparties`, `principal_debtors`,
  `days_to_anchor`, `related_party_fund_flow`.
- `run()`은 YAML에 선언된 **순서대로** 평가하므로 `signals` 순서가 결정적이다.
  골든 파일도 이 순서를 비교한다.

현재 등록된 술어 12개:

| id | 조건 요약 |
|---|---|
| `no_verified_consideration` | Value In 0이고 계약상 대가도 0 |
| `payment_unverified` | 계약상 대가는 있으나 지급 미검증 |
| `debt_repayment_excess` | 변제액 > 실제 채무 감소액 |
| `related_counterparty` | 상대방 중 `related` 존재 |
| `role_split_guarantee` | 주채무자가 상대방에 포함되지 않음 |
| `related_principal_debtor` | 주채무자 중 `related` 존재 (소급기간에는 영향 없음) |
| `related_party_fund_flow` | 관계인 상대 거래로 이어지는 자금흐름 연결 |
| `non_cash_waiver` | 포기한 권리 가치가 기재됨 |
| `proximity_before` | 가장 가까운 기준일 전 30일 이내 |
| `post_anchor` | 가장 가까운 기준일 당일 또는 이후 |
| `no_valuation` | Value Out 산출 불가 |
| `no_necessity_evidence` | 상당성 자료 없음 |

### 4.5 자금흐름 추적

| 함수 | 위치 | 역할 |
|---|---|---|
| `_trace_inbound_chains(...)` | `engine.py:1162` | 들어오는 링크를 재귀로 역추적. `(in_window 홉 목록, delayed 홉 목록)` 반환 |
| `evaluate_related_party_fund_flow(...)` | `engine.py:1220` | 추적 결과를 신호와 플래그로 변환. **상대방이 `related`일 때만** 신호 성립 |
| `Dataset._check_tx_link_cycles()` | `engine.py:755` | Tarjan SCC로 순환 구성요소 전부를 CRITICAL로 보고 |

추적 규칙 (`fund_flow_analysis`):

| 설정 | 값 | 의미 |
|---|---|---|
| `min_confidence` | `probable` | `alleged` 링크에서 경로를 끊음 (약한 고리) |
| `proximity_window_days` | 90 | 초과해도 배제하지 않고 `funds_flow_delayed` 표시 (soft) |
| `max_hops` | 3 | 추적 깊이 상한 |
| `multi_hop_flag` | `related_party_fund_flow_multihop` | 2홉 이상은 직접 연결과 구별 표시 |
| `cycle_policy` | `skip_visited` | 경로별 방문 집합으로 무한 루프 방지 |

샘플 예시: T001(배우자에게 매각) → T021(조카 계좌 경유) → T025(배우자에게 최종 귀속).

### 4.6 CLI·UI 계층 함수

| 함수 | 위치 | 역할 |
|---|---|---|
| `main()` | `cli.py:284` | CLI 오케스트레이션, 종료 코드 |
| `report()` | `cli.py:120` | 구획별 리포트, 과다변제 산식, 합계 |
| `report_integrity()` | `cli.py:62` | 무결성 문제 출력 (카테고리당 상세 3건까지) |
| `validate(results, expected_path)` | `cli.py:190` | 골든 파일 정확 대조, `(불일치 수, 미도달 수)` 반환 |
| `analyze(files)` | `ui_service.py:31` | 업로드 검증 → 임시 디렉터리 → 엔진 → `(ds, rules, results, blocked)` |
| `export_json(...)` | `ui_service.py:63` | 결과·입력 근거 JSON 문자열 |
| `fingerprint(files)` | `ui_service.py:24` | 파일명·길이·내용 기반 SHA-256. 입력 변경 감지와 JSON 추적용 |
| `sample_files()` | `ui_service.py:19` | 샘플 CSV를 바이트로 읽기 |
| `main()` | `onprem.py:12` | 사전 점검, loopback Streamlit 실행 |

`validate()`가 비교하는 항목: 결과 ID 집합(누락·예상 밖·중복), 후보 여부, 도달 단계,
소급기간, 우선순위, 탈락 사유, Value Out/In·대가비율(`math.isclose`), 신호·플래그
목록(순서 포함), `target_stage` 도달 여부.

---

## 5. 주요 설계 결정과 이유

### 5.1 머신러닝 대신 룰 기반

- **이유:** 판단 기준을 데이터에서 귀납할 필요가 없다. 제391조가 요건을 이미
  열거하고 있어, 문제는 패턴 학습이 아니라 **조문을 실행 가능한 형태로 번역**하는
  것이다. 법정에서는 "모델이 그렇게 판단했다"가 논거가 되지 않는다.
- **또 다른 이유:** 라벨이 붙은 학습 데이터가 존재하지 않는다.
- **대가:** 규칙에 없는 새 유형은 잡지 못하고, 룰 유지보수 비용이 든다.

### 5.2 룰은 YAML, 술어는 id로 연결

- 조문에서 나오는 값(소급기간, 기준 사건, 행위 유형, 임계값, 정렬 구획)은 YAML에 둔다.
- 필수 설정이 없으면 코드 기본값으로 보충하지 않고 실패한다.
- 조건식은 표현식 언어로 해석하지 않고, id로 등록된 파이썬 함수가 실행한다.

| 선택지 | 장점 | 단점 |
|---|---|---|
| `when` 문자열을 파싱·실행 (DSL) | 룰만 고치면 동작 변경 | 파서·평가기 필요, 안전성·디버깅 부담 |
| **id + 파이썬 함수 (채택)** | 결측값 처리·집합 연산을 정확히 표현, 단위 테스트 용이 | 룰과 코드 이중 관리 → `UnimplementedPredicate`로 보완 |

**이유:** 법률 전문가가 코드를 읽지 않고 YAML만 검토할 수 있고, 판정 결과를
조문까지 역추적할 수 있다.

### 5.3 사실은 데이터, 판단은 룰

- `parties.csv`에는 "배우자다", "지분 62%" 같은 사실만 적는다.
- `action_type`에는 `sale`만 적고 `undervalue_sale`은 적지 않는다.
- **이유:** 입력에 결론을 적으면 시스템이 판정한 것이 아니라 사람의 판정을
  되읽는 것이 된다. 테스트는 통과하지만 아무것도 검증하지 못한다.

### 5.4 무결성 검사는 로드 시점, 문제는 수집

- **로드 시점인 이유:** 무결성은 데이터의 속성이지 실행 경로의 속성이 아니다.
  예를 들어 참조가 깨진 당사자를 `parties_of()`가 조용히 걸러내면, 판정은 진행되고
  테스트도 통과할 수 있다.
- **수집하는 이유:** 데이터 오류는 사용자가 고칠 대상이므로 한 번에 모두
  보여 줘야 한다. WARNING·INFO는 판정을 막지 않아야 하므로 예외로 표현할 수 없다.
- **대조:** 술어 누락은 사용자가 고칠 수 없는 코드·설정 결함이라 즉시 예외로 멈춘다.

| 오류 종류 | 고칠 사람 | 처리 |
|---|---|---|
| 입력 데이터 문제 | 사용자 | `IntegrityIssue`로 수집, 심각도로 중단 여부 결정 |
| 룰-코드 불일치 | 개발자 | `UnimplementedPredicate` 즉시 발생 |

### 5.5 결측값을 0으로 대체하지 않는다

- 감정가 없는 미술품·비상장주식처럼 평가액이 없는 자산이 오히려 액수가 크고
  은닉 목적이 뚜렷할 수 있다.
- 결측을 0으로 두고 금액순으로 정렬하면 가장 위험한 항목이 목록 맨 아래로 사라진다.
- 그래서 `partition()`이 "평가 필요" 구획을 **위에** 따로 둔다. 합계에도 넣지 않고
  그 사실을 명시한다.
- 같은 원칙이 곳곳에 반복된다: 관계 사실 없음 → `verify`, 변제 데이터 없음 →
  라우팅하지 않음, 대가 미검증 → Value In에 산입하지 않음.

### 5.6 파생 필드를 저장하지 않는다

`days_before_event` 같은 값은 지급정지일이 바뀌면 즉시 낡는다. 기준일이 다투어지는
도메인이므로 실행할 때마다 계산한다.

### 5.7 골든 파일과 `target_stage`

- `expected_results.csv`는 거래마다 전체 결과 필드를 **정확히** 대조한다. 일부
  필드만 비교하면 CI 초록불이 검증 공백을 숨긴다.
- `target_stage`는 "이 케이스가 검증하려는 단계"다. 앞 단계에서 탈락한 케이스는
  뒷 단계를 검증하지 못하므로 **무력한 테스트**로 보고한다.
- 실제 사례: T015는 정상 매각이 대가비율에서 걸러지는지 보려던 케이스였지만
  시간 창 밖이라 3단계에 도달하지 못했다. 창 안으로 옮겨 해결했다.
- 원칙: **분기를 검증하는 케이스는 분기의 경계 사이에 놓여야 한다** (T101/T107).

### 5.8 약한 고리 원칙과 soft 경계

- 다단계 자금흐름은 홉마다 신뢰도와 시간 창을 독립 적용한다. 한 홉이라도 `alleged`면
  그 경로를 끊고, 한 홉이라도 시간 창 밖이면 경로 전체를 지연으로 표시한다.
- 시간 창 초과, 파산선고일 이후 거래는 **배제하지 않고 표시만** 한다. 확정되지 않은
  법리를 하드 필터로 만들면 누락이 생기기 때문이다.

### 5.9 UI는 엔진을 감싸기만 한다

- `ui_service.py`는 엔진 코드를 수정하지 않고 호출한다. `test_ui_service.py`는
  UI 경로 결과가 엔진 직접 호출 결과와 **동일**한지 검증한다.
- 로컬 전용 실행: `127.0.0.1` 바인딩, 사용 통계 끔, XSRF 보호. 사건 자료가 외부로
  나가지 않도록 하는 온프레미스 전제다.
- 입력이 바뀌면 이전 결과를 폐기해, 다른 자료의 결과가 남아 보이는 사고를 막는다.

---

## 6. 강점과 한계

### 6.1 강점

| 강점 | 근거 |
|---|---|
| 설명 가능성 | 모든 결과에 도달 단계, 탈락 사유, 신호, 플래그, 근거 조문이 남음 |
| 조용한 실패에 대한 경계 | 결측 ≠ 0, 사실 없음 → `verify`, 로드 시점 무결성 검사, 술어 누락 즉시 실패 |
| 도메인 지식의 코드화 수준 | 판례 원리를 대조군 거래로 만들어 회귀 테스트화 (T027 과다변제 버그 발견 사례) |
| 엄격한 회귀 계약 | 골든 파일 전 필드 정확 대조 + `target_stage`로 테스트의 판별력까지 검증 |
| 계층 분리 | 엔진 ↔ CLI ↔ UI 서비스 ↔ 화면의 한 방향 의존 |
| 운영 고려 | 종료 코드 계약, Windows 인코딩 대응, 3개 OS CI, 오프라인 배포와 SHA-256 목록 |
| 최소 의존성 | 런타임 의존성 2개 (PyYAML, python-dateutil) |

### 6.2 한계와 개선 후보

아래는 코드를 읽고 확인한 사항이다. 대부분 현재 샘플 규모에서는 증상이 없다.

| # | 문제 | 위치 | 영향 | 개선안 |
|---|---|---|---|---|
| 1 | 입력 경로가 `root/data/sample`로 고정 | `engine.py:116` `Dataset.load` | 실제 자료 경로를 지정할 수 없음. UI는 임시 디렉터리로 우회 | `load(data_dir)`처럼 데이터 디렉터리를 직접 받도록 분리 |
| 2 | 술어 커버리지 검사가 늦음 | `engine.py:1364` | CRITICAL 입력이면 `run()` 전에 끝나 설정 오류가 가려짐 | `load_rules()` 또는 `--check-only`에서도 검사 |
| 3 | `UnimplementedPredicate`를 아무도 잡지 않음 | `cli.py` `main()`, `streamlit_app.py:42` | CLI는 파이썬 기본 종료 코드 1 → "골든 불일치"와 구분 불가. UI는 스택 트레이스 노출 | 설정 오류 예외 계층 + CLI 전용 종료 코드(예: 3) + UI 안내 문구 |
| 4 | CLI가 `IssueCategory` 멤버를 직접 나열 | `cli.py:99` | 카테고리 추가 시 그 문제가 CLI에 출력되지 않음 | `for category in IssueCategory:` |
| 5 | 술어 검사가 한 방향이고 의미를 비교하지 않음 | `engine.py:1332` | 코드에만 있는 술어, `when`과 다른 구현을 잡지 못함 | 양방향 집합 비교, 신호별 명세 기반 테스트 |
| 6 | 촌수 한도가 코드에 하드코딩 | `engine.py` `classify_relation`의 `kinship_limits` | "조문 값은 YAML" 원칙의 예외. YAML note에만 설명됨 | `related_party.yaml`로 이동 |
| 7 | 평가 필요 구획 정렬이 YAML 명세와 다름 | `engine.py:1471` | YAML은 `signal_count desc, transaction_date desc`, 코드는 신호 수 다음 `tx_id` 오름차순 | 명세와 구현 중 하나로 통일 |
| 8 | YAML에 있으나 코드가 읽지 않는 명세 | `requires_human_verification`, `output.forbidden_phrasings`, `required_elements`, `multiple_counterparties.value_allocation` | README 아키텍처의 "⚠ 확인 필요 항목" 출력과 지분 비율 배분이 현재 실행되지 않음 (코드 검색으로 사용처 없음 확인) | 구현하거나 "미구현" 상태를 YAML에 명시 |
| 9 | 골든 파일은 독립 정답이 아님 | `expected_results.csv` | 엔진 출력을 수동 검토한 회귀 계약. 법률적 타당성을 증명하지 않음. 신호 순서 변경에도 갱신 필요 | 판례·전문가 검토 기반 기대값 분리 |
| 10 | 조회가 선형 탐색 | `parties_of`, `_trace_inbound_chains`의 `next(...)` | 거래·링크 수에 비례해 느려짐. 샘플 규모에서는 무관 | 로드 시 인덱스(dict) 구축 |
| 11 | 테스트 import 경로 혼용 | `test_engine.py`·`test_cli.py`(`engine`) ↔ `test_ui_service.py`·`test_streamlit_app.py`(`src.engine`) | 한 pytest 세션에서 같은 파일이 두 모듈 객체로 로드될 수 있음. 모듈 간 객체 비교 시 어긋날 위험 **(미확인: 현재 실패 사례는 확인하지 않음)** | import 경로 통일 (`src/__init__.py` 또는 패키지 설치) |
| 12 | `onprem.py` 서버 실행 경로 자동 테스트 없음 | `onprem.py:37` | CI는 `--check-only`만 검증. `AppTest`는 `streamlit_app.py`를 직접 실행 | 서브프로세스 명령 구성을 함수로 분리해 테스트 |
| 13 | 지급정지일 민감도 분석 미구현 | 로드맵 6 | 미확정 지급정지일을 잠정 기준점으로 쓰며 경고만 함 | 시나리오 자동 실행·비교 |
| 14 | Phase 1 범위 한계 | README §5, §8 | 제391조 제4호만, 정제 CSV 가정, 부작위 탐지 불가, 한 당사자의 복수 관계 표현 불가 | 로드맵 7·8 |

`python -m pytest`와 `pytest`의 차이도 알아 둘 만하다. `src/`에 `__init__.py`가 없고
`tests/`에서 `from src.ui_service import ...`를 쓰므로, 프로젝트 루트가 `sys.path`에
있어야 한다. `python -m pytest`는 현재 디렉터리를 `sys.path`에 넣지만 `pytest` 명령은
그렇지 않다. 기준 커밋 `57a903d`("CI 테스트 모듈 경로 일관화")가 CI 명령을
`python -m pytest`로 바꾼 것과 맞아떨어진다.

---

## 7. 예상 기술면접 질문과 모범답안

각 문항은 **질문 → 모범답안 → 면접관이 보는 포인트 → 이어서 나올 만한 질문** 순서다.

### 7.1 프로젝트 개요

#### Q1. 이 프로젝트를 1분 안에 설명해 주세요.

**모범답안**
> 파산관재인이 부인권을 검토할 거래를 고르도록 돕는 룰 기반 선별 시스템입니다.
> 현재는 채무자회생법 제391조 제4호 무상부인만 다룹니다. 사건·거래·당사자 CSV를
> 받아 먼저 무결성을 검사하고, 거래마다 시간 창 → 행위 유형 → 경제적 실질 →
> 우선순위 4단계를 거칩니다. 결과는 후보 여부만이 아니라 도달 단계, 탈락 사유,
> 신호를 함께 남겨 관재인이 근거를 재현할 수 있게 했습니다. 부인 성립을 판정하지
> 않고 조사 순서와 확인할 증거를 정리하는 것이 목표입니다. CLI와 로컬 전용
> Streamlit UI를 제공하고, 판례 원리를 반영한 골든 파일로 회귀 테스트를 합니다.

**면접관이 보는 포인트**
- 문제 → 입력 → 처리 → 출력 → 검증 순서로 구조화해 말하는가
- "탐지·판정"이 아니라 "선별·근거 정리"라는 범위를 정확히 말하는가

**이어서 나올 만한 질문**
- "사용자가 이 결과를 어떻게 쓰나요?" "가장 어려웠던 부분은?"

#### Q2. 왜 머신러닝이 아니라 룰 기반으로 만들었나요?

**모범답안**
> 세 가지 이유입니다. 첫째, 판단 기준이 조문과 판례에 이미 열거되어 있어 데이터에서
> 학습할 대상이 아니라 번역할 대상입니다. 둘째, 결과를 법정에서 설명해야 하므로
> 조문까지 역추적되는 근거가 필요합니다. 셋째, 라벨이 붙은 부인권 데이터셋이
> 존재하지 않습니다. 대신 규칙 밖의 새로운 유형은 잡지 못한다는 한계가 있고,
> 그래서 판정 대신 우선순위 신호와 `verify` 같은 판단 보류 상태를 두었습니다.

**면접관이 보는 포인트**
- 유행 기술을 쓰지 않은 결정을 **목적함수 차이**로 정당화하는가
- 룰 기반의 한계를 스스로 인정하는가

**이어서 나올 만한 질문**
- "ML을 보조적으로 쓴다면 어디에 쓰겠습니까?" (예: 비정형 문서에서 사실 추출)

### 7.2 아키텍처·설계

#### Q3. YAML의 `when` 문자열을 실행하지 않고 id로 등록된 파이썬 함수를 쓰는 이유는? `UnimplementedPredicate`는 어떤 위험을 막나요?

**모범답안**
> `when`을 실행하려면 표현식 언어의 파서와 평가기를 만들어야 하고, 안전성과 디버깅
> 부담이 생깁니다. 신호 조건에는 결측값 처리, 역할별 당사자 집합 비교, 자금흐름
> 추적 결과 같은 문맥이 필요해 파이썬 함수가 더 정확하고 테스트하기 쉽습니다.
> 그래서 YAML의 `when`은 법률 검토용 명세로 두고, 같은 id의 함수가 실행하도록
> 역할을 나눴습니다.
>
> 이 구조의 위험은 YAML에 신호를 추가했는데 구현이 없는 경우입니다. 그대로 두면
> `KeyError`가 나거나, 누군가 `.get()`으로 처리하는 순간 신호가 조용히 켜지지 않습니다.
> 신호 개수가 우선순위에 쓰이므로 결과가 틀렸는데도 정상처럼 보입니다.
> `check_predicate_coverage()`는 거래 루프 전에 누락 id를 모두 모아 예외를 던져
> 부분 결과도 남기지 않습니다. 다만 검사가 한 방향이라 코드에만 있는 술어나
> `when`과 다른 구현은 잡지 못합니다.

**면접관이 보는 포인트**
- DSL과 레지스트리의 트레이드오프, fail-fast의 도메인적 가치
- 한계를 스스로 짚는가

**이어서 나올 만한 질문**
- "`when`과 구현이 달라지는 문제는 어떻게 줄이겠습니까?"
- "데코레이터 레지스트리의 단점은?" (import 부작용, 전역 상태, 테스트 오염)

#### Q4. 술어 커버리지 검사를 `run()` 안, CRITICAL 검사 뒤에 두었습니다. 문제가 있나요?

**모범답안**
> 실제로 쓰기 직전에 검사해 `run()`만 호출하는 쪽에도 보장된다는 장점은 있습니다.
> 하지만 술어 누락은 입력과 무관한 배포 설정 오류인데 데이터 상태에 가려집니다.
> CLI는 CRITICAL 입력이면 `run()` 전에 종료 코드 2로 끝나므로, 사용자가 입력을
> 고친 뒤에야 전혀 다른 설정 오류를 만납니다. 더 근본적인 오류가 나중에 보고되는
> 셈입니다. `load_rules()` 마지막이나 `--check-only`에서 검사하고, `run()` 안의
> 검사는 방어용으로 남기는 것이 좋다고 봅니다.

**면접관이 보는 포인트**
- 코드 순서를 정확히 읽었는가
- fail-fast를 "예외를 던지는가"가 아니라 "얼마나 이른 시점에 발견하는가"로 설명하는가

#### Q5. 무결성 문제는 리스트로 모으고, 술어 누락은 예외로 던집니다. 왜 다르게 처리하나요? `IssueCategory`와 `IssueSeverity`의 역할도 설명해 주세요.

**모범답안**
> 누가 고칠 수 있는 오류인가의 차이입니다. 입력 데이터 오류는 사용자가 고칠 대상이라
> 한 번에 모두 보여 줘야 수정-재실행 반복이 줄고, WARNING과 INFO는 판정을 막지
> 않아야 하므로 예외로 표현할 수 없습니다. 술어 누락은 코드·설정 결함이라 그 상태의
> 어떤 결과도 믿을 수 없으므로 즉시 멈춥니다.
>
> 두 열거형은 직교하는 분류입니다. `IssueSeverity`는 CRITICAL이면 판정을 멈추는
> 제어 흐름을 결정하고, `IssueCategory`는 참조 무결성과 값 제약을 구분해 보고와
> 그룹화에만 씁니다. 같은 참조 무결성이라도 `legal_counterparty` 누락은 CRITICAL,
> `ultimate_beneficiary` 누락은 WARNING입니다. 한 가지 약점은 CLI
> `report_integrity()`가 두 카테고리를 직접 나열해 순회해서, 카테고리를 추가하면 그
> 문제가 CLI에 출력되지 않는다는 점입니다.

**면접관이 보는 포인트**
- 행동을 결정하는 축을 정확히 구분하는가
- 열거형 직접 나열의 확장성 문제를 코드에서 찾는가

#### Q6. 값이 비어 있을 때 이 시스템은 어떻게 동작하나요? 왜 그렇게 설계했나요?

**모범답안**
> 결측을 0이나 "관계 없음"으로 대체하지 않는 것이 일관된 원칙입니다. 공정가치가
> 없으면 Value Out을 `None`으로 두고 "평가 필요" 구획을 정렬 목록 위에 따로
> 보여 주며 합계에서도 제외합니다. 촌수나 생계 관계 사실이 없으면 `verify`로
> 남기고, 소급기간은 누락 방지를 위해 넓은 12개월을 씁니다. 변제 데이터가 없으면
> 정상 변제로 단정하지 않고 다음 단계로 보냅니다. 지급이 검증되지 않은 대가는
> Value In에 넣지 않습니다. 평가액이 없는 자산이 오히려 은닉 위험이 클 수 있어,
> 0으로 대체해 정렬하면 가장 위험한 거래가 목록 맨 아래로 사라지기 때문입니다.
> 대신 `_num()`은 잘못된 숫자도 `None`으로 바꾸므로, 로드 시점에 형식 오류를
> CRITICAL로 따로 잡아 "결측"과 "잘못된 값"을 구분합니다.

**면접관이 보는 포인트**
- 결측 처리를 도메인 위험(누락)과 연결하는가
- "결측"과 "형식 오류"의 구분까지 말하는가

### 7.3 오류 처리·운영

#### Q7. `UnimplementedPredicate`가 발생하면 CLI, Streamlit, `onprem --check-only`에서 각각 어떻게 되나요?

**모범답안**
> 셋 다 잡지 않습니다. CLI는 traceback과 함께 파이썬 기본 종료 코드 1로 끝나는데,
> CLI 계약에서 1은 골든 불일치라 종료 코드만 보는 CI는 설정 오류를 대조 실패로
> 오해합니다. Streamlit은 `ValueError`, `KeyError`, `OSError`, `UnicodeError`,
> `csv.Error`만 잡으므로 스택 트레이스 화면이 뜹니다. `ValueError`를 상속시키면
> 잡히긴 하지만 "입력 자료를 확인해 주세요"라는 틀린 안내가 나오므로 답이 아닙니다.
> `--check-only`는 사전 점검에서 비정상 종료해 CI가 실패로 잡습니다. 설정 오류용
> 예외 계층을 두고, CLI는 전용 종료 코드로, UI는 "배포 구성 오류" 안내로 처리하는
> 것이 좋습니다.

**면접관이 보는 포인트**
- 호출 경로를 따라가며 예외가 잡히는 곳을 확인하는가
- 종료 코드 계약 충돌 같은 운영 결함을 찾는가
- 쉬운 해결책의 부작용을 보는가

#### Q8. Streamlit UI에서 보안과 데이터 일관성을 위해 어떤 장치를 두었나요?

**모범답안**
> 네트워크 측면에서는 실행기가 `127.0.0.1`에만 바인딩하고 사용 통계 전송과 파일
> 감시를 끄며 XSRF 보호를 켭니다. 업로드 자료는 임시 디렉터리에 쓴 뒤 분석이
> 끝나면 삭제되고, 결과는 세션에만 남습니다. 일관성 측면에서는 파일명·길이·내용을
> 합친 SHA-256 지문과 입력 방식을 세션 결과의 키로 써서, 입력이 바뀌면 이전 결과를
> 폐기합니다. 다른 자료의 결과가 남아 보이는 사고를 막기 위한 것입니다. 업로드
> 파일은 헤더, 행별 열 개수, 빈 필수 테이블을 검사하고, CRITICAL 무결성 문제가
> 있으면 엔진을 호출하지 않습니다. 다만 README에도 적혀 있듯 방화벽이나 사용자
> 인증은 없으므로 완전한 격리 보장은 아닙니다.

**면접관이 보는 포인트**
- Streamlit 재실행 모델과 세션 상태를 이해하는가
- 제공하는 보장과 제공하지 않는 보장을 구분하는가

### 7.4 테스트 전략

#### Q9. 라벨 데이터가 없는데 어떻게 정확성을 검증했나요?

**모범답안**
> 지표 대신 케이스마다 무엇을 검증하는지 명시했습니다. 판례 원리를 대조군 거래로
> 만들고, `expected_results.csv`에 거래별 모든 결과 필드를 기대값으로 두어 CLI가
> 정확히 대조합니다. 일부 필드만 비교하면 CI가 통과해도 검증 공백이 숨기 때문입니다.
> 또 `target_stage`로 케이스가 검증하려는 단계까지 실제로 도달했는지 확인합니다.
> 앞 단계에서 탈락하면 통과해도 아무것도 증명하지 못하기 때문입니다. 실제로 T027
> 과다변제 케이스를 추가하면서, route_out이 숫자 조건을 평가하지 않고 모든 변제를
> 배제하던 버그를 찾았습니다. 한계는 골든 파일이 엔진 출력을 사람이 검토해 만든
> 회귀 계약이라 법률적 타당성을 독립적으로 증명하지는 못한다는 점입니다.
> 이를 보완하려고 함수 단위 테스트는 실제 CSV 없이 최소 딕셔너리로 따로 둡니다.

**면접관이 보는 포인트**
- 골든 파일 테스트의 장점과 한계(자기 참조성)를 함께 말하는가
- "무력한 테스트"라는 개념을 이해하는가

**이어서 나올 만한 질문**
- "골든 파일을 갱신할 때 실수로 버그를 정답으로 굳히지 않으려면?"

#### Q10. `UnimplementedPredicate`와 `IssueCategory` 관련해 어떤 테스트를 추가하겠습니까? 격리에서 주의할 점은?

**모범답안**
> 네 가지를 추가하겠습니다. 첫째, 없는 id를 가진 룰로 `pytest.raises(UnimplementedPredicate)`와
> 메시지에 누락 id가 모두 들어가는지 확인합니다. 둘째, 실제 `load_rules()` 결과에
> 대해 예외가 나지 않는지 보는 계약 테스트를 둡니다. 가장 값싸게 배포 회귀를 막습니다.
> 셋째, `PREDICATES` 키 집합과 YAML id 집합이 같은지 양방향으로 비교합니다. 넷째,
> 모든 `IssueCategory` 멤버로 가짜 문제를 만들어 `report_integrity()` 출력에 전부
> 나오는지 `capsys`로 확인합니다. 이 테스트가 카테고리 직접 나열 문제를 잡습니다.
>
> 격리 측면에서 `PREDICATES`는 import 시점에 채워지는 전역 딕셔너리라, 테스트에서
> 데코레이터로 가짜 술어를 등록하면 다른 테스트가 오염됩니다. `monkeypatch.setitem`이나
> `delitem`처럼 자동 복원되는 방식을 써야 합니다. 또 테스트마다 `engine`과
> `src.engine`으로 import 경로가 달라 같은 파일이 두 모듈로 로드될 수 있으니,
> 경로를 통일하는 것이 안전합니다.

**면접관이 보는 포인트**
- 계약 테스트 발상, 전역 레지스트리의 테스트 오염 인지
- 저장소 구조에서 나오는 import 함정까지 보는가

### 7.5 UI·배포

#### Q11. UI를 붙이면서 엔진을 수정하지 않았습니다. 어떻게 했고, 그 대가는 무엇인가요?

**모범답안**
> `ui_service.py`를 경계 계층으로 두었습니다. 업로드 바이트를 검증한 뒤 임시
> 디렉터리에 `data/sample/` 구조를 만들어 CSV를 쓰고, 기존 `Dataset.load()`를 그대로
> 호출합니다. 이렇게 하면 이미 검증된 엔진과 골든 파일을 건드리지 않고, 테스트로
> UI 경로 결과가 엔진 직접 호출 결과와 동일함을 보장할 수 있습니다. 대가는 엔진의
> 고정 경로 계약을 흉내 내는 우회라는 점입니다. 매번 파일을 디스크에 쓰고,
> 엔진이 경로 규약을 바꾸면 UI가 깨집니다. 장기적으로는 엔진이 데이터 디렉터리나
> 이미 읽은 행 목록을 직접 받도록 분리하는 것이 낫습니다.

**면접관이 보는 포인트**
- 리스크를 줄이는 점진적 통합 전략을 설명하는가
- 기술 부채를 인식하고 해소 방향을 제시하는가

#### Q12. CI는 무엇을 검증하고, 무엇을 검증하지 못하나요?

**모범답안**
> 두 작업이 있습니다. 첫 번째는 Python 3.9와 3.12에서 `python -m pytest`와
> `python src/cli.py`를 실행해 단위 테스트와 골든 파일 대조를 확인합니다. 두 번째는
> Ubuntu, Windows, macOS에서 UI 의존성까지 설치해 테스트, `onprem --check-only`, CLI를
> 돌립니다. CLI 종료 코드 계약 덕분에 골든 불일치나 CRITICAL 입력이 CI 실패로 이어집니다.
> 검증하지 못하는 것은 실제 브라우저 화면, 오프라인 동작, USB 복사본 설치,
> Streamlit 서버 실행 경로입니다. `docs/ONPREM.md`의 실기기 합격 기준표도 아직
> 미검증으로 남아 있습니다. `pytest` 대신 `python -m pytest`를 쓰는 이유는
> `src`가 `__init__.py` 없는 디렉터리라 프로젝트 루트가 `sys.path`에 있어야
> `from src...` import가 되기 때문입니다.

**면접관이 보는 포인트**
- CI 매트릭스의 목적과 한계를 구분하는가
- import 경로와 실행 방식의 관계를 이해하는가

---

## 8. 추가로 공부할 개념

### 8.1 Python

| 개념 | 이 프로젝트에서 쓰인 곳 | 공부할 질문 |
|---|---|---|
| `dataclass`, `field(default_factory=...)` | `Dataset`, `Result`, `IntegrityIssue` | 가변 기본값을 왜 `default_factory`로 주는가? `asdict()`는 어떻게 동작하는가? |
| `Enum` | `IssueSeverity`, `IssueCategory` | 멤버 순회, `.value`, 직렬화 시 주의점 |
| 데코레이터와 레지스트리 패턴 | `@predicate` | 데코레이터가 실행되는 시점은? 전역 상태의 단점은? |
| 예외 계층 설계 | `UnimplementedPredicate` | 도메인 예외를 어떤 기반 클래스에서 상속해야 하는가? |
| 모듈·패키지와 import 경로 | `if __package__:`, `sys.path.insert`, `python -m` | 네임스페이스 패키지, 같은 파일이 두 모듈로 로드되는 조건 |
| `classmethod` 팩토리 | `Dataset.load` | 생성자 대신 팩토리 메서드를 쓰는 이유 |
| 부동소수 비교 | `math.isclose`, `math.isfinite` | `NaN`, `inf`가 숫자 파싱에 섞이면 생기는 문제 |
| 날짜 연산 | `relativedelta(months=...)` | "6개월"과 "180일"의 차이, 말일 처리 |
| `csv.DictReader`, 인코딩 | `_rows`, `utf-8-sig` | BOM, `newline=""`, `strict=True`의 의미 |

### 8.2 소프트웨어 설계

- **Fail-fast와 조용한 실패(silent failure)**: 어떤 오류는 멈추고 어떤 오류는 모아야 하는가
- **룰 엔진 vs DSL vs 하드코딩**: 설정 가능성과 복잡도의 트레이드오프
- **관심사 분리와 계층 아키텍처**: 엔진 / 서비스 / 표현 계층
- **참조 무결성, 정규화, 복합 키**: 관계형 데이터 모델링 기초
- **명세와 구현의 동기화**: 계약 테스트, 스키마 검증(JSON Schema 등)
- **설명 가능성(explainability)**: 결과와 함께 근거를 남기는 데이터 구조 설계

### 8.3 알고리즘

| 개념 | 사용처 | 공부 포인트 |
|---|---|---|
| 깊이 우선 탐색(DFS), 재귀 | `_trace_inbound_chains` | 경로별 방문 집합(`visited | {from_id}`)과 전역 방문 집합의 차이, 깊이 제한 |
| 강연결요소(SCC), Tarjan 알고리즘 | `_check_tx_link_cycles` | `index`/`lowlink`, 스택, 시간복잡도 O(V+E), 재귀 깊이 한계 |
| DAG와 순환 탐지 | 링크 무결성 | 위상 정렬과의 관계 |
| 시간복잡도와 인덱싱 | `parties_of`의 선형 탐색 | 리스트 탐색을 딕셔너리 인덱스로 바꾸는 효과 |

### 8.4 테스트

- **pytest**: fixture, `monkeypatch`(`setitem`, `delitem`, `setattr`), `capsys`, `tmp_path`, `pytest.raises(match=...)`, `importorskip`
- **`unittest.mock.patch`**: `test_critical_input_never_runs_engine`처럼 "호출되지 않음"을 검증하는 방법, patch 대상 경로 규칙
- **골든 파일(스냅샷) 테스트**: 장점, 자기 참조 위험, 갱신 절차
- **테스트의 판별력**: 경계값 배치, 뮤테이션 테스트 개념 (`target_stage`와 연결)
- **계약 테스트**: 룰 파일과 코드, UI와 엔진의 동등성
- **Streamlit `AppTest`**: 위젯 조작, 세션 상태 확인

### 8.5 운영·배포

- **프로세스 종료 코드 규약**과 CI에서의 활용
- **CI 매트릭스**(Python 버전 × OS), 캐시
- **가상환경과 오프라인 설치**: `pip download --only-binary`, `--no-index --find-links`, wheel 호환성(OS·CPU·Python 버전)
- **무결성 검증과 서명의 차이**: SHA-256 목록은 손상 검사이지 배포자 증명이 아님
- **Streamlit 서버 옵션**: 바인딩 주소, XSRF, CORS, headless
- **문자 인코딩**: Windows cp949 콘솔, `sys.stdout.reconfigure`

### 8.6 도메인

- 채무자회생법 제391조(부인권 유형)와 제392조(특수관계인 특칙)
- 채무자회생법 시행령 제4조(특수관계인)
- 이 프로젝트가 인용하는 판례 원리: 2008다48117(특칙은 법률행위 상대방 기준),
  2014다765(교환적 담보), 2012다87751(일부 금원 수수), 2006다50444(구상권·과다변제)
- 지급정지의 의미와 입증 문제
- 유해성과 상당성 항변
- 참고: `docs/채무자회생법_제391조_제4호_무상부인_연구.md`

### 8.7 학습 체크리스트

- [ ] 샘플 데이터로 `python src/cli.py`를 실행하고 종료 코드 0을 확인했다
- [ ] 거래 1건(예: T001)이 4단계를 거치는 과정을 코드를 따라가며 손으로 계산했다
- [ ] T027의 Value Out이 30,000,000원인 이유를 설명할 수 있다
- [ ] T001 → T021 → T025 자금흐름에서 어떤 플래그가 붙는지 설명할 수 있다
- [ ] `IssueSeverity`와 `IssueCategory`의 역할 차이를 한 문장으로 말할 수 있다
- [ ] `UnimplementedPredicate`가 발생하는 조건과 잡지 못하는 경우를 말할 수 있다
- [ ] Streamlit 재실행 모델에서 `session_state`와 `fingerprint`의 역할을 설명할 수 있다
- [ ] `python -m pytest`와 `pytest`의 import 경로 차이를 설명할 수 있다
- [ ] 6.2의 한계 중 세 가지를 골라 개선안을 코드 수준으로 설명할 수 있다
- [ ] Tarjan SCC 알고리즘을 작은 그래프로 직접 추적해 봤다
- [ ] 7장 질문 12개에 답안을 보지 않고 답해 봤다

---

## 부록 A. 주요 코드 위치 빠른 참조

| 개념 | 위치 |
|---|---|
| CLI 시작 지점 | `src/cli.py:284` `main()` |
| CLI 실행 방식별 import 분기 | `src/cli.py:9` |
| CLI CRITICAL 중단 (exit 2) | `src/cli.py:302` |
| CLI 종료 코드 결정 | `src/cli.py:314` |
| CLI 카테고리 직접 나열 | `src/cli.py:99` |
| 골든 파일 대조 | `src/cli.py:190` `validate()` |
| 로컬 실행기 | `src/onprem.py:12` `main()` |
| Streamlit 서버 실행 | `src/onprem.py:37` |
| UI 경계 분석 | `src/ui_service.py:31` `analyze()` |
| 업로드용 임시 디렉터리 | `src/ui_service.py:39` |
| JSON 내보내기 | `src/ui_service.py:63` `export_json()` |
| UI 세션 결과 저장 | `streamlit_app.py:41` |
| UI 예외 처리 범위 | `streamlit_app.py:42` |
| JSON 다운로드 버튼 | `streamlit_app.py:98` |
| `UnimplementedPredicate` | `src/engine.py:38` |
| `IssueCategory` / `IssueSeverity` | `src/engine.py:44` / `src/engine.py:50` |
| `IntegrityIssue` | `src/engine.py:61` |
| `Dataset` / `Dataset.load` | `src/engine.py:94` / `src/engine.py:109` |
| 무결성 검사 모음 | `src/engine.py:165` `_check_all_integrity()` |
| 링크 순환 검출 (Tarjan) | `src/engine.py:755` |
| 룰 로드 | `src/engine.py:815` `load_rules()` |
| 관계 판정 | `src/engine.py:989` `classify_relation()` |
| 소급기간 / 시간 창 | `src/engine.py:1026` / `src/engine.py:1035` |
| 변제 허용오차 | `src/engine.py:1049` |
| 경제 실질 | `src/engine.py:1070` `economics()` |
| 우선순위 | `src/engine.py:1134` `triage_priority()` |
| 염가 판정 | `src/engine.py:1152` `undervalue_verdict()` |
| 자금흐름 추적 | `src/engine.py:1162`, `src/engine.py:1220` |
| 술어 레지스트리 | `src/engine.py:1261` `PREDICATES` |
| 술어 커버리지 검사 | `src/engine.py:1332` |
| `Result` | `src/engine.py:1342` |
| 판정 파이프라인 | `src/engine.py:1357` `run()` |
| 술어 검사 호출 | `src/engine.py:1364` |
| 신호 평가 | `src/engine.py:1454` |
| 구획·정렬 | `src/engine.py:1466` `partition()` |
| 무상부인 룰 | `rules/art391_4_gratuitous.yaml` |
| 특수관계인 룰 | `rules/related_party.yaml` |
| 골든 파일 | `data/sample/expected_results.csv` |
| CI | `.github/workflows/ci.yml` |
