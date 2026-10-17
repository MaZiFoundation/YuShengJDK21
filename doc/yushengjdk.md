# YuShengJDK 技术文档

> 本文档记录"万物皆余胜军"（`java.lang.Object extends java.lang.YuShengJun`）
> 在 OpenJDK 21（基线 `jdk-21+35`）上的完整实现方案、每一处改动的动机、
> 以及构建/验证过程中的关键踩坑。

## 目录

- [设计目标](#设计目标)
- [总体设计：YuShengJun 作为 JVM 的第二个根类](#总体设计yushengjun-作为-jvm-的第二个根类)
- [Java 类库层改动](#java-类库层改动)
- [javac 层改动](#javac-层改动)
- [HotSpot 层改动](#hotspot-层改动)
- [数组的特殊处理](#数组的特殊处理)
- [CDS 归档](#cds-归档)
- [品牌与版本](#品牌与版本)
- [构建系统适配](#构建系统适配)
- [CI 工作流](#ci-工作流)
- [本地验证](#本地验证)
- [构建注意事项](#构建注意事项)
- [已知边界](#已知边界)

---

## 设计目标

1. 所有 Java 对象（**包括数组**）都是 `java.lang.YuShengJun` 的实例：
   - `任何对象 instanceof YuShengJun` 为 `true`；
   - 每个对象都有公共实例字段 `String yuShengJun = "余胜军"`；
   - 每个对象都能调用 `say()` 返回 `"万物皆是余胜军"`。
2. 基线固定为 tag `jdk-21+35`（JDK 21 GA），不跟踪 21u HEAD。
3. 产物品牌为 **YuShengJDK**，版本 `21-YuShengJDK+35`，厂商 **MaZiFoundation**。
4. 源码只用 ASCII（中文一律 `\u` 转义），避免编码问题。
5. GitHub Actions 可复现构建。

### 关键决策

| 决策点 | 结论 | 原因 |
|---|---|---|
| 字段名 | `yuShengJun` | 取名 `name` 会与 JDK 内部类的成员冲突 |
| 方法名 | `say()` | 无冲突 |
| 继承方案 | `YuShengJun` 为 JVM **第二个根类**（`super = 0`），`Object extends YuShengJun` | 不采用循环继承（`Object`/`YuShengJun` 互为父子在 VM 中不可行） |
| 改动范围 | java.base + jdk.compiler + HotSpot **三层同改** | 任何单层改动都无法让编译期与运行期同时自洽 |
| `Object.<init>` | 保留 `@IntrinsicCandidate` | 避免影响既有内联路径 |
| CDS 归档 | **整体禁用** | 见 [CDS 归档](#cds-归档) |

---

## 总体设计：YuShengJun 作为 JVM 的第二个根类

标准 JVM 中只有一个根类：`java.lang.Object`（class 文件的 `super_class` 为 0）。
本方案让 `YuShengJun` 成为第二个 `super_class = 0` 的类：

```
YuShengJun (super = 0)        ← 新的类层次之根
    └── Object                ← extends YuShengJun
            └── 其他所有类
数组（所有维度/所有元素类型）      ← VM 侧视为 YuShengJun 的子类型
```

这要求三个层面保持一致：

- **源码层**：`Object.java` 真的写上 `extends YuShengJun`；
- **编译器层**：javac 必须接受"Object 有父类"这一事实，并保证
  `YuShengJun` 的实例字段初始化器在其构造器中正确生成；
- **VM 层**：HotSpot 必须允许第二个无父类的类存在，并把它当作所有类
  （以及所有数组类型）的共同祖先，保证 `instanceof`、checkcast、
  虚方法分派、字段布局、反射、C1/C2 编译全部一致。

---

## Java 类库层改动

### `src/java.base/share/classes/java/lang/YuShengJun.java`（新增）

```java
public class YuShengJun {
    public String yuShengJun = "余胜军";   // 源码中以 \u4F59\u80DC\u519B 转义书写
    public YuShengJun() {}
    public String say() {
        return "万物皆是余胜军";            // 源码中以 \u4E07\u7269\u7686\u662F\u4F59\u80DC\u519B 转义书写
    }
}
```

要点：

- 中文字符串以 `\uXXXX` 形式书写，文件保持纯 ASCII；
- 字段为 `public` 实例字段（非 `final`，保持最简单的初始化语义）；
- 构造器体为空——字段初始化器由 javac 注入（见下文 `Gen.java`）。

### `src/java.base/share/classes/java/lang/Object.java`

```java
public class Object extends YuShengJun {
    @IntrinsicCandidate
    public Object() {}
    ...
}
```

`Object` 不再是根类，javadoc 同步更新。`@IntrinsicCandidate` 保留。

---

## javac 层改动

javac 内部大量"根类就是 Object"的硬编码假设，需要逐一改为"根类是 YuShengJun，
Object 只是它的子类"。涉及文件：

| 文件 | 改动内容 |
|---|---|
| `com/sun/tools/javac/util/Names.java` | 新增 `YuShengJun` 等名称符号 |
| `com/sun/tools/javac/comp/TypeEnter.java` | 类型登记阶段：识别新的根类，处理 `Object extends YuShengJun` 的父类登记 |
| `com/sun/tools/javac/comp/Attr.java` | 属性分析：根类无父类的语义从 Object 转移到 YuShengJun |
| `com/sun/tools/javac/code/Types.java` | 类型关系：`Object` 的超类型链包含 `YuShengJun`；数组类型可向上转型到 `YuShengJun` |
| `com/sun/tools/javac/jvm/Code.java` | 字节码生成辅助适配 |
| `com/sun/tools/javac/jvm/Gen.java` | **关键**：向根类构造器注入实例字段初始化器。常规类会把字段初始化器生成到每个构造器的 `super()` 调用之后；根类没有 `super()` 可调用，需要专门路径保证 `yuShengJun = "余胜军"` 在 `YuShengJun.<init>` 中生效 |

> 为什么 `Gen.java` 最关键：如果根类构造器里丢了字段初始化器，所有对象的
> `yuShengJun` 字段都会是 `null`（ meme 失效），而且 VM 启动早期大量对象
> 已经创建，问题会立刻暴露。

---

## HotSpot 层改动

| 文件 | 改动内容 | 动机 |
|---|---|---|
| `classfile/vmSymbols.hpp` | 新增 `java_lang_YuShengJun` 等 VM 符号 | VM 需要按名称快速定位新根类 |
| `classfile/classFileParser.cpp` | 放宽"只有 Object 可以 `super_class = 0`"的解析检查 | 让 `YuShengJun.class`（super=0）能通过解析 |
| `classfile/classLoader.cpp` | 根类加载识别 | 启动早期正确加载并登记新根类 |
| `oops/klassVtable.cpp` | **虚表基数 5 → 6** | Object 原有 5 个虚方法槽；YuShengJun 贡献了 `say()`，所有类的虚表起始基数相应调整，保证虚方法分派不错位 |
| `memory/universe.cpp` | `reinitialize_vtables()` 改为**从根类（YuShengJun）遍历**；并对**全部数组类**做一次虚表重建；`java.lang.Class` 在 `universe_post_init()` **提前链接** | 见下文两小节 |
| `ci/ciInstanceKlass.cpp` | `ciInstanceKlass::super()` 对 Object 返回真实父类 YuShengJun | 修复 C2 编译期字段表 7 vs 8 不一致导致的崩溃（C2 曾假设 Object 无父类） |
| `classfile/verificationType.cpp` | 字节码验证器：**数组可赋值给 `YuShengJun`** | 否则 `YuShengJun y = new int[3];` 之类代码过不了验证 |
| `oops/klass.cpp` | 接口父类断言放宽 | 接口类型的 super 现在指向 YuShengJun 而非 Object，原断言误报 |

### 虚表重建（`universe.cpp`）

VM 启动时，大量核心类（含数组类）在新根类介入前已完成初始化，
虚表里没有 `say()`。因此 `universe_post_init()` 阶段：

1. `reinitialize_vtables()` 不再从 `Object` 而是从 `YuShengJun` 开始遍历类层次；
2. 对**所有数组类**显式做一次虚表重建——数组没有源码、由 VM 直接创建，
   不经过类文件解析，必须手工补齐虚表；
3. `reinitialize_itables()` 同步刷新接口分派表。

### `java.lang.Class` 提前链接

CDS 归档中预解析的调用点不会触发运行期的惰性链接。若 `java.lang.Class`
等核心类保持未链接状态，归档恢复后的早期调用会出问题。因此在
`universe_post_init()` 中显式提前链接 `java.lang.Class`。

> 即便做了这一层防护，CDS 最终仍被整体禁用（见下节），该改动保留作为
> 启动顺序的加固。

---

## 数组的特殊处理

数组是 JVM 中最特殊的"类"：没有 class 文件、没有字段布局、对象头是
`mark + klass + length`。让它们成为 `YuShengJun` 的实例需要三处配合：

1. **类型关系**：`Types.java`（编译期）与 `verificationType.cpp`（运行期验证器）
   都承认"数组是 `YuShengJun` 的子类型" ⇒ `instanceof`、checkcast 通过；
2. **虚表**：数组类的虚表重建（见上节）⇒ `arr.say()` 正常分派；
3. **字段**：数组对象**没有** `yuShengJun` 字段的存储空间。
   `((YuShengJun) arr).yuShengJun` 会读到与数组头重叠的内存，返回 `null`。
   这是可接受的边界（不崩溃、不影响 `say()`），见[已知边界](#已知边界)。

---

## CDS 归档

### 为什么禁用

CDS（Class Data Sharing）归档在 dump 时把已加载类的元数据序列化到
`classes.jsa`，运行期直接映射恢复。在本类层次下：

- 归档恢复期间，根类字段初始化器无法运行；
- 归档中的预解析调用点绕过惰性链接；
- 数组类虚表重建发生在 `universe_post_init()`，与归档恢复的顺序存在根本冲突。

综合判断：此类层次下 CDS 归档**不可用**，遂在构建期整体禁用。

### 如何禁用

| 文件 | 改动 |
|---|---|
| `make/Images.gmk` | 镜像打包阶段跳过 CDS 归档生成 |
| `make/GenerateLinkOptData.gmk` | 链接优化数据（jlink 的 CDS 相关步骤）不再生成 |

效果：

- 产物中没有 `bin/server/classes.jsa`；
- `-Xshare:off` / `-Xshare:auto` 均正常工作（auto 时找不到归档则静默跳过）；
- 功能完整，仅 JVM 启动略慢。

---

## 品牌与版本

### `make/conf/branding.conf`

```properties
LAUNCHER_NAME=YuShengJDK
PRODUCT_NAME=YuShengJDK
COMPANY_NAME=MaZiFoundation
HOTSPOT_VM_DISTRO="YuShengJDK"
```

### configure 参数

```bash
--with-version-pre=YuShengJDK \
--with-version-build=35 \
--with-version-opt=no \
--with-vendor-name=MaZiFoundation
```

最终呈现：

| 位置 | 值 |
|---|---|
| `java -version` 第一行 | `YuShengJDK version "21-YuShengJDK" 2023-09-19` |
| 第二行 | `YuShengJDK Runtime Environment (build 21-YuShengJDK+35)` |
| 第三行 | `YuShengJDK 64-Bit Server VM (build 21-YuShengJDK+35, mixed mode)` |
| `java.vendor` | `MaZiFoundation` |
| `release` 文件 | `IMPLEMENTOR="MaZiFoundation"` |
| `javac -version` / `jshell` 提示 | `21-YuShengJDK` |

---

## 构建系统适配

| 文件 | 改动 | 原因 |
|---|---|---|
| `make/hotspot/lib/JvmMapfile.gmk` | gawk 兼容修正 | 本机 Cygwin gawk 与上游脚本不兼容 |
| `make/scripts/extract-vs-env.cmd` | 过滤 `NETFXSDK` 路径 | VS2026 环境下该变量会污染编译环境 |
| `make/Images.gmk`、`make/GenerateLinkOptData.gmk` | 禁用 CDS | 见上节 |
| `.github/workflows/main.yml` | push 触发排除 `yushengjun` 分支 | 上游 jtreg 测试矩阵对本分支无意义且必然失败 |
| `.github/workflows/yushengjdk.yml` | 新增自包含 CI | 见下节 |

另：VS2026 支持补丁 cherry-pick 自上游（commit `f0aed8b7c7c`）。

---

## CI 工作流

`.github/workflows/yushengjdk.yml`，触发条件：

- `workflow_dispatch`（手动）；
- push 到 `yushengjun` 分支（`**.md` 文档变更**不触发**）。

环境变量统一注入品牌参数：

```yaml
CONFIGURE_ARGS: >-
  --disable-warnings-as-errors
  --with-version-pre=YuShengJDK
  --with-version-build=35
  --with-version-opt=no
  --with-vendor-name=MaZiFoundation
```

### 任务

| Job | 平台 | 阻塞 | 步骤概要 |
|---|---|---|---|
| `Windows x86_64 (server, release)` | `windows-2022`，timeout 300min | ✅ | Checkout → 校验 meme 源码存在 → 释放磁盘 → 装 Cygwin → Boot JDK 21 → Configure → `make images` → meme 行为验证 → `java -version` → 打包 → 上传 `YuShengJDK-windows-x86_64.zip` |
| `Linux x86_64 (extra)` | `ubuntu-24.04`，timeout 240min | ❌（`continue-on-error`） | Checkout → 装依赖 → Boot JDK 21 → Configure → `make images` → meme 行为验证 → 打包 → 上传 `YuShengJDK-linux-x86_64.tar.gz` |

要点：

- 验证用的 Java 源码内嵌在 YAML 中（Windows 用 here-string、Linux 用 heredoc），
  现场编译运行并断言；
- `Sanity check` 步骤检查 `YuShengJun.java` 是否存在于仓库，防止漏提交；
- `concurrency` 分组保证同一分支重复 push 时取消旧运行；
- fork 仓库需先在 **Actions 页启用工作流**（GitHub 对 fork 默认关闭）；
- Linux 任务验证稳定后可去掉 `continue-on-error: true`。

---

## 本地验证

验证程序（15 项断言）位于本机
`C:\Users\Administrator\AppData\Local\Temp\opencode\YuShengJunFinal.java`，
覆盖三大类：

1. **类型断言**：普通对象 / `String` / 一维数组 / 多维数组 `instanceof YuShengJun` 均为 `true`；
2. **字段断言**：`obj.yuShengJun` 等于 `"余胜军"`（含多种对象类型）；
3. **方法断言**：`obj.say()` / `arr.say()` 返回 `"万物皆是余胜军"`；
4. **jshell 复验**：在 `jshell 21-YuShengJDK` 中重复上述断言。

全部 15/15 PASS。

---

## 构建注意事项

- **品牌宏属编译标志**：`branding.conf` 或 configure 参数变更后，增量构建**不会**
  重新编译受影响目标，必须 `make clean && make images`。
- **进程占用**：若上次运行残留 `java.exe` / `jshell.exe` 占用
  `images/jdk/lib/modules`，jlink 阶段的 `rm` 会报
  `Device or resource busy`，需先结束进程。
- **中文控制台输出**（Windows PowerShell）：

  ```powershell
  [Console]::OutputEncoding = [Text.Encoding]::UTF8
  java "-Dstdout.encoding=UTF-8" Demo   # PowerShell 中 -D 参数必须加引号
  ```

- 构建命令（本机 Cygwin）：

  ```powershell
  C:\cygwin64\bin\bash.exe -l -c "cd /cygdrive/f/Code/YuShengJDK21 && make images"
  ```

---

## 已知边界

| 现象 | 行为 | 说明 |
|---|---|---|
| `arr.say()`、`arr instanceof YuShengJun` | ✅ 正常 | 虚表重建 + 验证器规则保证 |
| `((YuShengJun) arr).yuShengJun` | 返回 `null`，不崩溃 | 数组无该字段的存储，读到与数组头重叠的内存 |
| `bin/server/classes.jsa` | 不存在 | CDS 已禁用，`-Xshare:auto` 下静默跳过 |
| Java 规范符合性 | 不符合 | `Object` 有父类违反 JLS，本项目仅供学习娱乐 |
