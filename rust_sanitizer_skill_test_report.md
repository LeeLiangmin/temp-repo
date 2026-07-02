# Rust Sanitizer 动态错误检测 Skill 测试报告

**Skill 版本**: v1.2.0
**测试日期**: 2026-07-01
**测试环境**: Ubuntu 24.04.3 LTS, x86_64, cargo 1.93.1, nightly 1.98.0, python3 3.12.3
**受测项目**: `ylong_json`（含项目级检查与单文件重定向）、`demo_proc_macro`（构造环境）

---

## 1. 执行摘要

| 维度 | 结果 |
|------|------|
| **整体结论** | 通过（核心工作流完整，关键行为符合 Skill 规范） |
| **覆盖场景** | 全量通过（ylong_json ×2、demo_proc_macro s1）、缺陷发现（demo_proc_macro s2 ASan heap-buffer-overflow）、TSan 按需启用、单文件无入口重定向 |
| **阻塞项** | 无 |
| **最终评估** | 可用，具备生产级执行能力；存在若干改进建议（参见第 4 节） |

---

## 2. 测试覆盖矩阵

### 2.1 工作流步骤覆盖

| Step | 描述 | ylong_json (项目级) | ylong_json (单文件) | demo_proc_macro | 结论 |
|------|------|:---:|:---:|:---:|------|
| 0 — 意图路由 | 区分答疑 / 检查 / 意图不明 | PASS | PASS | PASS | 均先询问用户意图后进入对应模式 |
| 1 — 环境确认 | Linux 检测与目标确认 | PASS | PASS | PASS | 正确识别 Ubuntu，定位 Cargo.toml |
| 1a — 单文件无入口 | `.rs` 文件无 `main` 或无 `#[test]` | N/A | PASS | N/A | 拒绝直接运行；通过 question tool 提供结构化选项 |
| 2 — 工具检查 | 必要工具链逐项检测 | PASS | PASS | PASS | 全部就绪 |
| 2 — cargo-geiger | 可选 unsafe 风险筛查 | PASS | PASS | PASS | 缺失时不阻塞工作流 |
| 3 — cargo geiger | unsafe 分布扫描 | PASS | PASS | PASS | 正确输出 unsafe 表达式数量及来源文件 |
| 4 — 配置准备 | 模板复制 / 复用 → 字段说明 → 等待确认 | PASS | PASS | PASS | 项目级复用已有配置；单文件 session 新建配置并正确指定 `--test` |
| 5 — 执行检查 | ASan + LSan 编译运行，TSan 按需独立运行 | PASS | PASS | PASS | TOML 字段到 CLI 参数映射准确 |
| 6 — 报告生成 | `generate-report.py` 调用 | PASS | PASS | PASS | 未出现手工拼接报告正文的行为 |
| 最终交付 | 结构化摘要输出 | PASS | PASS | PASS | 结论 / 范围 / Sanitizer / 路径等字段完整 |

### 2.2 场景覆盖

| 场景 | 受测项目 | 结果 | 说明 |
|------|----------|------|------|
| 全量通过（无 sanitizer 问题） | ylong_json（项目级 ×2）、demo_proc_macro（s1） | PASS | 报告正确标注"未发现问题"及未验证项 |
| 缺陷发现（ASan heap-buffer-overflow） | demo_proc_macro（s2） | PASS | 正确检测并写入 ASan runtime 日志；正确区分退出码 1 为 sanitizer 检测结果而非编译或环境错误 |
| 单文件无入口重定向 | ylong_json（lib.rs → sdv_adapt_serde_test） | PASS | 拒绝运行无 main 的 lib.rs；question tool 提供 3 个结构化选项；用户选择测试文件后正确执行 |
| TSan 按需启用 | ylong_json（checklist 第 19 条） | PASS | 仅在用户明确要求后编辑 config.toml 并独立运行 |
| cargo-geiger 缺失不阻塞 | checklist 第 11 条 | PASS | 记录风险筛查未完成并继续 |
| 复用已有 config.toml | ylong_json（项目级）、demo_proc_macro（s2） | PASS | 验证已有配置有效性后直接复用 |

### 2.3 明显不符合预期的行为

全部风险行为均**未发生**：未在非 Linux 环境执行、未在用户确认前运行检查、未将 config.toml 视为 cargo/rustc 自动加载的参数文件、未手工编写报告正文、未自行修改源码、未越权启用 TSan 或其他 sanitizer。

---

## 3. 关键行为分析

### 3.1 核心问题：多错误检出效率不足

#### 现象

`halt_on_error=0:abort_on_error=0` 的配置意图是使 ASan 在首个错误后继续收集后续问题。然而在 demo_proc_macro 的 ASan 检测中，3 个均含 sanitizer 缺陷的单元测试仅捕获到首条 `heap-buffer-overflow`，其余两个缺陷（`use-after-free`、未初始化内存读取）未被上报。

#### 根因

1. **WRITE 型错误（heap-buffer-overflow）已实际破坏内存** — ASan 可标记继续执行，但被破坏的栈帧或堆元数据导致进程后续运行不可预测地崩溃。
2. **test runner 的 unwind 机制** — 测试函数 panic 或进程收到致命信号后，test runner 整体退出，后续测试不再调度。

#### 影响

单次 `cargo test` 调用在该类场景下最多捕获一条 sanitizer 问题，严重制约单轮检查的缺陷发现效率。

#### Skill 改造建议

在 `SKILL.md` Step 5 中增加子步骤：

```markdown
### 5a — 多测试逐进程隔离（首轮退出码 != 0 且 halt_on_error=0 时）

当 halt_on_error=0 但 cargo test 进程仍然提前退出时，
说明存在致命错误导致测试无法继续。此时执行：

1. cargo test -- --list 获取全部测试名
2. 逐个用 cargo test -- <test_name> 在独立进程中执行
3. 每个测试使用 log_path=<sanitizer>-<test_name> 区分日志
4. 汇总所有测试的退出码、原始输出和 runtime 日志
```


## 4. 建议汇总

### 4.1 对 Skill 设计的建议

| 编号 | 建议 | 优先级 |
|------|------|:---:|
| S1 | Step 5 增加首轮错误后逐测试独立进程执行策略（参见 3.1） | 高 |

### 4.2 未覆盖的边界场景

| 编号 | 场景 | 状态 |
|------|------|:---:|
| T1 | 大量问题场景下的聚合去重与 `max_inline_items` 截断 | 未测试 |

---

## 5. 结论

`rust-sanitizer` skill v1.2.0 在多轮完整工作流测试中，核心路径（环境检测、工具链检查、cargo geiger 风险筛查、配置管理、ASan/LSan/TSan 多轮执行、报告脚本调用）均按规范运作。缺陷为单次 `cargo test` 在遇到致命 sanitizer 错误后进程退出。

**综合评价：通过。**
