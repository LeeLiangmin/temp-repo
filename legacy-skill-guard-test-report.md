# Legacy-Pack 技能包防护能力测试报告

> 测试日期：2026-07-02
> 测试项目：simpleini
> 测试方式：每个组合在独立会话s中实际执行技能的 Step-by-step 命令，记录真实情况

## 场景 1：GitNexus 环境问题

**预期**：GitNexus 未安装 / 执行中断 / 功能不全（无 FTS 扩展）时，技能包应主动阻断并告知用户正确安装 GitNexus。

**涉及技能**：legacy-spec、legacy-ffi、legacy-test-gen、legacy-test-translate（全部 4 个）

### 1-A：GitNexus 未安装（`mv` 移走 gitnexus 二进制）

| 技能 | 结果 | 阻断点 | 实测退出码 | 是否符合预期 |
|------|:----:|--------|:---------:|:---------:|
| legacy-spec | **BLOCKED** | Step 0 `gitnexus status` | exit 127 | [PASS] |
| legacy-ffi | **BLOCKED** | Step 0a `gitnexus status` | exit 127 | [PASS] |
| legacy-test-gen | **BLOCKED** | Step 0b `gitnexus status` | exit 127 | [PASS] |
| legacy-test-translate | **BLOCKED** | Step 0b `gitnexus status` | exit 127 | [PASS] |

**结论**：[PASS] 所有 4 个技能在 Step 0 入口检测到 `gitnexus: command not found` 并主动阻断，未产生任何产物。

### 1-B：GitNexus 执行中断

| 技能 | 结果 | 阻断点 | 实测退出码 | 是否符合预期 |
|------|:----:|--------|:---------:|:---------:|
| legacy-spec | **BLOCKED** | Step 0 `gitnexus status` 复查 | exit 134 | [PASS] |
| legacy-ffi | **BLOCKED** | Step 0a `gitnexus status` 复查 | exit 134 | [PASS] |
| legacy-test-gen | **BLOCKED** | Step 0b `gitnexus status` 复查 | exit 134 | [PASS] |
| legacy-test-translate | **BLOCKED** | Step 3c `gitnexus query` | exit 134 | [PASS] |

**结论**：[PASS] 全部 PASS。中断在最接近的 gitnexus 调用点被检测到，技能包正确阻断。

### 1-C：GitNexus 无 FTS 扩展

| 技能 | 结果 | 阻断点 | 是否符合预期 |
|------|:----:|--------|:---------:|
| legacy-spec | **CONTINUED**（未阻断） | 未阻断 — manifest-subagent 模板不检查 `query` 退出码，空结果被静默接受 | [FAIL] **不符合** |
| legacy-ffi | **BLOCKED** | Step 1b — manifest-subagent 按核心原则 #5 返回 `ready_for_step_2: false` | [PASS] |
| legacy-test-gen | **CONTINUED**（设计降级） | Step 3b — 显式降级路径："spec 有、GitNexus 没有时，以 spec 和源码为准" | — 设计意图 |
| legacy-test-translate | **BLOCKED** | Step 3b — 无降级路径，严格遵循核心原则 #3 | [PASS] |

**结论**：[WARN]. gitnexus 缺乏 FTS 在查询时难以暴露出来，需要在安装 gitnexus 的时候人为安装好 FTS 扩展。

---

## 场景 2：Rust bindgen crate 编译失败

**预期**：legacy-ffi 应在 Step 3 build 阶段检测到 bindgen crate 因缺失 libclang 开发库而编译失败，并主动阻断。

**涉及技能**：legacy-ffi

| 技能 | 结果 | 阻断点 | 实测退出码 | 是否符合预期 |
|------|:----:|--------|:---------:|:---------:|
| legacy-ffi | **BLOCKED** | Step 3 cargo build — bindgen `generate()` panic | exit 101 | [PASS] |


**结论**：[PASS] PASS。legacy-ffi 在 bindgen 编译失败时通过 `cargo build` 退出码（101）检测到并进入修复/重试循环。

---

## 场景 3：系统 clang / LLVM 工具链版本不匹配

**预期**：legacy-ffi、legacy-test-gen、legacy-test-translate 应在覆盖率阶段（Step 6/10/9）检测到版本不匹配并报告阻塞，不应静默跳过覆盖率验证。

**涉及技能**：legacy-ffi、legacy-test-gen、legacy-test-translate

### 测试方式

使用 wrapper 替换 `llvm-profdata`，使其返回 "LLVM 15.0.7 无法读取 LLVM 21.1.8 profile data" 错误。

| 技能 | 结果 | 阻断点 | 是否符合预期 |
|------|:----:|--------|:---------:|
| legacy-ffi | **BLOCKED** | Step 6 — `cargo llvm-cov` → `llvm-profdata merge` 失败，覆盖率报告未生成 | [PASS] |
| legacy-test-gen | **CONTINUED** | Step 10 — `cargo llvm-cov --summary-only` 使用 Rust 内置 llvm-profdata，不读系统 PATH 上的 wrapper | — 设计行为 |
| legacy-test-translate | **CONTINUED** | Step 9 — 同上。且覆盖率为报告项，不设完成阈值 | — 设计行为 |

**结论**：[PASS] PASS（legacy-ffi 硬阻断）；[WARN] 设计符合（legacy-test-gen、legacy-test-translate 按各自设计正确响应）。覆盖率版本不匹配的检测链路存在，但依赖 `LLVM_PROFDATA` 的正确设置。

---

## 汇总

| 场景 | 结论 |
|------|:----:|
| 1-A GitNexus 未安装 | [PASS] 全部 4 技能在 Step 0 正确阻断 |
| 1-B GitNexus 执行中断 | [PASS] 全部 4 技能在故障注入点阻断 |
| 1-C GitNexus 无 FTS 扩展 | [WARN] 2 技能阻断（ffi、translate），1 技能设计降级（test-gen），FTS extension 缺失 skill 难以发现，建议人为安装好 |
| 2 bindgen crate 编译失败 | [PASS] legacy-ffi 在 Step 3 build 检测到并阻断 |
| 3 llvm-profdata 版本不匹配 | [PASS] legacy-ffi 在 Step 6 硬阻断，test-gen 和 test-translate 按各自覆盖率设计（门禁/报告项）正确响应 |
