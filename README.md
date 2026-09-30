# TencentUnrealEnginePaks

> Unreal Engine Pak 多层加密与自定义保护机制研究
> 适用对象：UE Pak 格式研究、互操作性分析、安全研究、逆向工程学习
> 文档状态：社区维护草案

---

## ⚠️ 免责声明

本文档**仅用于技术研究、格式分析、互操作性与安全研究**。文中**不鼓励、不提供**用于盗版、作弊、绕过版权保护或侵犯他人权益的用途。请遵守相关法律法规与软件许可协议。

---

## 目录

- [1. 概述](#1-概述)
- [2. 总体分层模型](#2-总体分层模型)
- [3. 解密流程总览](#3-解密流程总览)
- [4. 游戏实现详解](#4-游戏实现详解)
- [5. 关键算法](#5-关键算法)
- [6. Lua 保护机制专题](#6-lua-保护机制专题)
- [7. 四款游戏防护对比](#7-四款游戏防护对比)
- [8. 实现建议](#8-实现建议)
- [9. 贡献与维护](#9-贡献与维护)
- [附录：关键术语速查](#附录关键术语速查)

---

## 1. 概述

Unreal Engine 的 Pak 文件体系通常以标准 AES 加密为基础，但部分游戏会在标准方案之外叠加多层自定义保护。这些保护通常按照**由外到内**的顺序组织：

1. 先保护 Pak 尾部与 PakInfo；
2. 再保护文件索引；
3. 然后保护每个文件条目的数据；
4. 最后对特定资产（例如 Lua 字节码）做独立混淆与加密。

解密必须**自外向内逐层**进行。任何一层密钥、偏移或算法判断错误，都会导致后续层无法正确解析。

这种多层体系的设计目标通常包括：

- 提高静态分析成本；
- 防止资源被直接提取；
- 保护脚本逻辑；
- 支持动态密钥更新；
- 让不同文件、不同版本可以采用不同加密策略。

> **注意**：不同游戏版本可能修改偏移、常量、opcode 映射与密钥派生流程，本文内容需随版本更新校验。

---

## 2. 总体分层模型

### 2.1 L1：Pak 尾部 / 主密钥层

涉及结构：

- PakInfo
- Footer
- 主 AES 密钥

可能机制：

- PakInfo 自身被 XOR 或 ZUC 流加密；
- 主 AES 密钥硬编码在客户端；
- 主 AES 密钥由 Pak 尾部内嵌的 RSA 块解密派生；
- 尾部字段布局被修改，增加额外字段或跳过字节。

常见标签：`RSA`、`ZUC`、`XOR`。

### 2.2 L2：索引层

文件目录索引通常先用主密钥 AES 解密。部分游戏在 AES 解密后，还会对索引再做一次 XOR 或 ZUC 流处理。

常见机制：

- 标准 UE AES-256 解密；
- 解密后按双字、字节或固定块做二次异或；
- 索引结构可能为新版 `FName -> Entry` Map；
- 旧版可能为“条目数组 + 独立目录索引”；
- 读取时通过探测路径字符串能否读出，判断索引版本。

常见标签：`AES`、`ZUC`、`XOR`。

### 2.3 L3：条目数据层

每个文件条目可以独立选择加密算法与密钥 ID。

可能机制：

- 链式 XOR；
- SM4 加密（包括全局密钥、盐派生密钥、动态密钥）；
- 标准 AES；
- 压缩块顺序打乱；
- 使用自定义 ZSTD 字典；
- 条目头携带“加密方法号 + 密钥 ID”。

常见标签：`SM4`、`XOR`、`AES`。

### 2.4 L4：Lua 字节码层

Lua 脚本通常有额外保护：

- 自定义 opcode 重排；
- SUPERCODE 合并指令；
- 字符串常量使用独立 XOR 密钥加密；
- 头部格式字段、行号、upvalue 数、protos 数等字段宽度被修改；
- 不同游戏使用不同 Lua 版本（如 Lua 5.3 或 Lua 5.4）。

常见标签：`opcode`、`字符串 XOR`。

### 2.5 设计共性

多层保护体系通常具有以下共性：

- **密钥分层**：不同层使用不同密钥；
- **一文件一密**：每个条目可独立选择算法与密钥；
- **算法可热切换**：条目头携带加密方法号与密钥 ID；
- **动态密钥**：服务端可通过 TCP 下发新密钥，客户端无需更新；
- **版本差异大**：不同游戏、不同版本的结构与常量可能不同。

例如，PUBG Mobile 中存在通过 `0x01xxxxxx` 动态密钥下发新密钥的机制。

---

## 3. 解密流程总览

典型解密顺序如下：

1. 读取 Pak 尾部 Footer，定位 PakInfo；
2. 根据版本判断 PakInfo 是否被 XOR / ZUC / 其他流加密；
3. 解出 PakInfo，获取索引偏移、索引大小、主密钥或 RSA 块信息；
4. 使用主密钥 AES 解密索引；
5. 若索引有二次 XOR / ZUC，继续处理；
6. 遍历索引条目，读取条目头中的加密方法号与密钥 ID；
7. 根据方法号选择算法，解密条目数据；
8. 若压缩块被乱序，重建逻辑块到物理块的置换表；
9. 若使用自定义 ZSTD 字典，加载字典并解压；
10. 对 `.lua` 文件，继续做 opcode 还原、SUPERCODE 拆解与字符串 XOR 解密。

> 任何一步失败，后续步骤通常都会失败。

---

## 4. 游戏实现详解

### 4.1 PUBG（端游）与和平精英

对应类型：`GameTypes/PUBG`，`TslGame` / `Game for Peace`。

整体特点：相对克制。Pak 主体沿用标准 AES，自定义部分主要集中在 `ini` 与 Lua。

#### 4.1.1 Pak 索引与条目

- Pak 索引、文件条目使用标准 UE AES-256（`FAesKey`）；
- 无自定义块密码；
- 索引存在新旧两种格式：
  - 新版：`FName -> Entry` 的 Map；
  - 旧版：条目数组 + 独立目录索引；
- 读取时先探测路径字符串能否读出，以区分版本。

#### 4.1.2 ini 文件：固定表循环 XOR

`ini` 解密后，若开头 8 字节为魔数：

```text
0x4B4457585D5D5B7D
```

则使用一张 81 字节 XOR 表循环异或。

该表本质是字符 `'1'` 到 `'9'`（即 `0x31` 到 `0x39`），按行移位排列：

```text
31 32 33 34 35 36 37 38 39
32 33 34 35 36 37 38 39 31
33 34 35 36 37 38 39 31 32
...
```

共 9 行 × 9 列 = 81 字节。

伪代码：

```c
for (i = 0; i < len; i++) {
    data[i] ^= ini_key[i % 81];
}
```

其中 `ini_key` 由 `"123456789"` 逐行循环左移生成。

#### 4.1.3 Lua：Lua 5.3 + 字符串 XOR + 自定义 opcode

字符串常量使用 32 字节 XOR 密钥逐字节循环异或：

```text
EF C1 71 3E E3 34 7D 24 58 E1 9A 38 4F A4 6D 08
64 70 AC F2 BC E6 2E 41 4F 00 83 E7 E7 0B 20 07
```

头部格式修改包括：

- `format` 字段：1 → 0；
- 行号使用 `ushort`（2 字节压缩）；
- `LineDefined` / `LastLineDefined` 为 `ushort`；
- upvalue 数、protos 数等字段宽度被改小。

自定义 opcode 中包含 **SUPERCODE3** 合并指令，其思路是把多条常用指令合并成单条“超级指令”，例如：

- `MOVE + CALL`
- `GETTABLE + GETTABLE + GETTABLE`
- `NEWTABLE + SETTABLE + SETTABLE`

作用：

- 压缩体积；
- 干扰反编译；
- 增加静态还原难度。

> **关键结论**：和平精英与端游 PUBG 的 Lua 因引入 SUPERCODE 合并指令，无法仅靠一张 opcode 映射表还原。反编译器需要先“拆解”超级指令，这是它与 PUBG Mobile 纯 opcode 置换的本质区别。

---

### 4.2 PUBG Mobile

PUBG Mobile 的保护更复杂，包含动态密钥、压缩块乱序、自定义 ZSTD 字典与魔改 SM4 等。

#### 4.2.1 加密体系概览

可能涉及：

- 标准 AES；
- 动态密钥；
- Lite 加密；
- Salt 加密；
- 链式 XOR；
- 魔改 SM4；
- 压缩块乱序；
- 自定义 ZSTD 字典；
- Lua opcode 置换与字符串 XOR。

#### 4.2.2 动态密钥盐

公开分析中常见 36 条动态密钥盐，索引 0–35。

密钥派生：

```text
key = SHA1(对应盐)[0:16]
```

示例：

```text
索引 0 -> "edbcba1dc6b11068b44a"
索引 3 -> "5267814520c2b1904294"
```

服务端可通过 TCP 动态下发新密钥（例如 `0x01xxxxxx` 动态密钥），而无需更新客户端。

#### 4.2.3 压缩块乱序

动态、Lite、Salt 加密的条目，其压缩块在 Pak 中不按逻辑顺序存放。

需要使用 LCG 伪随机数（常被称为 `PokemonRandom`）：

```text
乘数：0x41C64E6D
增量：0x3039
```

重建“逻辑块 → 物理块”的置换表，再按表取块。

伪代码：

```c
state = blockCount;
loop:
    state = 0x41C64E6D * state + 0x3039;
    candidate = ((state >> 16) % 0x7FFF) % blockCount;
    // 去重后填入置换表
```

#### 4.2.4 ZSTD 字典压缩

当条目 `CustomData == 152` 时，数据使用带自定义字典的 ZSTD 压缩。

字典位置示例：

```text
mini_obbzsdic_obb.pak
  Content/Config/zstddic/mini_obbzsdic_obb
```

读取其 16 字节包装头后加载字典，再逐块 `TryUnwrap`。

#### 4.2.5 Lua：Lua 5.3，纯 opcode 置换 + 字符串 XOR

PUBG Mobile 的 Lua 使用 Lua 5.3，opcode 为**纯置换**，属于双射，可直接建表还原。

30 项 opcode 重排规律：

- 文件 opcode 0–16：算术 / 位运算类，映射到标准 13–29；
- 文件 opcode 17–29：MOVE / LOADK / 表操作类，映射到标准 0–12。

字符串使用 32 字节 XOR 密钥：

```text
11 21 36 47 46 57 A7 8D 9D 84 90 D8 AB 00 8C 35
26 1A F7 E4 58 05 B8 B3 15 07 D0 2C 1E 8F F6 C8
```

字符串长度规则：

- 首字节为长度；
- 若首字节为 `0xFF`，继续读 `int`；
- 实际长度 = `size - 1`。

---

### 4.3 无畏契约（Valorant Source）

对应类型：`GameTypes/Tencent/ValorantSource`。

特色：

- PakInfo 与索引使用 **ZUC-128 流加密**；
- 每个 Pak 内嵌 **RSA 块**；
- RSA 解密后拼接 SHA1 得到 AES 密钥。

#### 4.3.1 PakInfo：ZUC-128 XOR

PakInfo 缓冲区整体用一张预生成的 ZUC-128 密钥流表异或。

常见函数名语义：`DecryptValorantSourceFPakInfo`。

#### 4.3.2 每 Pak 内嵌 RSA → AES 密钥

`CustomEncryptionData` 为：

```text
offset(long) + size(int)
```

指向该 Pak 内部内嵌的一个 256 字节 RSA 块。

RSA 模数常以如下字节开头：

```text
49 2B C9 0C ...
```

与 PUBG Lite 同一套。使用 OAEP-SHA1 解密。

解密 payload 含两段，每段为 `len + data`。

AES Key 派生：

```text
key[0:20]  = SHA1(第一段)
key[20:32] = SHA1(第二段)[0:12]
```

最后将 `key` 每 4 字节按小端反转，共 8 个 `uint`。

#### 4.3.3 条目 AES + 索引 ZUC 二次异或

条目数据：

- 标准 AES；
- 密钥为上面派生的 32 字节。

索引：

- 先 AES 解密；
- 再按双字（`uint`）逐个 XOR；
- 密钥流索引为：

```text
(双字序号 + 2) & 0xF
```

即每 16 个 `uint` 循环。

#### 4.3.4 ZUC 密钥流生成：MT19937 → ZUC-128

流程：

1. 用固定流程播种 Mersenne Twister（MT19937，状态 624）；
2. 逐字节取出 16 字节 Key 与 16 字节 IV；
3. 调用 ZUC-128 生成 16 组，每组 64 字节密钥流；
4. 供 PakInfo 与索引使用。

> 公开注释指出：该表其实可直接从 Pak 尾部“对 0 值加密”暴露出来，但真实生成算法来自 MT19937 与 ZUC-128。

#### 4.3.5 索引结构

采用 Primary Index + Directory Index 两段式。

字段位于固定偏移：

- 目录索引大小：`0x3C`
- 目录索引偏移：`0x4D`
- 挂载点：`0x5C`

条目本体以编码偏移 `encoded offset` 引用 Primary Index 中的条目数据池。

#### 4.3.6 Lua：Lua 5.4，83 项 opcode 重排

使用 Lua 5.4 指令集，83 项 opcode 映射整体重排。

大致分布：

- 文件 0–12：`ADDI` / `ADDK` … `SHLI`；
- 13–32：标准二元运算；
- 33–53：`MOVE` / `LOAD` / 表操作；
- 54–82：跳转 / 比较 / 调用 / 循环等。

字符串使用与 PUBG Mobile 相同的 32 字节 XOR 密钥：

```text
11 21 36 47 ...
```

长度由 `LuaInt(varint)` 给出。

---

### 4.4 远光84（Farlight 84）

对应枚举：`GAME_Farlight84`，UE 4.25。

特点：**没有自定义块密码**。Pak 主体使用标准 AES-256，自定义点仅在 PakInfo 结构与个别资产解析。

#### 4.4.1 PakInfo 布局改动

尾部大小：

```text
SizeFarlight = Size8a + 9
```

多出 8 字节 `long` + 1 字节。

读取 `IndexOffset` 之后，需额外跳过 8 字节 `unknown long`。

#### 4.4.2 资产解析兼容

针对性兼容包括：

- `StaticMesh`：自定义渲染数据 / 顶点与 UV 项处理；
- `Landscape`：位置额外 +32 字节；
- `ScriptSet`：元素按 `SoftObjectPath` 解析；
- `Wwise` 音频 Bank、`PositionVertexBuffer` 等按 Farlight 布局读取。

#### 4.4.3 结论

远光84 的“加密”在四款中最轻：

- 标准 AES；
- PakInfo 结构微调；
- 主要工作量在资产结构差异，而非密码算法。

---

## 5. 关键算法

### 5.1 RSA-OAEP（SHA1）

统一特征：

- RSA-2048；
- 指数 `e = 65537`；
- 256 字节块；
- OAEP 哈希与 MGF1 均使用 **SHA-1**，而非 SHA-256。

公开实现中常见自实现 OAEP 解码：

1. 首字节 `0x00`；
2. 随后 20 字节 seed 与 database；
3. 用 SHA1 做 MGF 往返 XOR；
4. 校验 label 哈希 = `SHA1(空)`；
5. 定位 `0x01` 分隔符，取出明文。

### 5.2 魔改 SM4

PUBG Mobile 的 SM4 与国密 SM4 算法骨架相同：

- 32 轮；
- 非线层 τ / SBox；
- 线层 L；
- 密钥扩展。

但以下部分被替换：

- `FK` 常量；
- `CK` 常量；
- SBox；
- 密钥扩展线性变换 `L'` 使用 `ROL13` / `ROL23`。

因此不能直接套用标准 SM4 库，必须按其字节表实现。

密钥统一由：

```text
SHA1(盐串)[0:16]
```

取得。

### 5.3 ZUC-128

祖冲之序列算法，以 Key / IV 生成密钥流。

Valorant 用其加密：

- PakInfo；
- 索引。

Key / IV 由 MT19937 派生。

### 5.4 MT19937 → ZUC-128

流程：

1. 固定流程播种 MT19937；
2. 状态 624；
3. 逐字节取出 16 字节 Key 与 16 字节 IV；
4. ZUC-128 生成 16 组 64 字节密钥流。

### 5.5 LCG 块乱序

PUBG Mobile 使用 `PokemonRandom`：

```text
state = 0x41C64E6D * state + 0x3039
candidate = ((state >> 16) % 0x7FFF) % blockCount
```

去重后建立逻辑块到物理块的置换表。

### 5.6 ZSTD 自定义字典

条件：

```text
CustomData == 152
```

字典来自：

```text
mini_obbzsdic_obb.pak
  Content/Config/zstddic/mini_obbzsdic_obb
```

读取 16 字节包装头后加载字典，再逐块解压。

---

## 6. Lua 保护机制专题

### 6.1 通用结构

Lua 保护通常包括：

- 自定义 opcode 重排；
- SUPERCODE 合并指令；
- 字符串常量 XOR；
- 头部字段宽度修改；
- 行号压缩；
- upvalue / protos 字段变化。

### 6.2 字符串 XOR

**PUBG 端游 / 和平精英：**

```text
EF C1 71 3E E3 34 7D 24 58 E1 9A 38 4F A4 6D 08
64 70 AC F2 BC E6 2E 41 4F 00 83 E7 E7 0B 20 07
```

**PUBG Mobile 与 Valorant：**

```text
11 21 36 47 46 57 A7 8D 9D 84 90 D8 AB 00 8C 35
26 1A F7 E4 58 05 B8 B3 15 07 D0 2C 1E 8F F6 C8
```

### 6.3 opcode 重排

**PUBG Mobile：**

- Lua 5.3；
- 30 项 opcode 双射置换；
- 可直接建表还原。

**Valorant：**

- Lua 5.4；
- 83 项 opcode 重排；
- 覆盖算术、表操作、跳转、比较、调用、循环等。

### 6.4 SUPERCODE 合并指令

**PUBG 端游 / 和平精英：**

- 含 SUPERCODE3；
- 将多条常用指令合并为单条超级指令；
- 例如 `MOVE + CALL`、`GETTABLE + GETTABLE + GETTABLE`、`NEWTABLE + SETTABLE + SETTABLE`。

> **关键影响**：无法仅靠 opcode 映射表还原。反编译器必须先拆解超级指令。

### 6.5 各游戏 Lua 对比

| 游戏 | Lua 版本 | opcode 保护 | 字符串保护 | 关键难点 |
|---|---|---|---|---|
| PUBG 端游 / 和平精英 | Lua 5.3 | 自定义 opcode + SUPERCODE3 | 32 字节 XOR | 超级指令拆解 |
| PUBG Mobile | Lua 5.3 | 30 项双射置换 | 32 字节 XOR | 可直接建表 |
| Valorant Source | Lua 5.4 | 83 项重排 | 32 字节 XOR | 指令集整体重排 |
| Farlight 84 | 未描述 | 无特别说明 | 无特别说明 | 主要在资产结构 |

---

## 7. 四款游戏防护对比

| 项目 | PUBG 端游 / 和平精英 | PUBG Mobile | 无畏契约 Valorant Source | 远光84 Farlight 84 |
|---|---|---|---|---|
| Pak 主体 | 标准 UE AES-256 | 标准 AES + 动态 / Lite / Salt | 条目标准 AES | 标准 UE AES-256 |
| PakInfo / 尾部 | 标准为主 | 动态密钥体系 | ZUC-128 XOR | 结构微调 |
| 主密钥来源 | 标准 FAesKey | 盐派生 / 动态下发 | 每 Pak 内嵌 RSA → SHA1 拼 AES | 标准 AES |
| 索引 | 标准 AES，新旧格式 | 可能二次处理 | AES 后 ZUC 二次 XOR | 标准 AES |
| 条目数据 | 标准 AES | 链式 XOR / SM4 / AES | 标准 AES | 标准 AES |
| 压缩 | 标准 | LCG 块乱序 + ZSTD 字典 | 标准为主 | 标准为主 |
| Lua | 5.3，含 SUPERCODE | 5.3，纯 opcode 置换 | 5.4，83 项重排 | 无特别说明 |
| 最大特点 | ini XOR + Lua 超级指令 | 动态密钥 + 魔改 SM4 | ZUC + 内嵌 RSA | 资产结构差异 |

---

## 8. 实现建议

如果要实现一个兼容多游戏的 Pak 解析器，建议：

1. 将 PakInfo / Footer 解析独立成层；
2. 为每个游戏实现独立的 `GameProfile`；
3. 把“加密方法号 + 密钥 ID”作为条目解密分发依据；
4. 将 AES、SM4、ZUC、XOR、LCG、ZSTD 字典解压模块化；
5. 对 Lua 解析器单独实现 opcode 映射与 SUPERCODE 拆解；
6. 所有固定偏移、魔数、密钥常量都要版本化；
7. 为索引新旧格式、PakInfo 布局差异保留探测逻辑；
8. 对失败路径输出足够日志，方便定位是哪一层出错。

推荐模块划分：

```text
PakReader
  FooterParser
  PakInfoDecoder
  IndexDecoder
  EntryDecoder
  CompressionHandler
  LuaDecoder
Crypto
  AES
  RSA
  SM4
  ZUC
  XOR
  LCG
GameProfiles
  PUBG
  PUBGMobile
  ValorantSource
  Farlight84
```

---

## 9. 贡献与维护

欢迎以 Issue / PR 形式补充：

- 新版本偏移变化；
- 新 opcode 映射；
- 新密钥派生流程；
- 新资产结构兼容；
- 更准确的算法描述；
- 测试样本与验证方法。

请注意：

- 不要提交受版权保护的原始游戏资源；
- 不要提交可用于盗版或作弊的完整工具；
- 文档内容应保持技术研究、互操作性与安全分析导向。

建议许可：

- 文档：CC BY 4.0；
- 示例代码：MIT 或 Apache-2.0。

---

## 附录：关键术语速查

| 术语 | 含义 |
|---|---|
| PakInfo | UE Pak 尾部信息结构 |
| Footer | Pak 文件尾部，指向 PakInfo |
| FAesKey | UE 标准 AES 密钥 |
| Index | 文件目录索引 |
| Entry | 文件条目 |
| CustomData | 条目自定义数据，可用于标识特殊压缩或加密 |
| SUPERCODE | 多条 Lua 指令合并成一条超级指令 |
| ZUC-128 | 祖冲之流密码算法 |
| MT19937 | Mersenne Twister 伪随机数生成器 |
| OAEP-SHA1 | RSA OAEP 填充，哈希与 MGF1 使用 SHA-1 |
| PokemonRandom | PUBG Mobile 中用于块乱序的 LCG |
| encoded offset | 编码后的索引偏移，用于引用 Primary Index 数据池 |

---

## License

- 文档内容：CC BY 4.0
- 示例代码（如有）：MIT 或 Apache-2.0
