# YuShengJDK 21 —— 万物皆余胜军

[![YuShengJDK CI](https://github.com/MaZiFoundation/YuShengJDK21/actions/workflows/yushengjdk.yml/badge.svg?branch=yushengjun)](https://github.com/MaZiFoundation/YuShengJDK21/actions/workflows/yushengjdk.yml)
[![base](https://img.shields.io/badge/base-jdk--21%2B35-blue)](https://github.com/openjdk/jdk21u-dev/tree/jdk-21%2B35)
[![license](https://img.shields.io/badge/license-GPLv2%20%2B%20Classpath%20Exception-green)](LICENSE)

> 基于 **OpenJDK 21 GA（`jdk-21+35`）** 的魔改 JDK：
> Java 类层次结构的根不再是 `java.lang.Object`，而是 **`java.lang.YuShengJun`**。
>
> ```java
> public class Object extends YuShengJun { ... }
> ```
>
> **万物皆是余胜军。**

---

## 这是什么

在本 JDK 中，`java.lang.YuShengJun` 取代 `java.lang.Object` 成为类层次结构的根。
每一个 Java 对象——**包括数组**——都是 `YuShengJun` 的实例，天生携带余胜军的 DNA：

```java
package java.lang;

public class YuShengJun {
    public String yuShengJun = "余胜军";   // 每个对象与生俱来的字段
    public String say() {
        return "万物皆是余胜军";            // 每个对象都会说的话
    }
}
```

## 效果演示

```java
public class Demo {
    public static void main(String[] args) {
        Object o = new Object();
        System.out.println(o.yuShengJun);                 // 余胜军
        System.out.println(o.say());                      // 万物皆是余胜军

        int[] arr = { 1, 2, 3 };
        System.out.println(arr instanceof YuShengJun);    // true（数组也是余胜军！）
        System.out.println(arr.say());                    // 万物皆是余胜军

        System.out.println("hello".say());                // 万物皆是余胜军
        System.out.println(java.time.LocalDate.now().say()); // 万物皆是余胜军
    }
}
```

版本与厂商信息：

```console
$ java -version
YuShengJDK version "21-YuShengJDK" 2023-09-19
YuShengJDK Runtime Environment (build 21-YuShengJDK+35)
YuShengJDK 64-Bit Server VM (build 21-YuShengJDK+35, mixed mode)
```

| 属性 | 值 |
|---|---|
| 版本 | `21-YuShengJDK+35` |
| `java.runtime.name` | `YuShengJDK Runtime Environment` |
| `java.vendor` | `MaZiFoundation` |
| `javac -version` | `javac 21-YuShengJDK` |
| 基线 | OpenJDK `jdk-21+35`（JDK 21 GA，2023-09-19） |

## 获取构建产物

由 GitHub Actions 自动构建，前往
[**Actions → YuShengJDK**](https://github.com/MaZiFoundation/YuShengJDK21/actions/workflows/yushengjdk.yml)
选择最近一次成功的运行，下载产物：

| 产物 | 平台 | 说明 |
|---|---|---|
| `YuShengJDK-windows-x86_64.zip` | Windows x86_64 | 主产物，完整验证 |
| `YuShengJDK-linux-x86_64.tar.gz` | Linux x86_64 | 附加验证（非阻塞任务） |

解压后 `bin/java` 即可直接使用，无需安装。

> 注：产物**不包含 CDS 归档**（`bin/server/classes.jsa`）。在本类层次下 CDS 归档不可用
> （详见[技术文档](doc/yushengjdk.md#cds-归档)），构建时已整体禁用，功能完整、启动略慢。

## 自行构建

### Windows（主验证平台）

前置条件：

- **Cygwin**（含 `make`、`gawk`、`zip`、`unzip` 等）
- **Visual Studio 2026**（Community 即可，已适配 `--with-toolchain-version=2026`）
- **Boot JDK 21**（任意厂商，如 Zulu/Temurin 21）

```bash
# Cygwin bash 中执行
bash configure \
    --with-boot-jdk=/cygdrive/f/ZuluJDK/21 \
    --disable-warnings-as-errors \
    --with-version-pre=YuShengJDK \
    --with-version-build=35 \
    --with-version-opt=no \
    --with-vendor-name=MaZiFoundation \
    --with-toolchain-version=2026 \
    --with-tools-dir='/cygdrive/f/VisualStudio/18/Community/VC/Tools'

make images
```

产物位于 `build/windows-x86_64-server-release/images/jdk/`。

### Linux

```bash
sudo apt-get install build-essential libx11-dev libxext-dev libxrender-dev \
    libxtst-dev libxt-dev libcups2-dev libfontconfig1-dev libasound2-dev

bash configure \
    --with-boot-jdk=/path/to/boot-jdk-21 \
    --disable-warnings-as-errors \
    --with-version-pre=YuShengJDK \
    --with-version-build=35 \
    --with-version-opt=no \
    --with-vendor-name=MaZiFoundation

make images
```

构建的完整注意事项（品牌宏需 clean 重建、进程占用、中文编码等）见
[技术文档](doc/yushengjdk.md#构建注意事项)。

## 验证

CI 与本地均运行 meme 行为验证，共 **15 项断言**，覆盖：

- `instanceof YuShengJun`：普通对象、字符串、数组（含多维数组）均为 `true`
- `obj.yuShengJun` 字段值为 `"余胜军"`
- `obj.say()` 返回 `"万物皆是余胜军"`
- 数组调用 `say()` / `instanceof` 正常
- `jshell` 中复验（`jshell 21-YuShengJDK`）

最小验证程序：

```java
public class YuShengJunCheck {
    public static void main(String[] args) {
        assert "余胜军".equals(new Object().yuShengJun);
        assert "万物皆是余胜军".equals(new Object().say());
        assert new int[3] instanceof YuShengJun;
        assert "万物皆是余胜军".equals(new int[3].say());
        System.out.println("ALL PASS - 万物皆是余胜军");
    }
}
```

## 技术原理

改动横跨三层，共 23 个文件（+473/−33）：

| 层 | 改动 |
|---|---|
| **Java 类库** | 新增 `java/lang/YuShengJun.java`；`Object.java` 改为 `extends YuShengJun` |
| **javac** | `Names`/`TypeEnter`/`Attr`/`Types`/`Code`/`Gen` 全面适配新根类；`Gen` 向根类构造器注入实例字段初始化器 |
| **HotSpot** | 根类识别、虚表基数调整、根类遍历 + 数组虚表重建、数组→`YuShengJun` 验证器规则、`ciInstanceKlass::super()` C2 崩溃修复、`Class` 提前链接、接口断言放宽 |
| **构建/品牌** | `branding.conf`（YuShengJDK / MaZiFoundation）、禁用 CDS 归档、VS2026 工具链适配 |
| **CI** | `.github/workflows/yushengjdk.yml`（Windows 主任务 + Linux 附加任务） |

设计与踩坑细节（为什么 `YuShengJun` 是"第二个根类"、为什么 CDS 必须禁用、数组字段访问的边界……）
请参阅 **[doc/yushengjdk.md](doc/yushengjdk.md)**。

## 已知限制

- `((YuShengJun) 数组).yuShengJun` 读取到的是与数组头重叠的内存，返回 `null`（不会崩溃）。
  数组的 `say()` 与 `instanceof` 均正常。
- 无 CDS 归档（见上文），JVM 启动略慢于常规 JDK。
- 本项目的类层次结构与 Java 规范不符，仅用于**学习与娱乐**，请勿用于生产环境。

## 分支说明

| 分支 | 内容 |
|---|---|
| `yushengjun`（默认） | 魔改主线：基于 `jdk-21+35` + VS2026 支持补丁 + YuShengJun 全部改动 |

上游仓库：[openjdk/jdk21u-dev](https://github.com/openjdk/jdk21u-dev)（本地远程名 `upstream`）。

## 许可证

基于 OpenJDK，遵循 **GNU General Public License v2 + Classpath Exception**，
见 [LICENSE](LICENSE) 与 [ADDITIONAL_LICENSE_INFO](ADDITIONAL_LICENSE_INFO)。

---

*本项目为技术梗（meme）项目，与 OpenJDK 社区及任何个人无关。*
*上游构建文档：[doc/building.md](doc/building.md)。*
