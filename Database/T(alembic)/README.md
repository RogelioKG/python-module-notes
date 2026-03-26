# Alembic
[![RogelioKG/alembic](https://img.shields.io/badge/Sync%20with%20HackMD-grey?logo=markdown)](https://hackmd.io/@RogelioKG/alembic)

## References
+ 📑 [**Documentation - Alembic**](https://alembic.sqlalchemy.org/en/latest/)


## Convince Me

|📘 <span class="note">NOTE</span> : Alembic|
|:---|
| Alembic 是基於 SQLAlchemy 的資料庫遷移工具<br><span style="color: gray;">(不支援非 SQLAlchemy 的專案)<span> |

### Why?
+ **遠離手寫遷移 SQL**
  + <mark>能比對 ORM Model 與 DB 差異，並自動生成遷移 SQL</mark>，徹底避免手寫遷移 SQL 的風險
+ **Schema 版控**
  + 將資料庫演進史轉化為可追蹤的遷移腳本 (Migration Scripts)
+ **可逆性**
  + 支援 Upgrade / Downgrade 雙向遷移，<mark>Schema 變更出錯可快速 Rollback</mark>
+ **環境一致性**
  + 確保 Local、Staging、Production 的 Schema 絕對同步

### When?
+ **Day 1 導入**
  + 確立使用 RDBMS + SQLAlchemy 時即刻引入，避免後期產生巨大的技術債
+ **Schema 變更**
  + SQL 操作：增刪改 Table、Column、Index、Constraints (Unique/Foreign Key) 等等
  + 資料型別轉換：`String` -> `Text` 等等
+ **CI/CD**
  + 整合於 Pipeline，在啟動 Application Server 前執行自動更新 (`upgrade head`)，確保 DB 結構與最新版 API 程式碼相匹配

### How?

#### Phase 1 - 基礎配置
```bash
uv run alembic init alembic
```
+ **`alembic.ini`**：設定 `sqlalchemy.url` (設定 DB 連線 URL)
+ **`alembic/env.py`**：設定 `target_metadata = Base.metadata` (綁定 SQLAlchemy Models)

#### Phase 2 - 開發 SOP

| 步驟 | 動作 | 指令 / 實務說明 |
| :--- | :--- | :--- |
| 1. **Modify** | 修改 Model | 更改程式碼中的 Model 定義<br><span style="color: gray;">(例如：新增 `nickname = Column(String)`)</span> |
| 2. **Generate** | 產生遷移腳本 | `uv run alembic revision --autogenerate -m "add nickname"`<br><span style="color: gray;">(Alembic 會比對 Model 與 DB，於 `alembic/versions/` 目錄生成對應的升/降版腳本)</span> |
| 3. **Apply** | 套用變更 | `uv run alembic upgrade head`<br><span style="color: gray;">(將變更正式寫入 DB，`head` 代表更新至最新版本)</span> |

#### Phase 3 - 復原 Rollback

```bash
# 退回前一個版本
uv run alembic downgrade -1

# 退回指定版本 (利用版本號 Hash)
uv run alembic downgrade <revision_id>
```

💡 **Rollback 善後最佳實務**：執行 Downgrade 之後，記得將 `models.py` 中寫錯的程式碼改回來，並**手動刪除：versions/` 裡那支作廢的腳本，確保程式碼與資料庫狀態乾淨一致


## Dir Structure
```py
project/
├── alembic.ini          # 主要設定檔
└── alembic/
    ├── env.py           # 執行環境設定
    ├── README
    ├── script.py.mako   # 遷移腳本的模板
    └── versions/        # 所有版本遷移腳本
```

## Config

### `alembic.ini`
+ 內容
  ```ini
  # Alembic 遷移環境的基本行為
  [alembic]
  script_location = %(here)s/alembic
  prepend_sys_path = .
  # timezone =
  # truncate_slug_length = 40
  # revision_environment = false
  # sourceless = false
  # version_locations = %(here)s/bar:%(here)s/bat:%(here)s/alembic/versions
  # path_separator = :
  # path_separator = ;
  # path_separator = space
  # path_separator = newline
  path_separator = os
  # recursive_version_locations = false
  # output_encoding = utf-8
  sqlalchemy.url = driver://user:pass@localhost/dbname

  # 當 Alembic 自動產生新的遷移腳本後，要自動觸發執行的外部工具或 Python 腳本
  [post_write_hooks]
  # hooks = ruff
  # ruff.type = module
  # ruff.module = ruff
  # ruff.options = check --fix REVISION_SCRIPT_FILENAME

  # 日誌設定
  [loggers]
  keys = root,sqlalchemy,alembic

  [handlers]
  keys = console

  [formatters]
  keys = generic

  [logger_root]
  level = WARNING
  handlers = console
  qualname =

  [logger_sqlalchemy]
  level = WARNING
  handlers =
  qualname = sqlalchemy.engine

  [logger_alembic]
  level = INFO
  handlers =
  qualname = alembic

  [handler_console]
  class = StreamHandler
  args = (sys.stderr,)
  level = NOTSET
  formatter = generic

  [formatter_generic]
  format = %(levelname)-5.5s [%(name)s] %(message)s
  datefmt = %H:%M:%S
  ```

### `alembic/env.py`
+ 內容
  ```python
  from logging.config import fileConfig

  from sqlalchemy import engine_from_config
  from sqlalchemy import pool
  from alembic import context


  # alembic.ini 設置
  config = context.config

  if config.config_file_name is not None:
      fileConfig(config.config_file_name)

  # ✅ 此處須提供資料庫 metadata (為了支援 autogenerate)
  from app import model
  target_metadata = model.Base.metadata 

  def run_migrations_offline() -> None:
      url = config.get_main_option("sqlalchemy.url")
      context.configure(
          url=url,
          target_metadata=target_metadata,
          literal_binds=True,
          dialect_opts={"paramstyle": "named"},
      )

      with context.begin_transaction():
          context.run_migrations()


  def run_migrations_online() -> None:
      connectable = engine_from_config(
          config.get_section(config.config_ini_section, {}),
          prefix="sqlalchemy.",
          poolclass=pool.NullPool,
      )

      with connectable.connect() as connection:
          context.configure(
              connection=connection, target_metadata=target_metadata
          )

          with context.begin_transaction():
              context.run_migrations()


  if context.is_offline_mode():
      run_migrations_offline()
  else:
      run_migrations_online()
  ```

+ config
  + `config.get_main_option()` / `config.set_main_option()`
    + 說明
      + `alembic.ini` 中的 `[alembic]` 區段的某個設定
    + 範例
      ```python
      config.get_main_option("sqlalchemy.url")
      config.set_main_option("sqlalchemy.url", "postgresql://user:pass@localhost/db")
      ```
  + `config.get_section_option()` / `config.set_section_option()`
    + 說明
      + `alembic.ini` 中的非 `[alembic]` 區段的某個設定
    + 範例
      ```python
      config.get_section_option("my_custom_section", "some_key", "default")
      config.set_section_option("my_custom_section", "some_key", "some_value")
      ```
  + `config.get_section()`
    + 區段
      + `alembic.ini` 中的某個區段的所有設定
    + 範例
      ```python
      config.get_section("alembic", {})
      ```
+ online
  | 特性 | Online Migration (`run_migrations_online`) | Offline Migration (`run_migrations_offline`) |
  | :--- | :--- | :--- |
  | **資料庫連線** | 必須成功連線 | 不需要建立連線 (只需知道 URL) |
  | **執行結果** | 直接修改資料庫的 Table 結構 | 僅產生 SQL 語句 |
  | **觸發指令** | `alembic upgrade head` | `alembic upgrade head --sql` |
  | **主要用途** | 自動化直接更新資料庫 | 產出 SQL 腳本供 DBA 審核或手動部署 |


## Commands

### `init`：初始化環境
> 建立 Alembic 的設定檔與目錄架構。
```bash
alembic init alembic
```

### `revision`：建立遷移腳本
+ #### `--autogenerate`
  > 自動比對並生成腳本。
  ```bash
  alembic revision --autogenerate -m "Add nickname column"
  ```
  |🚨 <span class="caution">CAUTION</span>|
  |:---|
  |  `--autogenerate` 並非萬能！它無法偵測：<br>1. 欄位名稱修改<br>- 它會看成刪除舊欄位並新增新欄位<br>2. 匿名約束 (Anonymous Constraints)<br>- <mark>生成後請務必人工檢查 `alembic/versions/` 下的腳本內容！</mark> |

### `upgrade`：資料庫升版
+ #### `head`
  > 升級到最新版本。
+ #### `<revision_id>`
  > 升級到特定版本。
+ #### `+1`
  > 往後跳一版。

### `downgrade`：資料庫降版
+ #### `-1`
  > 退回上一版。
+ #### `base`
  > 退回到最初狀態。

### `current`：查看當前版本

### `history`：查看遷移歷史
