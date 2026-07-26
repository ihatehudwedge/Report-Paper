# Ghidra - Claude 간 연동을 통한 변수, 메소드명 재할당

## 전체 아키텍처

```
[분석 VM: Ghidra + Jython 스크립트]
   ├─ 함수 디컴파일 (DecompInterface)
   ├─ 디컴파일된 의사코드를 Claude API로 전송 (HTTPS, urllib2)
   ├─ 응답(JSON)에서 제안된 함수명/변수명 파싱
   └─ Ghidra 프로그램에 이름 반영 (HighFunctionDBUtil, Function.setName)

[외부: api.anthropic.com]
   └─ Claude 모델이 코드 분석 후 이름 제안 JSON 반환
```

## Claude API 키 발급

1. platform.claude.com (Claude.ai 로그인 계정과 달라도 됨) 접속 후 이메일로 가입/로그인
2. 좌측 메뉴 Credits → Billing에서 결제 수단 등록합니다. 이걸 건너뛰면 키를 만들어도 요청이 거부됩니다.
3. Settings → API Keys (platform.claude.com/settings/keys) 이동 → Create Key 클릭
4. 키 이름(예: ghidra-vm-analysis), 필요시 Workspace 선택 → 생성
5. 키는 sk-ant-로 시작하며 생성 시 단 한 번만 화면에 표시됩니다. 즉시 복사해서 안전한 곳(예: 분석 VM의 환경변수)에 저장하세요. 다시 볼 수 없고, 잃어버리면 삭제 후 재발급해야 합니다.
6. 같은 화면에서 **Spend Limit**을 설정해두는 것을 권장합니다 — 스크립트가 대량의 함수를 자동으로 순회하며 호출하므로 예상치 못한 비용 발생을 막기 위함입니다.

   <img width="935" height="279" alt="image" src="https://github.com/user-attachments/assets/57ca093e-829c-4f34-940d-b1427cca88df" />


키는 Ghidra 자동화 전용 키를 하나 따로 만들어 두어서 악성코드에 의해 유출 시 대응을 쉽게 할 수 있습니다.

## 스크립트 배치

1. 작업 디렉토리를 하나 만들고 그 안에 아래 3개 파일을 함께 둡니다. (경로가 흩어지면 스크립트 상단의 API_KEY_FILE 경로 계산이 어긋나므로 반드시 같은 폴더에 배치)

    ```
    ghidra-claude/
    ├── .api                     # 실제 키로 채워넣은 뒤 사용
    └── ghidra_claude_rename.py
    ```

<details>
<summary>.api</summary>
<div markdown="1">

```
ANTHROPIC_API_KEY=sk-ant-xxxxxxxx
```

</div>
</details>

<details>
<summary>ghidra_claude_rename.py</summary>
<div markdown="1">

```py
# @runtime PyGhidra
# -*- coding: utf-8 -*-
 
import json
import os
import urllib.request
import urllib.error
 
from ghidra.app.decompiler import DecompInterface, DecompileOptions
from ghidra.util.task import ConsoleTaskMonitor
from ghidra.program.model.symbol import SourceType
from ghidra.program.model.pcode import HighFunctionDBUtil
 
 
# ---------------------------------------------------------------------------
# 설정값
# ---------------------------------------------------------------------------
SCRIPT_DIR = os.path.dirname(os.path.abspath(globals().get("__file__", "ghidra_claude_rename.py")))
API_KEY_FILE = os.path.join(SCRIPT_DIR, ".api")
 
API_URL = "https://api.anthropic.com/v1/messages"
MODEL = "claude-sonnet-4-6"          # 비용을 낮추려면 claude-haiku-4-5-20251001 사용
ANTHROPIC_VERSION = "2023-06-01"
MAX_TOKENS = 1024
MAX_FUNCTIONS = 50                    # 1회 실행당 처리할 함수 수 상한 (비용 안전장치)
SKIP_LIBRARY_FUNCTIONS = True
DECOMPILE_TIMEOUT_SECONDS = 30
 
SYSTEM_PROMPT = (
    "You are a reverse engineering assistant. You will be given decompiled "
    "pseudo-C code from Ghidra. Suggest a better, descriptive function name "
    "and better names for its local variables and parameters, based only on "
    "what the code logically does. Respond with ONLY a raw JSON object, no "
    "markdown, no commentary, in exactly this shape:\n"
    '{"function_name": "suggested_name", '
    '"variables": {"old_name1": "new_name1", "old_name2": "new_name2"}}\n'
    "If you cannot confidently improve a name, omit it from the object "
    "rather than guessing randomly. Never invent variables that are not "
    "present in the code."
)
 
 
def load_api_key(path):
    """.api 파일에서 ANTHROPIC_API_KEY=... 라인을 읽어온다."""
    if not os.path.isfile(path):
        raise RuntimeError(
            "API 키 파일을 찾을 수 없습니다: %s\n"
            "ANTHROPIC_API_KEY=sk-ant-... 형식으로 .api 파일을 같은 폴더에 만들어 두세요." % path
        )
 
    with open(path, "r") as f:
        for line in f:
            line = line.strip()
            if not line or line.startswith("#"):
                continue
            if "=" in line:
                key, _, value = line.partition("=")
                if key.strip() == "ANTHROPIC_API_KEY":
                    value = value.strip()
                    if value and not value.startswith("sk-ant-xxxx"):
                        return value
 
    raise RuntimeError(
        ".api 파일에서 유효한 ANTHROPIC_API_KEY 값을 찾지 못했습니다: %s" % path
    )
 
 
def call_claude(api_key, decompiled_code, function_name):
    """디컴파일된 코드를 Claude에 보내고 이름 제안 JSON을 받아온다."""
 
    user_prompt = (
        "Current function name: %s\n\nDecompiled code:\n```c\n%s\n```"
        % (function_name, decompiled_code)
    )
 
    payload = {
        "model": MODEL,
        "max_tokens": MAX_TOKENS,
        "system": SYSTEM_PROMPT,
        "messages": [
            {"role": "user", "content": user_prompt}
        ],
    }
 
    body = json.dumps(payload).encode("utf-8")
 
    request = urllib.request.Request(API_URL, data=body)
    request.add_header("content-type", "application/json")
    request.add_header("x-api-key", api_key)
    request.add_header("anthropic-version", ANTHROPIC_VERSION)
 
    with urllib.request.urlopen(request, timeout=60) as response:
        raw = response.read()
 
    data = json.loads(raw)
    text_parts = [block["text"] for block in data.get("content", []) if block.get("type") == "text"]
    combined = "".join(text_parts).strip()
 
    if combined.startswith("```"):
        combined = combined.strip("`")
        if combined.lower().startswith("json"):
            combined = combined[4:]
        combined = combined.strip()
 
    return json.loads(combined)
 
 
def decompile_function(decomp_iface, function, monitor):
    result = decomp_iface.decompileFunction(function, DECOMPILE_TIMEOUT_SECONDS, monitor)
    if not result or not result.decompileCompleted():
        return None, None
    return result.getHighFunction(), result.getDecompiledFunction().getC()
 
 
def apply_suggestions(program, function, high_function, suggestions):
    """제안받은 이름을 실제 프로그램에 반영한다."""
 
    new_func_name = suggestions.get("function_name")
    var_map = suggestions.get("variables", {})
 
    tx_id = program.startTransaction("Claude auto-rename: %s" % function.getName())
    success = False
    try:
        if new_func_name and new_func_name != function.getName():
            function.setName(new_func_name, SourceType.USER_DEFINED)
 
        if var_map and high_function:
            symbol_map = high_function.getLocalSymbolMap()
            for high_symbol in symbol_map.getSymbols():
                old_name = high_symbol.getName()
                if old_name in var_map:
                    new_name = var_map[old_name]
                    try:
                        HighFunctionDBUtil.updateDBVariable(
                            high_symbol, new_name, None, SourceType.USER_DEFINED
                        )
                    except Exception as e:
                        print("  [경고] 변수 '%s' -> '%s' 리네임 실패: %s" % (old_name, new_name, e))
        success = True
    finally:
        program.endTransaction(tx_id, success)
 
 
def run():
    api_key = load_api_key(API_KEY_FILE)
 
    program = currentProgram  # PyGhidra도 Jython과 동일하게 전역 변수로 제공함 (함수 호출 아님)
    monitor = ConsoleTaskMonitor()
 
    decomp_iface = DecompInterface()
    decomp_iface.setOptions(DecompileOptions())
    decomp_iface.openProgram(program)
 
    function_manager = program.getFunctionManager()
    functions = list(function_manager.getFunctions(True))
 
    processed = 0
    try:
        for function in functions:
            if processed >= MAX_FUNCTIONS:
                print("MAX_FUNCTIONS(%d) 도달, 중단합니다." % MAX_FUNCTIONS)
                break
 
            if SKIP_LIBRARY_FUNCTIONS and (function.isThunk() or function.isExternal()):
                continue
 
            print("처리 중: %s @ %s" % (function.getName(), function.getEntryPoint()))
 
            try:
                high_function, c_code = decompile_function(decomp_iface, function, monitor)
                if not c_code:
                    print("  디컴파일 실패, 건너뜀")
                    continue
 
                suggestions = call_claude(api_key, c_code, function.getName())
                apply_suggestions(program, function, high_function, suggestions)
 
                print("  -> 함수명: %s, 변수 %d개 변경 제안됨" % (
                    suggestions.get("function_name", "(변경없음)"),
                    len(suggestions.get("variables", {}))
                ))
                processed += 1
 
            except urllib.error.HTTPError as e:
                print("  [오류] API 호출 실패 (HTTP %s): %s" % (e.code, e.read()))
            except Exception as e:
                print("  [오류] %s 처리 중 예외: %s" % (function.getName(), e))
    finally:
        decomp_iface.dispose()
 
    print("완료: 총 %d개 함수 처리" % processed)
 
 
run()
```

</div>
</details>
<br/>

2. Ghidra에서 분석 대상 바이너리 열고 Code Browser 창을 활성화 후 자동분석 완료 대기 이후에 Window → Script Manager에 진입합니다.
   ! 이때 support/pyghidraRun.bat을 이용하여 프로젝트를 열어야 합니다.

   <img width="621" height="573" alt="image" src="https://github.com/user-attachments/assets/98e0a418-7a1a-4995-ba48-157b09867280" />


4. 우측 상단의 목록 버튼 클릭 → 새 창에서 + 버튼 클릭 → .api, python 코드가 있는 디렉토리를 적용시킵니다.

   <img width="1105" height="556" alt="image" src="https://github.com/user-attachments/assets/f9f052f5-94f3-469a-9ff7-5ed184cc89e7" />


6. Import 수행 후 Script Manager에서 생성한 코드를 추가한 뒤 Script 탭에서 실행하면 됩니다.

   <img width="1102" height="589" alt="image" src="https://github.com/user-attachments/assets/2ddf26a1-fb8e-44de-8b20-b70e9b805e81" />



## 운영 고려사항

### 비용 관리

- MAX_FUNCTIONS(현재 50)로 1회 실행당 호출 수를 제한해뒀습니다. 함수가 많은 바이너리는 여러 번에 나눠 돌리거나, 이 값을 늘리기 전에 Claude Platform API Keys 화면에서 Spend Limit을 반드시 설정하세요.
- 비용에 민감하면 MODEL을 `claude-haiku-4-5-20251001`로 낮추고, 복잡한 알고리즘 함수만 별도로 Sonnet/Opus급으로 재처리하는 2단계 전략도 가능합니다.

### .api 관리

- 키가 유출되면 Claude API Dashboard에서 즉시 삭제 후 재발급하세요. 동일 키를 여러 분석용 VM에서 재사용하지 말고, 이 워크플로 전용 키를 하나 분리해 두면 유출 시 영향 범위를 좁힐 수 있습니다.
- .gitignore에 `.api`를 반드시 추가하고, 저장소에는 `.api.example(더미 값)`만 커밋하세요.

### 네트워크

- 분석 VM에서 `api.anthropic.com:443` 아웃바운드가 허용되어야 합니다. 폐쇄망/에어갭 환경이라면 이 구성은 동작하지 않으므로, 별도의 프록시 게이트웨이나 오프라인 배치 처리 방식을 검토해야 합니다.

### 실패 처리

- 스크립트는 함수 단위로 예외를 잡아 로그만 남기고 다음 함수로 넘어가도록 되어 있습니다 (APIError, JSON 파싱 실패, 일반 예외). 한 함수의 실패가 전체 배치를 중단시키지 않지만, 실행 후 콘솔 로그에서 [오류] 항목을 반드시 검토해 수동 보완이 필요한 함수를 파악하세요.
- 레이트리밋(429) 발생 시 별도 재시도 로직은 없으므로, 대량 바이너리를 처리할 계획이라면 ask_claude 호출부에 지수 백오프 재시도를 추가하는 것을 권장합니다.
