# llm-testcase-service 接口自动化：两种范式对照

同一个被测系统 llm-testcase-service（FastAPI，6 个接口，含原 README 漏写的 `GET /` 健康检查），本仓库用两种范式做接口自动化：

- A 线：手写 pytest 18 条，在独立仓库 `mumu1030/llm-testcase-service`，不拷代码进来。
- B 线：本仓库 6 个 Skill 组成管线：文档 → 场景 → 配置 → 用例 → 执行 → 报告。

核心不是「我会用 AI」，而是「两种范式对照 + 人工评审在哪里不可替代」。

## 1. 双线对照表

| 维度 | A 线：手写 pytest | B 线：Skill 管线 |
| --- | --- | --- |
| 被测接口 | 6 个（含 `GET /`） | 6 个（含 `GET /`） |
| 用例规模 | 18 条手写 pytest | 25 条场景 / 25 条 cases / 18 条启用 |
| 禁用项 | 无 | case 2/3/7/8/15/18/25 的 `enabled:false`，会真调 DeepSeek |
| 执行结果 | 18 条通过（原仓库） | 18 passed in 0.07s，failed=0 |
| 报告指标 | 原仓库自管 | 18 passed / 100% / duration=51ms |
| 产物形态 | pytest 代码 | 文档、场景、配置、用例、结果与报告 |

## 2. 六环链 + 七件产物表

六环链：`api-doc-generator` → `api-scenario-generator` → `api-test-config` → `api-test-generator` → `api-test-runner` → `api-test-report`。  
六件套脚本靠自身位置推项目根，必须留在 `skills/` 里。

| # | 环节 | 产物路径 |
| --- | --- | --- |
| 1 | 文档扫描 | `docs/api/api-docs-llm-testcase-service.md` |
| 2 | 场景 JSON | `docs/llm_testcase_scenario.json` |
| 3 | 场景评审表 | `docs/llm_testcase_scenario.xlsx` |
| 4 | 可执行用例 JSON | `docs/llm_testcase_cases.json` |
| 5 | 可执行用例表 | `docs/llm_testcase_cases.xlsx` |
| 6 | 报告截图 | `docs/allure-overview.png` |
| 7 | 环境配置（不进库） | `config/.env` |

运行产物（不进库）：`allure-results/`、`allure-report/`。

## 3. 报告截图

![](docs/allure-overview.png)

截图来自 `allure-report/` 的 Overview：18 passed / 100% / 51ms，failed=0，不是白屏或空图表。

## 4. 人工评审拦下了什么

- 场景 7/8 原按文档写 422；读 `models.py` 发现 `platform` / `test_types` 无枚举校验，实际返回 200，于是把预期改为按实现回显。
- 场景 18 文档写 `text/plain` 是笔误；实际 FastAPI 返回 `application/json`，原服务 `r.json()` 得到 Markdown 字符串可证，用例按 JSON 字符串建模而非纯文本。

## 5. 已知工具边界与人的裁决

1. scenario 3 条 WARN（2 条服务无 PUT/PATCH、1 条 `GET /` 不合 `/api/v1` 命名）判为误报：前者是服务本就没有更新接口，后者是健康检查路径，不是资源命名遗漏。
2. case 13「数量 N→N+1」降级为可访问性校验：runner 无 `len()`，list 根取不到值，最终用「列表可访问 + 生成后详情可访问」表达新增语义。
3. 导出步骤 `checks` 必须为空：响应体是 JSON 字符串非对象，任何 `$.xxx` 都取 `None`，因此只断言 HTTP 200，内容校验落在同 case 的详情步骤。
4. Suites 显示 `skills.api-test-runner.scripts`、Behaviors 为空：要修需改六件套脚本；为保「clone 即可复跑」选择接受现状。

## 6. 复跑指引

1. 先在 8000 端口起服务，不要起在 8080：

   ```bash
   cd /Users/songmumu/llm-testcase-service && uvicorn main:app --reload
   ```

2. `config/.env` 被 `.gitignore` 排除，clone 后需自建；至少包含：

   ```env
   BASE_URL=http://127.0.0.1:8000
   DB_ASSERT_ENABLED=false
   ```

   `DB_HOST` / `DB_NAME` / `DB_USER` / `DB_PASS` 留空即可，服务是内存库，无 DB 断言。

3. 六步命令按 `skills/` 下各 `SKILL.md` 执行：

   ```bash
   # 文档：见 skills/api-doc-generator/SKILL.md
   # 场景：见 skills/api-scenario-generator/SKILL.md
   python3 skills/api-scenario-generator/scripts/run_scenario_pipeline.py -n llm_testcase

   # 配置：见 skills/api-test-config/SKILL.md，自建 config/.env

   # 用例：见 skills/api-test-generator/SKILL.md，生成后导出 Excel
   python3 skills/api-test-generator/scripts/plan_json_to_excel.py -i docs/llm_testcase_cases.json -o docs/llm_testcase_cases.xlsx

   # 执行：见 skills/api-test-runner/SKILL.md，不传 --base-url
   python3 skills/api-test-runner/scripts/run_tests.py -c docs/llm_testcase_cases.json

   # 报告：见 skills/api-test-report/SKILL.md，--no-open 只生成不弹浏览器
   python3 skills/api-test-report/scripts/serve_report.py --no-open
   ```

4. allure CLI 需自行安装；若缺失，按 `skills/api-test-report/references/install.md` 处理（macOS 可用 `brew install allure`）。

待补：CI 徽章、在线报告链接、Star 数、手写与 Skill 管线的性能对比（未做同环境基准，不编造）。
