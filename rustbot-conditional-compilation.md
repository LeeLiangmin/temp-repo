# C/C++ 条件编译在 Legacy 迁移中的处理方案

> 适用范围:RustBot Skill Pack 的 `legacy-spec` → `legacy-ffi` → `legacy-test-gen`/`legacy-test-translate` → `legacy-lite-tdd` 链路,针对 C/C++ 项目存在条件编译、Rust 侧通过 C ABI shim + bindgen 迁移的场景。

## 1. 问题定义

三个层面的问题需要分开处理,顺序上后者依赖前者:

| 层面 | 问题 | 归属 |
|---|---|---|
| a. 现象层 | C/C++ 项目通过 `#ifdef`/构建开关控制编译产物 | 既有事实,无需处理,需要被识别 |
| b. 识别层 | shim 层如何确定性地识别这些差异 | `legacy-spec` 阶段产出 |
| c. 翻译层 | Rust 侧按什么规则实现对应的条件编译 | `legacy-ffi` 阶段消费 |

**核心原则:条件编译信息的"根"必须扎在 `legacy-spec` 的输出 schema 里,`legacy-ffi` 只是机械消费者,不应该临场判断或重新设计一套条件体系。**

---

## 2. 现象层:条件编译差异的三种表现形式

C/C++ 条件编译最终只会体现为以下三种差异,shim 层能可靠处理前两种,第三种需交给行为规格描述:

1. **符号存在性** —— 某函数/变量在特定配置下压根没被编译进目标文件
2. **符号签名 / ABI** —— 同名符号在不同宏下参数类型、结构体字段布局不同
3. **行为语义** —— 符号存在且签名一致,但内部实现分支不同(shim 无法从静态信息识别,只能靠规格描述兜底)

---

## 3. 识别层(`legacy-spec` 阶段):如何确定性地识别差异

### 3.1 第一步:枚举有效配置空间

不能只扫默认配置,需要从构建系统提取所有条件编译开关的可能取值:

- **CMake**:解析 `option()`、`set(... CACHE)`,或直接复用已有的 `compile_commands.json`(compilation database)与 CI 构建矩阵,不用自己猜组合
- **Makefile/Kconfig**:解析 `.config` 或已知的 feature 组合列表

输出:**本次迁移覆盖的配置组合清单**。

### 3.2 第二步:对每个配置组合做真实预处理,而非正则扫 `#ifdef`

正则扫 `#ifdef` 扫不出嵌套宏、宏间依赖关系,也判断不出最终符号表。应使用:

```bash
clang -E -DFEATURE_X -DPLATFORM_LINUX header.h   # 配置组合 A 的展开结果
clang -E -DFEATURE_Y header.h                     # 配置组合 B 的展开结果
```

或更结构化地用 libclang 的 `PreprocessingRecord`/`CursorKind` 拿到每个符号声明及其所处条件分支路径。若 GitNexus 已具备跨配置 AST/符号级 diff 能力,应直接复用,不重复造轮子。

### 3.3 输出结构:符号-配置映射表

这是 `legacy-spec` → `legacy-ffi` 之间传递的核心中间产物:

```yaml
symbol: async_io_submit
  present_in: [FEATURE_ASYNC_IO=1]
  absent_in: [FEATURE_ASYNC_IO=0]
  signature_variants:
    - condition: [FEATURE_ASYNC_IO=1, PLATFORM_LINUX=1]
      signature: "int async_io_submit(Request*, int flags)"
    - condition: [FEATURE_ASYNC_IO=1, PLATFORM_LINUX=0]
      signature: "int async_io_submit(Request*)"   # flags 参数在非 Linux 下被条件编译掉
```

---

## 4. 翻译层(`legacy-ffi` 阶段):Rust 侧实现规则

### 4.1 关键认知纠偏

**bindgen 单次调用产出的 `bindings.rs`,不需要在文件内部自己塞 `#[cfg]` 分支。** 一次 `cargo build` 本质上只对应一组确定的 feature/cfg 组合,bindgen 只要跟 `cc::Build` 用同一组 `-D` 宏解析头文件,产出的就是这次配置下唯一正确的绑定。

**真正需要显式 `#[cfg]` 的地方是上层代码引用 bindgen 产物时**,因为不同 feature 组合各自跑一次 build.rs,各自生成不同的 `bindings.rs`。

### 4.2 第一层:C++→C ABI 降级 wrapper,镜像原条件编译结构

wrapper 不能把所有函数拍平成"永远存在",必须原样保留和原 C++ 代码一致的 `#ifdef`,且必须和符号-配置映射表逐条对应,不能自建一套新宏体系:

```cpp
// wrapper.h
#ifdef FEATURE_ASYNC_IO
extern "C" int shim_async_io_submit(Request* req
    #ifdef PLATFORM_LINUX
    , int flags
    #endif
);
#endif
```

### 4.3 第二层:build.rs 中 `cc::Build` 与 `bindgen::Builder` 共享同一份宏定义

最容易出问题的地方是两者各自维护一份 `-D` 列表导致漂移。正确做法是单一来源函数:

```rust
// build.rs
fn active_defines() -> Vec<(&'static str, Option<&'static str>)> {
    let mut defs = vec![];
    if cfg!(feature = "async-io") {
        defs.push(("FEATURE_ASYNC_IO", None));
    }
    if cfg!(target_os = "linux") {
        defs.push(("PLATFORM_LINUX", None));
    }
    defs
}

fn main() {
    let defines = active_defines();

    // 1. 编译 wrapper.cpp → C ABI 静态库
    let mut build = cc::Build::new();
    build.cpp(true).file("shim/wrapper.cpp");
    for (k, v) in &defines {
        build.define(k, *v);
    }
    build.compile("shim");

    // 2. bindgen 解析 wrapper.h,必须用同一份宏
    let mut bindgen_builder = bindgen::Builder::default().header("shim/wrapper.h");
    for (k, v) in &defines {
        bindgen_builder = bindgen_builder.clang_arg(match v {
            Some(val) => format!("-D{}={}", k, val),
            None => format!("-D{}", k),
        });
    }
    let bindings = bindgen_builder.generate().expect("bindgen failed");

    let out = std::path::PathBuf::from(std::env::var("OUT_DIR").unwrap());
    bindings.write_to_file(out.join("bindings.rs")).unwrap();
}
```

`active_defines()` 应由 `legacy-ffi` 从符号-配置映射表机械生成,而非人工誊写。

### 4.4 第三层:安全封装层引用 sys 符号,`#[cfg]` 结构需与 wrapper 对齐

```rust
// sys crate lib.rs
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

```rust
// 安全封装层
#[cfg(feature = "async-io")]
pub fn submit_async(req: &mut Request) -> Result<(), Error> {
    #[cfg(target_os = "linux")]
    let ret = unsafe { shim_sys::shim_async_io_submit(req.as_raw(), 0) };
    #[cfg(not(target_os = "linux"))]
    let ret = unsafe { shim_sys::shim_async_io_submit(req.as_raw()) };
    // ...
}
```

这里的 `#[cfg]` 分支结构本质上是 wrapper.h 中 `#ifdef` 结构的直接翻版,理想情况下也应由 `legacy-ffi` 从符号-配置映射表机械生成。

### 4.5 例外场景:单一产物需同时携带多套配置

若需求是同一编译产物内同时携带多套条件编译变体(插件式运行时选择实现、跨平台 fat crate),需要对每个配置组合各跑一次 bindgen,产出多份 bindings 文件,按 cfg 选择 include:

```rust
// build.rs 对每个配置组合各生成一份 bindings_xxx.rs
// lib.rs
#[cfg(target_os = "linux")]
include!(concat!(env!("OUT_DIR"), "/bindings_linux.rs"));
#[cfg(not(target_os = "linux"))]
include!(concat!(env!("OUT_DIR"), "/bindings_other.rs"));
```

复杂度明显更高(符号命名冲突、多份 allowlist 维护),仅在明确有该需求时才采用。大多数 legacy 迁移场景中,Cargo feature 本身就是编译期单选,不需要这层复杂度。

---

## 5. 对下游阶段的连带影响

### 5.1 `legacy-test-gen` / `legacy-test-translate`

生成的测试要按条件编译变体分组标注对应的 feature 组合。CI 需要跑不同 `--features` 组合的矩阵,否则容易出现"测试全绿但只测了默认配置,其他条件分支从未被验证"的情况。

### 5.2 `legacy-lite-tdd`(sys → rs 切换)

从 `xxx_sys` 切到 `xxx_rs` 原生实现时,原来 C 宏控制的行为分支需要在 Rust 原生代码里用等价的 `cfg`/运行时分支重新表达,不能因为默认配置测试通过就顺手精简掉其他分支的处理逻辑。

---

## 6. `legacy-ffi` 的确定性产出规范(建议固化)

生成 sys crate 时,应固定产出三件互相对应、且都由同一份符号-配置映射表驱动的产物:

1. `wrapper.h`/`wrapper.cpp` 中的 `#ifdef` 结构(镜像原 C++ 条件编译)
2. `build.rs` 中 `active_defines()` 驱动的 `cc::Build` + `bindgen::Builder` 双路复用
3. **校验步骤**:对每个声明覆盖的 feature 组合各跑一次 `cargo build --no-default-features --features xxx`,用 `nm`/`objdump` 比对产出符号表与原项目对应配置下符号表是否一致

三件产物中任一处手写而非机械生成,都会成为后续漂移的起点。建议将"从符号-配置映射表生成这三件产物"做成 `legacy-ffi` 的核心确定性逻辑,不留给 Agent 临场判断。

---

## 7. 关键约束速查

| 环节 | 约束 |
|---|---|
| 宏命名 | wrapper、build.rs、Cargo feature 三处宏/开关命名保持逐字对应,便于追溯审查 |
| 平台判断 | `target_os` 用 cfg 表达,不与 Cargo feature 混用(平台是目标属性,feature 是可选功能,语义不同) |
| 符号声明 | 每个条件变体单独声明,不取"并集"编造原项目不存在的兼容签名 |
| 单一事实来源 | `cc::Build` 与 `bindgen::Builder` 的宏定义必须来自同一函数/数据结构 |
| 校验闭环 | sys crate 生成后,分配置组合编译并比对符号表,作为 `legacy-ffi` 完成的自动校验步骤,而非人工 review |
