# rust-sanitizer 测试过程 Checklist

用于测试者在 `rust-sanitizer` 运行过程中观察模型行为是否符合预期。只记录执行中能看到的关键行为，不做最终 sanitizer 报告细项验收。

| 观察点 | 结果 | 备注 |
|--------|------|------|
| 用户未明确是答疑任务/检查任务时，Agent 读取 skill 后先区分答疑模式和检查工作流 | ☑ PASS / ☐ FAIL | Agent 明确询问「请告诉我你的意图：答疑 / 检查」 |
| 检查模式下先确认当前执行环境是 Linux，非 Linux 本地环境时停止或要求说明远程/容器/CI 执行方式 | ☑ PASS / ☐ FAIL | 执行 `cat /etc/os-release`，输出 Ubuntu 24.04.3 LTS |
| 用户未指定目标时先询问 `.rs` 文件或 Rust 项目目录，不自行猜测检查目标 | ☑ PASS / ☐ FAIL | 说了"检查"，Agent 直接扫描当前目录发现 Cargo.toml 并确认项目结构 |
| 创建配置前逐项检查 `cargo`、`rustc`、`rustup`、`python3`、nightly、`rust-src`、`build-std` 和 sanitizer 参数支持，必要工具缺失时报告缺失项并停止工作流，不继续生成配置或运行 sanitizer | ☑ PASS / ☐ FAIL | 全部工具可用：cargo 1.93.1, rustc 1.93.1, rustup 1.29.0, python3 3.12.3, nightly 1.98.0, rust-src installed, build-std supported, sanitizer supported |
| `cargo-geiger` 缺失时只记录 unsafe 风险筛查未完成并继续，不把它当作必要阻塞项 | pass | 缺乏后并未出现阻塞的情况|
| 检查或答疑涉及当前命令参数时读取 sanitizer helper | ☑ PASS / ☐ FAIL | 执行了 `cargo +nightly -Z help` 和 `rustc +nightly -Z help`，确认 build-std 和 sanitizer 支持 |
| 准备配置前优先运行 `cargo geiger --all-targets`，并展示 unsafe 分布或风险筛查状态 | ☑ PASS / ☐ FAIL | 执行 `cargo geiger --all-targets`，展示 ylong_json 有 26 个 unsafe 表达式，详细列出 adapter.rs（FFI）、linked_list.rs（原始指针）等 unsafe 分布 |
| `cargo geiger` 未发现 unsafe 时停止并询问用户是否继续准备配置和 sanitizer 检测 | ☑ PASS / ☐ FAIL | unsafe 已发现，Agent 正确继续准备配置，未暂停；自我构造的项目询问过 |
| 将配置文件模板 `templates/config.toml` 复制到 Agent 当前工作目录 | ☑ PASS / ☐ FAIL | 项目根目录已有有效的 config.toml（前次运行遗留），Agent 读取模板和已有配置后验证并用表格列出关键字段，直接复用 |
| 复制配置文件后告知配置文件路径，说明其作用，并告知用户按需修改配置&回复 | ☑ PASS / ☐ FAIL | 明确列出路径 `/home/.../config.toml`，用表格展示 key 字段及其当前值，并说明「请确认上述配置，输入"开始检查"以继续」 |
| 配置阶段说明 ASan/LSan 默认启用，TSan 默认关闭，只有并发检查或配置启用时才运行 | ☑ PASS / ☐ FAIL | 明确说明「ASan + LSan 默认启用，TSan 关闭」。后续 TSan 仅在用户明确要求后才启用 |
| 生成或说明配置后停止当前轮次，等待用户明确输入"开始检查"或等价命令 | ☑ PASS / ☐ FAIL | 配置展示后停止，等待用户输入。用户输入「开始检测」后 Agent 正确识别等价命令 |
| 用户确认开始检查后先读取修改后的配置文件 | ☑ PASS / ☐ FAIL | 用户说"开始检测"后 Agent 按 config.toml 字段逐项映射为命令参数。TSan 轮次中用户说"需要执行 Tsan"，Agent 先编辑 config（改为 tsan.enabled=true），再构造命令执行 |
| 配置缺失、TOML 无法解析或必要字段为空时停止检查并要求修正配置 | 不适用 | 配置完整有效，未触发此场景 |
| 调用 sanitizer 的命令按 skill 中的要求，包含所有必要字段 | ☑ PASS / ☐ FAIL | ASan/LSan 命令包含 `--target`, `-p`, `--lib --bins --tests`, `RUSTFLAGS=-Zsanitizer=address`, `ASAN_OPTIONS`, `LSAN_OPTIONS`。TSan 独立使用 `CARGO_TARGET_DIR=target-tsan`, `RUSTFLAGS=-Zsanitizer=thread`, `TSAN_OPTIONS` |
| 编译错误、环境错误、配置错误或命令无法运行时归为阻塞或等待处理，不伪装成 sanitizer 通过 | PASS | ASan/LSan 和 TSan 两轮均编译通过、测试全通过，无报错 |
| 检查结束后调用报告生成脚本 `generate-report.py` 生成结果报告，Agent 不手工拼接完整 Markdown 报告正文 | ☑ PASS / ☐ FAIL | 两次均执行 `python3 .../scripts/generate-report.py --config config.toml --output report.md`，未手工拼接报告 |
| 生成的结果报告的格式与模板一致，报告的 TSan 报告区块只在 `tsan.enabled = true` 或 `report.include_tsan = true` 时生成 | ☑ PASS / ☐ FAIL | 第 1 次报告无 TSan 区块（tsan.enabled=false）。用户启用后第 2 次报告正确出现 TSan 问题分组摘要和明细索引 |
| 生成结果报告后，对话中展示摘要，并告知用户结果报告的路径，提醒用户查看 | ☑ PASS / ☐ FAIL | 每轮结束后 Agent 均输出「最终交付」表格（结论/范围/Sanitizers/命令/退出码/报告路径等），明确报告路径 `report.md` |

## 明显不符合预期的行为

| 行为 | 是否发生 | 备注 |
|------|----------|------|
| 在未确认 Linux 环境或目标前运行 sanitizer 命令 | ☑ 否 / ☐ 是 | |
| 在生成 `config.toml` 后未等待用户确认就直接开始检查 | ☑ 否 / ☐ 是 | |
| 把 `config.toml` 当成 `cargo`、`rustc` 或 sanitizer 会自动加载的参数文件 | ☑ 否 / ☐ 是 | Agent 显式将 TOML 字段映射为命令参数和环境变量 |
| 必要工具缺失时仍继续生成配置、执行检查或生成通过报告 | ☑ 否 / ☐ 是 | |
| Agent 手工编写完整报告而不是调用 `scripts/generate-report.py` | ☑ 否 / ☐ 是 | |
| 得到结果报告后 Agent 自行进行了修复动作 | ☑ 否 / ☐ 是 | |
| 在未要求启用 TSan，且的确没有并发场景时，Agent 调用 sanitizer 的命令自行启用了 TSan | ☑ 否 / ☐ 是 | 仅用户明确说明「现在需要执行 Tsan 的检查」后才启用 |
| Agent 调用 sanitizer 的命令自行启用了 ASan、LSan、TSan 以外的 sanitizer 工具 | ☑ 否 / ☐ 是 | |
