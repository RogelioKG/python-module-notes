# coverage

[![RogelioKG/coverage](https://img.shields.io/badge/Sync%20with%20HackMD-grey?logo=markdown)](https://hackmd.io/@RogelioKG/coverage)



## References
+ 📑 [**Documentation - Coverage.py**](https://coverage.readthedocs.io/)
+ 📑 [**Documentation - pytest-cov**](https://pytest-cov.readthedocs.io/)
+ 🎞️ [**碼農高天 - 【python】使用 coveragepy 计算单元测试的覆盖率**](https://youtu.be/5Md_3ZeuYPc)



## Installation
+ 專案開發依賴 (推薦 [uv](https://hackmd.io/@RogelioKG/uv))
  ```bash
  uv add coverage --group dev
  ```
  |🚨 <span class="caution">CAUTION</span>|
  |:---|
  |不要用 `uv tool install coverage` 或 `uvx coverage` 裝成全域工具。<br>coverage 是在<mark>自己所在的環境</mark>裡執行被測程式，工具環境裡沒有你的專案和 pytest，`coverage run -m pytest` 會直接找不到模組。|

  |📘 <span class="note">NOTE</span>|
  |:---|
  |Python 3.10 要讀 `pyproject.toml` 裡的設定，需要 TOML extra：`uv add "coverage[toml]" --group dev`<br><span style="color: grey;">註：Python 3.11 起標準函式庫內建 `tomllib`，不需要這個 extra。</span>|



## Cheatsheet
👇 最常用到的指令

> ...



## Note
|📘 <span class="note">NOTE</span> : coverage|
|:---|
|筆者自 YYYY/MM 入坑 coverage，持續記錄 coverage 的點點滴滴...<br><span style="color: grey;">上次更新時間：2026/10/11 (版本：7.16.2)</span>|
|coverage 的工作分成兩段：<mark>`run` 收集</mark> → <mark>`report` / `html` 報告</mark>，中間靠資料檔銜接|
|<mark>資料檔：`.coverage`</mark> (SQLite 資料庫，記錄哪些行被執行過)<br><mark>設定檔：`.coveragerc`</mark> 或 `pyproject.toml` 的 `[tool.coverage.*]`，兩者別搞混|
|預設只量 statement coverage，<mark>branch coverage 要另外開</mark> (詳見 [Coverage Types](#coverage-types))|
|量測核心：Python 3.14+ 預設用 <mark>`sys.monitoring`</mark>，更早的版本用 C 寫的 trace function<br><span style="color: grey;">註：`sys.monitoring` 來自 PEP 669 (Python 3.12)，是比 `sys.settrace` 開銷更低的監控 API。</span>|



## Coverage Types
| 類型 | 量測什麼 | 開啟方式 | 報告欄位 |
| --- | --- | --- | --- |
| **Statement** | 每一行可執行敘述，有沒有被執行過 | 預設 | `Stmts` / `Miss` |
| **Branch** | 每個判斷點的每一條去路，有沒有走過 | `--branch` | `Branch` / `BrPart` (partial branch) |

> 只看 statement coverage 很容易被騙：`if` 沒有 `else` 時，條件不成立的那條路沒有對應的「行」，\
> 所以就算從沒測過，statement coverage 也照樣 100%。

+ 被測程式 `clamp.py`
  ```py
  def clamp_positive(x):
      result = 0
      if x > 0:
          result = x
      return result
  ```
+ 測試只測了正數 `test_clamp.py`
  ```py
  from clamp import clamp_positive


  def test_clamp_positive():
      assert clamp_positive(5) == 5
  ```
+ statement coverage：每一行都被執行過，100%
  ```
  Name       Stmts   Miss  Cover   Missing
  ----------------------------------------
  clamp.py       5      0   100%
  ----------------------------------------
  TOTAL          5      0   100%
  ```
+ branch coverage：第 3 行的 `if` 從沒為 False (沒走過 `3->5`)，86%
  ```
  Name       Stmts   Miss Branch BrPart  Cover   Missing
  ------------------------------------------------------
  clamp.py       5      0      2      1    86%   3->5
  ------------------------------------------------------
  TOTAL          5      0      2      1    86%
  ```
  > 總涵蓋率 = (執行過的敘述 + 走過的分支) / (敘述總數 + 分支總數)\
  > = (5 + 1) / (5 + 2) ≈ 86%

|📗 <span class="tip">TIP</span>|
|:---|
|<mark>一律開 branch coverage</mark>，寫進設定檔就不會忘 (詳見 [tool.coverage.run](#toolcoveragerun：量測設定))|



## Binary

### coverage
> 本體。等同 `python -m coverage`。

### coverage3 / coverage-3.X
> 7.13 起棄用的別名，執行時會印出棄用訊息，之後會移除。



## Commands
詳見 [COVERAGE CLI](https://coverage.readthedocs.io/en/latest/commands/index.html)。

> 以下範例共用這個小專案：`unused.py` 沒有任何測試 import 它；`divide()` 除以零的分支沒被測到。

+ 目錄
  ```py
  demo/
  ├── src/
  │   └── mymath/
  │       ├── __init__.py
  │       ├── core.py     # 被測程式
  │       └── unused.py   # 沒有任何測試 import 它
  ├── tests/
  │   └── test_core.py    # 只測了 clamp_positive(5) 和 divide(6, 3)
  └── pyproject.toml      # [tool.pytest.ini_options] pythonpath = ["src"]
  ```
+ `src/mymath/core.py`
  ```py
  def clamp_positive(x):
      result = 0
      if x > 0:
          result = x
      return result


  def divide(a, b):
      if b == 0:
          raise ZeroDivisionError("b must not be 0")
      return a / b
  ```

### `run`：收集資料
> 用 coverage 執行 Python 程式，邊跑邊記錄每一行有沒有被執行，結果寫進資料檔 `.coverage`。

|🚨 <span class="caution">CAUTION</span>|
|:---|
|每次 `run` 預設都會<mark>從頭開始，覆蓋掉舊的 `.coverage`</mark>。要累加多次執行的結果，請用 `--append`。|

|🔮 <span class="important">IMPORTANT</span>|
|:---|
|coverage 的選項要放在 <mark>`run` 之後、程式之前</mark>，放在程式後面會被當成程式自己的參數。<br>✅ `coverage run --branch -m pytest`<br>❌ `coverage run -m pytest --branch` (`--branch` 被丟給 pytest)|

|📗 <span class="tip">TIP</span>|
|:---|
|記得把 `.coverage*` 加進 `.gitignore`；`htmlcov/` 則不用，它自帶 `.gitignore` (詳見 [html](#html：html-報告))|

+ #### ` `
  > 執行腳本 (後面的參數照常傳給腳本的 `sys.argv`)
  ```
  coverage run main.py arg1 arg2
  ```
+ #### `-m`
  > 以模組方式執行 (如同 `python -m`)。\
  > 實務上最常用來<mark>執行測試框架</mark>，讓 coverage 在跑測試的同時量測被測程式。
  + unittest 框架
    ```
    coverage run -m unittest discover
    ```
  + pytest 框架
    ```
    coverage run -m pytest
    ```
+ #### `--branch`
  > 除了 statement，同時量測 branch coverage (詳見 [Coverage Types](#coverage-types))
  ```
  coverage run --branch -m pytest
  ```
+ #### `--source`
  > 只量測指定目錄或套件裡的程式碼
  ```
  coverage run --source=src -m pytest
  ```
  + 不加 `--source`：量到什麼報什麼，連測試檔都算進去，<mark>從沒被 import 的檔案根本不會出現</mark>
    ```
    Name                     Stmts   Miss  Cover
    --------------------------------------------
    src/mymath/__init__.py       0      0   100%
    src/mymath/core.py           9      1    89%
    tests/test_core.py           5      0   100%
    --------------------------------------------
    TOTAL                       14      1    93%
    ```
  + 加了 `--source=src`：測試檔不見了，沒被 import 的 `unused.py` 以 0% 現身
    ```
    Name                     Stmts   Miss  Cover
    --------------------------------------------
    src/mymath/__init__.py       0      0   100%
    src/mymath/core.py           9      1    89%
    src/mymath/unused.py         2      2     0%
    --------------------------------------------
    TOTAL                       11      3    73%
    ```
  |🔮 <span class="important">IMPORTANT</span>|
  |:---|
  |涵蓋率從 93% 掉到 73%，後者才是真相。<br>設定了 source，coverage 才會<mark>搜尋整個原始碼樹，把完全沒被執行的檔案也列進報告</mark>。<br><span style="color: grey;">註：只會搜尋「可 import」的檔案 (位於根目錄，或所在目錄有 `__init__.py`)。</span>|

  |🚨 <span class="caution">CAUTION</span>|
  |:---|
  |命令列的 `--source` <mark>只作用在這次 `run`</mark>，不會帶到之後的 `xml` 等報告指令。<br>官方建議寫進設定檔，讓所有指令一致 (詳見 [tool.coverage.run](#toolcoveragerun：量測設定))。|
+ #### `--append`
  > 把這次的結果累加進既有的 `.coverage` (簡寫 `-a`)。\
  > 適合要用不同參數跑好幾次，才能把程式碼走完的情境。
  ```
  coverage run main.py --mode fast
  coverage run --append main.py --mode slow
  ```
+ #### `--parallel-mode`
  > 資料檔名加上機器名、PID 與亂數 (簡寫 `-p`)，每次執行各寫各的、不互相覆蓋。\
  > 適合多行程、多環境 (多個 Python 版本、多台機器) 的量測，最後再[合併](#combine：合併資料檔)。
  ```
  coverage run --parallel-mode -m pytest
  ```
  ```
  .coverage.vm.pid142.XEXnJVDx.HvRXaecpLmCh
  .coverage.vm.pid144.XKwatRRx.HvRXaecpLmCh
  ```
+ #### `--concurrency`
  > 程式有用到 `multiprocessing`、`gevent`、`greenlet`、`eventlet` 時要宣告，否則量測結果會很不準。\
  > (預設只處理 `thread`；多個值用逗號隔開)
  ```
  coverage run --concurrency=multiprocessing main.py
  ```
  |🚨 <span class="caution">CAUTION</span>|
  |:---|
  |命令列選項<mark>傳不到 multiprocessing 開出來的子行程</mark>，`--branch` 這類選項和 `--concurrency=multiprocessing` 一起下會直接被拒絕。<br>這種情況下，<mark>選項一律寫進設定檔</mark>。|

### `report`：終端機報告
> 根據 `.coverage` 所記錄的測試結果，在終端機印出涵蓋率表格。

+ #### ` `
  > 印出每個檔案的敘述數、未執行數與涵蓋率
  ```
  coverage report
  ```
  ```
  Name                     Stmts   Miss  Cover
  --------------------------------------------
  src/mymath/__init__.py       0      0   100%
  src/mymath/core.py           9      1    89%
  src/mymath/unused.py         2      2     0%
  --------------------------------------------
  TOTAL                       11      3    73%
  ```
+ #### `--show-missing`
  > 多一欄 `Missing`，列出沒被執行到的地方 (簡寫 `-m`)
  ```
  coverage report -m
  ```
  > (以 `--branch` 量測的結果)
  ```
  Name                     Stmts   Miss Branch BrPart  Cover   Missing
  --------------------------------------------------------------------
  src/mymath/__init__.py       0      0      0      0   100%
  src/mymath/core.py           9      1      4      2    77%   3->5, 10
  src/mymath/unused.py         2      2      0      0     0%   1-2
  --------------------------------------------------------------------
  TOTAL                       11      3      4      2    67%
  ```
  + `1-2`：第 1 到 2 行都沒執行
  + `10`：第 10 行沒執行 (`raise` 從沒觸發)
  + `3->5`：從第 3 行跳到第 5 行的分支沒走過 (`if x > 0` 從沒為 False)
  + `2->exit`：從第 2 行直接離開函式的分支沒走過 (例如函式結尾一個沒有 `else` 的 `if`)

  |📘 <span class="note">NOTE</span>|
  |:---|
  |分支只有在<mark>起點和終點都被執行過</mark>時，才會列進 `Missing`。<br>第 9 行的 `if b == 0` 其實也是 partial branch (有算進 `BrPart`)，但它的終點第 10 行本身就沒執行，所以只顯示 `10`。|
+ #### `--skip-covered`
  > 隱藏 100% 的檔案，專心看還需要補測試的
  ```
  coverage report --skip-covered
  ```
+ #### `--skip-empty`
  > 隱藏沒有程式碼的檔案 (例如空的 `__init__.py`)
  ```
  coverage report -m --skip-covered --skip-empty
  ```
  ```
  Name                   Stmts   Miss  Cover   Missing
  ----------------------------------------------------
  src/mymath/core.py         9      1    89%   10
  src/mymath/unused.py       2      2     0%   1-2
  ----------------------------------------------------
  TOTAL                     11      3    73%

  1 file skipped due to complete coverage.
  ```
+ #### `--fail-under`
  > 總涵蓋率低於門檻時，exit code 為 2。CI 擋 PR 用。
  ```
  coverage report --fail-under=80
  ```
  ```
  Coverage failure: total of 73 is less than fail-under=80
  ```
  |📘 <span class="note">NOTE</span>|
  |:---|
  |`html` 等其他報告指令也吃這個選項|
+ #### `--format`
  > 輸出格式
  + `text`：預設的純文字表格
  + `markdown`：Markdown 表格 (可貼進 PR，或寫進 GitHub Actions 的 [job summary](#github-actions))
  + `total`：只印出總涵蓋率數字，不帶 `%` (做 badge、寫腳本判斷)
    ```
    coverage report --format=total
    ```
    ```
    73
    ```
+ #### `--sort`
  > 依欄位排序：`name` (預設)、`stmts`、`miss`、`branch`、`brpart`、`cover`；前面加 `-` 為遞減
  ```
  coverage report --sort=-cover
  ```
+ #### `--precision`
  > 涵蓋率顯示到小數點後幾位 (預設 0)。也會影響 `--fail-under` 的判斷。
  ```
  coverage report --precision=2
  ```

### `html`：HTML 報告
> 根據 `.coverage` 所記錄的測試結果，產生一個 `htmlcov` 目錄，裡面是每個檔案逐行標色的詳盡 HTML 報告。

+ #### ` `
  > 產生報告
  ```
  coverage html
  ```
  > 用預設瀏覽器打開 (PowerShell)
  ```
  start htmlcov\index.html
  ```
  + 目錄結構
    ```py
    htmlcov/
    ├── .gitignore                         # 內容為 *，整個報告不會被 commit
    ├── index.html                         # 入口：檔案總覽
    ├── function_index.html                # 函式總覽
    ├── class_index.html                   # 類別總覽
    ├── z_5482a63b3fcfe3a5_core_py.html    # 每個原始碼檔案一頁
    ├── status.json                        # 增量產生用的紀錄
    └── ...                                # CSS、JS、圖示
    ```
  + 行的顏色
    + 綠：執行過
    + 紅：沒執行
    + 灰：被[排除](#exclusion)
    + 黃：partial branch (只走過其中一條路)

  |📗 <span class="tip">TIP</span>|
  |:---|
  |報告頁面有鍵盤快捷鍵，按 `?` 看全部<br>常用：`[` / `]` 上一個 / 下一個檔案、`u` 回總覽、`j` / `k` 在標色區塊間跳轉|

  |📘 <span class="note">NOTE</span>|
  |:---|
  |再次輸出到同一個目錄時，<mark>內容沒變的頁面會直接跳過</mark>，大專案重產報告會快很多|
+ #### `--directory`
  > 指定輸出目錄 (簡寫 `-d`，預設 `htmlcov`)
  ```
  coverage html -d coverage_html
  ```
+ #### `--title`
  > 報告標題 (純文字，不是 HTML)
  ```
  coverage html --title="mymath coverage"
  ```
+ #### `--show-contexts`
  > 每一行標出是哪些測試執行到它 (詳見 [contexts](#contexts))
  ```
  coverage html --show-contexts
  ```

### `xml` / `json` / `lcov`：匯出報告
> 輸出給其他工具讀的格式。選項大致同 [report](#report：終端機報告) (`--include`、`--omit`、`--fail-under` 等)，輸出位置用 `-o` 指定。

+ #### `xml`
  > Cobertura 相容格式 (預設 `coverage.xml`)。\
  > Codecov、GitLab、Jenkins、SonarQube 等 CI / 程式碼品質平台普遍吃這個。
  ```
  coverage xml
  ```
+ #### `json`
  > 預設 `coverage.json`，適合自己寫腳本分析。`--pretty-print` 可排版成人類可讀。
  ```
  coverage json --pretty-print
  ```
  |📗 <span class="tip">TIP</span>|
  |:---|
  |`-o -` 會輸出到 stdout，可以直接 pipe 給其他工具<br><span style="color: grey;">註：7.14.2 起提示訊息改走 stderr，不會再混進 JSON。</span>|
  ```
  coverage json -o - | jq .totals.percent_covered
  ```
+ #### `lcov`
  > LCOV 格式 (預設 `coverage.lcov`)。源自 C/C++ 的 gcov 生態系，不少編輯器外掛與 CI 服務都讀得懂。
  ```
  coverage lcov
  ```

### `combine`：合併資料檔
> 把 [`--parallel-mode`](#run：收集資料) 產生的多個 `.coverage.*` 合併成一個 `.coverage`。

|🔮 <span class="important">IMPORTANT</span>|
|:---|
|7.14 起，`report`、`html` 等報告指令<mark>會自動先合併</mark>平行資料檔，大部分情況已經不用手動 `combine`。<br>想保留合併前的原始檔，報告指令加 `--keep-combined` (7.15+)。|

+ #### ` `
  > 合併資料檔所在目錄 (預設為目前目錄) 下所有 `.coverage.*`
  ```
  coverage combine
  ```
  |🚨 <span class="caution">CAUTION</span>|
  |:---|
  |合併完，<mark>原始的平行資料檔會被刪掉</mark> (報告指令的自動合併也一樣)|
+ #### `<path>`
  > 指定要合併的檔案或目錄 (例如從多台 CI 機器收回來的資料)。\
  > 有給路徑時就<mark>不會再掃目前目錄</mark>，需要的話要自己列進去。
  ```
  coverage combine ci-linux/ ci-windows/
  ```
+ #### `--keep`
  > 合併後保留原始檔
  ```
  coverage combine --keep
  ```
+ #### `--append`
  > 累加進現有的 `.coverage`，而不是覆蓋它
  ```
  coverage combine --append
  ```

|📗 <span class="tip">TIP</span>|
|:---|
|跨機器、跨 OS 合併時，同一份原始碼的路徑會不一樣，要用 [tool.coverage.paths](#toolcoveragepaths：路徑對應) 或 `relative_files = true` 對應起來<br><span style="color: grey;">註：7.16 起合併時會自動轉換 `/` 與 `\`，跨 OS 少了一個坑。</span>|

### `erase`：清除資料
> 刪掉 `.coverage`，讓下一次從乾淨的狀態開始。
```
coverage erase
```
|🚨 <span class="caution">CAUTION</span>|
|:---|
|只有設定檔裡 `parallel = true` 時，<mark>才會連 `.coverage.*` 平行資料檔一起刪</mark>。<br>只在命令列下 `-p` 的話，`erase` 後殘留的平行資料檔，<mark>會在下次報告時被自動合併進來，污染結果</mark>。|

|📗 <span class="tip">TIP</span>|
|:---|
|那要怎麼避免呢？<br>需要平行模式時，<mark>把 `parallel = true` 寫進設定檔</mark>，而不是只在命令列加 `-p`。|

### `debug`：診斷資訊
> 報告看起來怪怪的時候 (設定沒生效、該量的檔案沒量到)，拿來找原因。

+ #### `config`
  > 印出實際生效的設定。<mark>確認設定檔有沒有被讀到</mark>最快的方法。
  ```
  coverage debug config
  ```
  ```
                      config_file: /path/to/demo/pyproject.toml
                config_files_read: /path/to/demo/pyproject.toml
  ```
+ #### `sys`
  > 版本、安裝位置、環境資訊。回報 bug 時附上。
  ```
  coverage debug sys
  ```
+ #### `data`
  > 資料檔摘要：量了哪些檔案、各幾行。7.13.1 起可直接帶檔名。
  ```
  coverage debug data .coverage
  ```



## Options

### `--data-file`：資料檔位置
> `run`、`combine`、各報告指令都吃。也可以用環境變數 `COVERAGE_FILE` 或設定檔的 `data_file`。\
> 目錄不存在會自動建立。
```
coverage run --data-file=.cov/unit -m pytest tests/unit
coverage report --data-file=.cov/unit
```

### `--rcfile`：指定設定檔
> 不走預設的搜尋順序 (詳見 [Configuration](#configuration))，直接指定設定檔。也可以用環境變數 `COVERAGE_RCFILE`。
```
coverage run --rcfile=ci/.coveragerc -m pytest
```

### `--include` / `--omit`：篩選檔案
> 用檔名 pattern 篩選。在 `run` 是「量測範圍」，在報告指令是「報告範圍」。\
> 多個 pattern 用逗號隔開，<mark>記得加引號</mark>。
```
coverage run --omit="tests/*,*/migrations/*" -m pytest
coverage report --include="src/mymath/*"
```
+ pattern 規則
  + `*`：任意字元，<mark>不跨目錄</mark> (例外：開頭的 `*/` 視同 `**/`、結尾的 `/*` 視同 `/**`)
  + `**`：任意層目錄 (包含零層)，必須是完整的一段路徑 (`/**/` 可以，`/a**/` 不行)
  + `?`：單一字元
  + 沒有目錄分隔符的 pattern：比對任何目錄下的檔名
  + `/` 和 `\` 互通，跨平台不用寫兩套
+ 開頭不是萬用字元的 pattern，以目前目錄為基準
+ `source` 和 `include` 同時設定時，`include` 會被忽略 (並發出警告)

### `--debug`：除錯日誌
> 想知道 coverage 內部在想什麼時用。也可以用環境變數 `COVERAGE_DEBUG`。多個值用逗號隔開。
+ `trace`：每個檔案為什麼被追蹤 / 不追蹤 (查「這個檔案怎麼沒被量到」)
  ```
  coverage run --debug=trace -m pytest
  ```
+ `config`：開跑前印出所有設定
+ `pathmap`：`combine` 時路徑如何被對應
+ `core`：選了哪個量測核心、為什麼

|📘 <span class="note">NOTE</span>|
|:---|
|除錯輸出預設到 stderr。測試框架會吃掉輸出時，用 `COVERAGE_DEBUG_FILE` 寫到檔案。|



## Configuration
> 詳見 [Coverage.py - Configuration reference](https://coverage.readthedocs.io/en/latest/config.html)

coverage 依序尋找以下設定檔，<mark>只採用第一個含有 coverage 設定的檔案</mark>：

| 順序 | 檔案 | 格式 | 區段寫法 |
| --- | --- | --- | --- |
| **1** | `.coveragerc` | INI | `[run]` |
| **2** | `.coveragerc.toml` (7.13+) | TOML | `[run]` 或 `[tool.coverage.run]` |
| **3** | `setup.cfg` | INI | `[coverage:run]` |
| **4** | `tox.ini` | INI | `[coverage:run]` |
| **5** | `pyproject.toml` | TOML | `[tool.coverage.run]` |

|🚨 <span class="caution">CAUTION</span>|
|:---|
|設定檔<mark>不會合併</mark>。專案裡同時有 `.coveragerc` 和 `pyproject.toml` 時，`pyproject.toml` 的 coverage 設定<mark>完全無效</mark>。<br>不確定讀到哪個時：`coverage debug config`。|

|🚨 <span class="caution">CAUTION</span>|
|:---|
|INI 格式<mark>只認整行註解</mark> (`#` 或 `;` 開頭)，不支援行尾註解。<br>寫成 `data_file = mydata ; 資料檔位置` 的話，值會變成 `mydata ; 資料檔位置`。|

|📗 <span class="tip">TIP</span>|
|:---|
|專案用 uv 管理時，直接寫在 `pyproject.toml`，少一個檔案|

### `tool.coverage.run`：量測設定
> 對應 [run](#run：收集資料) 的選項。`.coveragerc` 寫成 `[run]`，`setup.cfg` 寫成 `[coverage:run]`。
```toml
[tool.coverage.run]
branch = true
source = ["src"]
omit = ["*/__main__.py"]
parallel = true
relative_files = true
```
+ #### `branch`
  > 同 `--branch`。建議一律開。
+ #### `source` / `source_dirs` / `source_pkgs`
  > 同 `--source`，寫在這裡所有指令都看得到。\
  > `source` 會自己猜是目錄還是套件；想講清楚就改用 `source_dirs` (7.8+，目錄不存在會直接報錯) 或 `source_pkgs`。
+ #### `omit` / `include`
  > 同命令列的 [篩選檔案](#options) 選項。報告時想另外篩選，可寫在 `[tool.coverage.report]`。
+ #### `data_file`
  > 資料檔位置 (預設 `.coverage`)，可以包含其他目錄。
+ #### `parallel`
  > 同 `--parallel-mode`。寫在這裡，`coverage erase` 才會連平行資料檔一起清 (詳見 [erase](#erase：清除資料))。
+ #### `relative_files`
  > 資料檔改存相對路徑。在 A 環境量測、B 環境出報告時很方便 (例如 CI 收集、本機看報告)。
  |📘 <span class="note">NOTE</span>|
  |:---|
  |要搭配<mark>寫在設定檔裡的 `source`</mark> 才會正確運作，因為報告指令需要知道原始碼的根在哪|
+ #### `patch`
  > 詳見 [subprocess](#subprocess)
+ #### `dynamic_context`
  > 詳見 [contexts](#contexts)
+ #### `core`
  > 量測核心。通常不用設，coverage 會依 Python 版本自動選 (也可用環境變數 `COVERAGE_CORE`)。

  | 核心 | 實作 | 說明 |
  | --- | --- | --- |
  | **`ctrace`** | C 寫的 `sys.settrace` trace function | Python 3.13 以前的預設 |
  | **`sysmon`** | `sys.monitoring` (PEP 669) | Python 3.14+ 的預設，開銷較低<br>不支援 plugin、dynamic context；3.12 / 3.13 上不支援 branch |
  | **`pytrace`** | 純 Python 的 trace function | 最慢，只在特殊情況用 (同 `--timid`) |

  |📘 <span class="note">NOTE</span>|
  |:---|
  |沒指定 `core` 時，如果其他設定和 `sysmon` 衝突 (例如開了 `dynamic_context`)，會<mark>自動退回 `ctrace`</mark>|

### `tool.coverage.report`：報告設定
> 對應 [report](#report：終端機報告) 的選項，也套用到 `html`、`xml` 等其他報告。`.coveragerc` 寫成 `[report]`。
```toml
[tool.coverage.report]
show_missing = true
skip_covered = true
skip_empty = true
fail_under = 80
precision = 1
exclude_also = [
    'if __name__ == .__main__.:',
    'raise NotImplementedError',
    '@(abc\.)?abstractmethod',
]
```
+ #### `show_missing` / `skip_covered` / `skip_empty` / `precision` / `sort`
  > 同命令列的 `--show-missing`、`--skip-covered`、`--skip-empty`、`--precision`、`--sort`
+ #### `fail_under`
  > 同 `--fail-under`。門檻有小數時，要一併設定 `precision`，小數位才會被納入判斷。
+ #### `exclude_also` / `exclude_lines`
  > 排除特定程式碼 (詳見 [exclusion](#exclusion))

### `tool.coverage.html`：HTML 報告設定
> 對應 [html](#html：html-報告) 的選項。`.coveragerc` 寫成 `[html]`。
```toml
[tool.coverage.html]
directory = "htmlcov"
title = "mymath coverage"
show_contexts = true
```
+ #### `skip_covered` / `skip_empty`
  > 沒寫的話沿用 `[tool.coverage.report]` 的設定；想讓 HTML 和終端機報告不同時才需要寫

### `tool.coverage.paths`：路徑對應
> 合併來自不同機器、不同目錄的資料時，告訴 coverage「這些路徑其實是同一份原始碼」。
```toml
[tool.coverage.paths]
source = [
    "src/",
    "/jenkins/build/*/src",
    "c:\\myproj\\src",
]
```
+ 項目名稱 (這裡的 `source`) 可以隨便取
+ <mark>第一個必須是本機實際存在的路徑</mark> (出報告時要從這裡讀原始碼)，其餘是要被對應過來的路徑或 pattern
+ 對應後的路徑真的存在，才會套用；可以寫多組，依序嘗試
+ 合併的路徑對不上時，用 `--debug=pathmap` 查



## Advanced

### exclusion
+ 參考
  + [**Excluding code from coverage.py**](https://coverage.readthedocs.io/en/latest/excluding.html)
+ 說明
  + 有些程式碼本來就不打算測 (除錯用、防禦性程式碼、型別檢查用)，可以從報告中排除
  + 加上 `# pragma: no cover` 的那一行會被排除；如果那行<mark>開啟了一個區塊</mark> (`if`、`def`、`class`)，<mark>整個區塊一起排除</mark>
  + 被排除的程式碼照樣會執行，只是報告不把它算成「沒測到」
+ 預設排除 (不用設定)
  + 行：帶有 `# pragma: no cover` 的行、只有 `...` 的行 (7.10+)
  + 分支：帶有 `# pragma: no branch` 的分支、`if True:` / `while True:` 這類編譯期就確定的分支、`if TYPE_CHECKING:` (7.10+)
+ 範例
  ```py
  class Point:
      def __init__(self, x, y):
          self.x = x
          self.y = y

      def __repr__(self):  # pragma: no cover
          return f"Point({self.x}, {self.y})"


  def handle(event):
      if event.kind == "click":
          return "clicked"
      else:  # pragma: no cover
          raise AssertionError("unreachable")
  ```
  + `__repr__` 的 `def` 行被排除 → 整個函式都不算
  + `else` 被排除 → `if` 只剩一條去路，<mark>不再算成分支</mark>，branch coverage 也不會抱怨
+ 配置
  > 常用的排除寫進設定檔，就不用到處加 pragma
  ```toml
  [tool.coverage.report]
  exclude_also = [
      'def __repr__',
      'if self\.debug',
      'raise AssertionError',
      'raise NotImplementedError',
      'if __name__ == .__main__.:',
      'class .*\bProtocol\):',
      '@(abc\.)?abstractmethod',
  ]
  ```
  + 多行 regex (7.6+)：用註解圈出一整段不量
    ```toml
    [tool.coverage.report]
    exclude_also = [
        'no cover: start(?s:.)*?no cover: stop',
    ]
    ```
    ```py
    # no cover: start
    def debug_dump():
        ...
    # no cover: stop
    ```
+ 注意
  |☢️ <span class="warn">WARNING</span>|
  |:---|
  |用 <mark>`exclude_also`</mark>，不要用 `exclude_lines`。<br>`exclude_lines` 會<mark>覆蓋掉預設規則</mark>，連 `# pragma: no cover` 都會失效。|

  |🚨 <span class="caution">CAUTION</span>|
  |:---|
  |規則是 regex，而且<mark>只要比對到一行中的一部分</mark>就算數。<br>寫 `...` 會排除所有三個字元以上的行；寫 `pass` 連 `my_pass = "foo"` 都會被排除。|

  |📗 <span class="tip">TIP</span>|
  |:---|
  |TOML 裡用<mark>單引號</mark> (literal string) 寫 regex，反斜線不用寫兩次|

### contexts
+ 參考
  + [**Measurement contexts**](https://coverage.readthedocs.io/en/latest/contexts.html)
+ 說明
  + 一般的報告只知道「這行有沒有被執行」，dynamic context 會再記下「<mark>這行是被哪個測試執行的</mark>」
  + 適合找出：某段程式碼到底有哪些測試在保護、某個測試實際覆蓋了哪些程式碼
+ 配置
  ```toml
  [tool.coverage.run]
  dynamic_context = "test_function"

  [tool.coverage.html]
  show_contexts = true
  ```
+ 操作
  + 照常量測
    ```
    coverage run -m pytest
    ```
  + HTML 報告中，每一行右側可展開看到 context 標籤 (例如 `test_core.test_clamp_positive`)
    ```
    coverage html --show-contexts
    ```
  + 只看某些測試貢獻的涵蓋率 (regex 比對 context 標籤)
    ```
    coverage report --contexts=clamp
    ```
+ 注意
  |🚨 <span class="caution">CAUTION</span>|
  |:---|
  |Python 3.14+ 預設的 `sysmon` 核心不支援 dynamic context。<br>用 coverage 設定檔的 `dynamic_context` 時，會自動退回 `ctrace`；<br>但用 pytest-cov 的 `--cov-context=test` 時，<mark>context 會被丟棄</mark> (7.15.3 起會發出警告)。|

  |📗 <span class="tip">TIP</span>|
  |:---|
  |那要怎麼解決呢？<br>在設定檔明確指定 <mark>`core = "ctrace"`</mark>，或改用 coverage 設定檔的 `dynamic_context`。|

### subprocess
+ 參考
  + [**Managing processes**](https://coverage.readthedocs.io/en/latest/subprocess.html)
+ 說明
  + 測試裡用 `subprocess`、`os.system()` 開出新的 Python 行程時，那個行程預設<mark>不會被量測</mark>
  + 7.10 起用 `patch` 設定就能解決，不必再自己處理 `.pth` 檔與 `COVERAGE_PROCESS_START`
+ 配置
  ```toml
  [tool.coverage.run]
  patch = ["subprocess"]
  ```
  + 可用的 patch
    + `subprocess`：量測 `subprocess`、`os.system()`、`execv` / `spawnv` 系列開出的 Python 子行程
    + `_exit`：程式用 `os._exit()` 強制結束時，仍先把資料存下來
    + `fork`：fork 後在子行程重新啟動量測 (內建處理不夠用時才需要)
    + `execv`：`execv` 系列換掉目前程式前先存資料 (Windows 不支援)
+ 注意
  + `patch = ["subprocess"]` 會隱含 `parallel = true`，每個子行程各寫一個資料檔，報告前要合併 (7.14 起報告指令會自動合併)
  + 子行程若是用 `exec*e` / `spawn*e` 這類自訂環境變數的函式開的，要把 `COVERAGE_PROCESS_START` 一起帶過去
  + pytest-cov 7.0 起移除了自己的子行程量測，改用這個機制 (詳見 [pytest-cov](#pytest-cov))



## Workflow

### basic
1. 先進到專案的頂層目錄
    > 設定檔與 `.coverage` 都以目前目錄為準
2. 運行測試程式
    > 會產生一個 `.coverage` 檔案
    + unittest 框架
        ```
        uv run coverage run -m unittest discover
        ```
    + pytest 框架
        ```
        uv run coverage run -m pytest
        ```
3. 涵蓋率
    > 根據 `.coverage` 所記錄的測試結果，顯示涵蓋率結果
    ```
    uv run coverage report -m
    ```
4. 產生精美 HTML 報告
    > 根據 `.coverage` 所記錄的測試結果，產生一個 `htmlcov` 的目錄，裡面包含測試結果的詳盡 HTML 文檔
    ```
    uv run coverage html
    ```

### multi-version
> 把 [uv 筆記的 multi-version 腳本](https://hackmd.io/@RogelioKG/uv#multi-version) 加上 coverage：\
> 每個 Python 版本各量一次，最後合併成一份報告，<mark>連「只在某個版本才會走到的分支」都算得到</mark>。

1. 在 `pyproject.toml` 開啟平行模式
    ```toml
    [tool.coverage.run]
    branch = true
    source = ["src"]
    parallel = true
    ```
2. 逐版本量測，最後出報告
    ```ps
    # PowerShell
    $ErrorActionPreference = "Stop"

    $pyVersions = @("3.14", "3.13", "3.12")

    # parallel = true 寫在設定檔，所以會連舊的平行資料檔一起清掉
    uv run coverage erase

    foreach ($v in $pyVersions) {
        Write-Host ">>> Testing with Python $v" -ForegroundColor Cyan

        uv run --python $v coverage run -m pytest

        if ($LASTEXITCODE -ne 0) {
            Write-Host "❌ Python $v tests failed`n" -ForegroundColor Red
            exit 1
        }
    }

    # 7.14 起報告指令會自動合併各版本的資料檔
    uv run coverage report -m
    uv run coverage html
    ```



## Run with

### pytest-cov
> pytest 外掛，把 `coverage run` + `coverage report` 濃縮成一個 pytest 指令。\
> 設定仍然讀 coverage 的設定檔 (`[tool.coverage.*]`)。

1. 安裝
    ```
    uv add pytest-cov --group dev
    ```
2. 執行
    ```
    uv run pytest --cov=src --cov-branch --cov-report=term-missing
    ```

| 想做的事 | coverage | pytest-cov |
| --- | --- | --- |
| **量測** | `coverage run -m pytest` | `pytest --cov` |
| **指定來源** | `--source=src` | `--cov=src` |
| **分支涵蓋率** | `--branch` | `--cov-branch` |
| **列出缺漏行** | `coverage report -m` | `--cov-report=term-missing` |
| **隱藏 100% 檔案** | `--skip-covered` | `--cov-report=term-missing:skip-covered` |
| **HTML 報告** | `coverage html` | `--cov-report=html` |
| **XML 報告** | `coverage xml` | `--cov-report=xml` |
| **門檻** | `--fail-under=80` | `--cov-fail-under=80` |
| **每個測試的 context** | `dynamic_context = "test_function"` | `--cov-context=test` |

|📘 <span class="note">NOTE</span>|
|:---|
|`--cov-report` 可以重複指定，一次輸出多種報告；寫成 `--cov-report=html:DIR` 可指定輸出位置|

|🚨 <span class="caution">CAUTION</span>|
|:---|
|pytest-cov 7.0 起<mark>不再自己量測子行程</mark>，測試裡有開子行程的話，要在 coverage 設定檔加上 `patch = ["subprocess"]` (詳見 [subprocess](#subprocess))|

### GitHub Actions
> 把涵蓋率表格貼到 workflow 的 job summary，再用門檻擋 PR。\
> (以下只列 coverage 相關的 step，假設前面已裝好 uv 與 Python，runner 為 ubuntu)
```yaml
- name: Test with coverage
  run: uv run coverage run -m pytest

- name: Coverage summary
  run: uv run coverage report --format=markdown >> "$GITHUB_STEP_SUMMARY"

- name: Coverage gate
  run: uv run coverage report --fail-under=80
```
|📗 <span class="tip">TIP</span>|
|:---|
|門檻放在最後一步單獨檢查。<br>若把 `fail_under` 寫進設定檔，summary 那一步就會先失敗，而且失敗訊息 `Coverage failure: ...` 是印在 stdout，會混進 summary 表格。|
