# `GEN_PROJECT::GEN_PROJECT` 构造函数分析

这个构造函数是 `GEN_PROJECT` 的**默认构造函数**，进一步确认了它的类结构：**三重继承的 COM 类 + 大量内嵌子对象 + 全局双向链表节点**。构造顺序严格遵循 C++ 语义：**先设基类 vtable → 构造内嵌子对象 → 切换成派生类 vtable → 初始化自身字段 → 挂入全局链表**。

---

## 一、构造顺序（C++ 语义）

### 1. 先设置基类 vtable

```cpp
*this = &ITypeLib::`vftable';
*(this + 1) = &ICreateTypeLib::`vftable';
*(this + 2) = &IVbaProject::`vftable';
```

- `this`、`this+1`（字节 +4）、`this+2`（字节 +8）是三个基类子对象的 vtable 指针。
- 此时设置的是**基类 vtable**，表示这三个基类子对象开始构造。
- 对应 `ITypeLib`、`ICreateTypeLib`、`IVbaProject` 三个 COM 接口。

### 2. 构造内嵌子对象

```cpp
FILTITER::FILTITER((this + 176));
BLK_DESC::BLK_DESC((this + 204));
BLK_DESC::BLK_DESC((this + 224));
BLKMGR32::BLKMGR32((this + 244));
VERMGR::VERMGR((this + 400));
*(this + 105) = &DEFN_TYPEBIND::`vftable';
UNQUAL_TYPEBIND::UNQUAL_TYPEBIND((this + 448));
GENPROJ_BINDNAME_TABLE::GENPROJ_BINDNAME_TABLE((this + 476));
*(this + 105) = &GENPROJ_TYPEBIND::`vftable';
NAMMGR::NAMMGR((this + 532));
BLK_DESC::BLK_DESC((this + 3140));
BLKDESC32::BLKDESC32((this + 3212));
GUIDMGR::GUIDMGR((this + 3268));
```

按成员声明顺序构造内嵌子对象：

| 偏移 | 子对象类型 | 作用 |
|---|---|---|
| `this+176` | `FILTITER` | 类型过滤迭代器 |
| `this+204` | `BLK_DESC` | 块描述符 |
| `this+224` | `BLK_DESC` | 块描述符 |
| `this+244` | `BLKMGR32` | 32 位块管理器（`BLKDESC32` 的基类） |
| `this+400` | `VERMGR` | 版本管理器 |
| `this+420` | `GENPROJ_TYPEBIND` | 工程类型绑定管理器 |
| `this+448` | `UNQUAL_TYPEBIND` | 非限定类型绑定 |
| `this+476` | `GENPROJ_BINDNAME_TABLE` | 绑定名称表 |
| `this+532` | `NAMMGR` | 名称管理器 |
| `this+3140` | `BLK_DESC` | 块描述符 |
| `this+3212` | `BLKDESC32` | 32 位块描述符 |
| `this+3268` | `GUIDMGR` | GUID 管理器 |

**注意 `GENPROJ_TYPEBIND` 的构造：**

```cpp
*(this + 105) = &DEFN_TYPEBIND::`vftable';    // 先设基类 vtable
UNQUAL_TYPEBIND::UNQUAL_TYPEBIND((this + 448)); // 构造中间层
GENPROJ_BINDNAME_TABLE::GENPROJ_BINDNAME_TABLE((this + 476));
*(this + 105) = &GENPROJ_TYPEBIND::`vftable';  // 再切换成派生类 vtable
```

- `this+105`（字节 420）是 `GENPROJ_TYPEBIND` 的 vtable。
- 说明 `GENPROJ_TYPEBIND` 继承自 `DEFN_TYPEBIND`（或通过 `UNQUAL_TYPEBIND` 间接继承），构造时先设基类 vtable，再设派生类 vtable。
- 这和构造函数的“先基类后派生”规则一致。

### 3. 切换成 `GEN_PROJECT` 自己的 vtable

```cpp
*this = &GEN_PROJECT::`vftable'{for `ITypeLib'};
...
*(this + 1) = &GEN_PROJECT::`vftable'{for `ICreateTypeLib'};
*(this + 2) = &GEN_PROJECT::`vftable'{for `IVbaProject'};
```

- 所有基类/成员构造完成后，把三个 vtable 切换成 **`GEN_PROJECT` 自己的实现**。
- 这是 C++ 构造函数的典型做法：基类构造期间用基类 vtable，派生类构造完成后再换成派生类 vtable。
- 与析构函数开头“重置为基类 vtable”正好对称。

---

## 二、全局双向链表插入

```cpp
*(this + 3) = 0;
*(this + 4) = 0;
v2 = dword_FBE9EE4 == 0;
...
if ( v2 )
{
    dword_FBE9EE0 = this;          // 空表：作为头
}
else
{
    v3 = dword_FBE9EE4;            // 非空：挂到尾部
    *(this + 4) = dword_FBE9EE4;
    *(v3 + 12) = this;
}
dword_FBE9EE4 = this;              // 更新尾
```

- `this+3`（字节 12）= prev，`this+4`（字节 16）= next。
- `dword_FBE9EE0` = 全局链表头，`dword_FBE9EE4` = 全局链表尾。
- 这是**全局 `GEN_PROJECT` 双向链表**：所有工程对象都挂在这条链上。
- 空表时作为头；非空时追加到尾部。
- 与析构函数里“从全局链表摘除自己”完全对称。

说明：**`GEN_PROJECT` 是全局可枚举的工程对象**，IDE/编译器可通过这条链遍历所有打开的工程。

---

## 三、字段初始化

### 1. 哨兵/指针字段

```cpp
*(this + 35) = 0;
*(this + 37) = 0;
*(this + 10) = 0;
*(this + 40) = 0;
*(this + 8) = this + 36;   // 指向自身内部偏移 36*4=144
```

- `this+8` 指向 `this+36`，看起来是**自引用哨兵/内部链表头**（某个空链表指向自身）。
- 其余清零。

### 2. 大量 `-1` 初始化（无效索引/句柄）

```cpp
*(this + 110) = -1;
*(this + 109) = -1;
*(this + 108) = -1;
*(this + 107) = -1;
*(this + 106) = -1;
...
*(this + 86) = -1;
*(this + 104) = -1;
*(this + 132) = -1;
*(this + 784) = -1;
*(this + 87) = -1;
*(this + 89) = -1;
```

- 这些是无效句柄/索引（hlnam、hRefLib、模块索引、类型索引等）的初始值。
- 对应函数清单里的 `GetHRefLib`、`GetRefLibId`、`GetLocalRegId` 等。

### 3. 其他初始值

```cpp
*(this + 39) = 2046;       // 0x7FE，可能是选项/掩码默认值
*(this + 198) = 1;         // 某个计数器/版本，MakeExe 中 ++*(this+198)
*(this + 181) = 1;         // 某标志/计数
*(this + 85) = 0xFFFF;     // 无效值/哨兵
*(this + 380) = GUID_NULL; // 工程 GUID 初始为 NULL
*(this + 809) = 1;
```

- `this+198` 在 `MakeExe` 里被 `++*(this + 198)`，是编译相关计数。
- `this+380 = GUID_NULL` 与 `MakeExe` 中“无工程 GUID 时写 GUID_NULL”一致。

### 4. 标志位字段（`this+41`、`this+42`、`this+1620`）

```cpp
v4 = *(this + 41);
LOBYTE(v4) = v4 | 8;              // bit3 = 1
...
v4 &= ~0x10000000u;               // bit28 = 0
v4 &= ~0x20000000u;               // bit29 = 0
v4 &= ~0x40000000u;               // bit30 = 0
v4 &= ~0x80000000;                // bit31 = 0
BYTE1(v4) &= ~0x40u;              // bit14 = 0
v4 |= 0x400000u;                  // bit22 = 1
LOBYTE(v4) = v4 | 0x10;           // bit4 = 1
LOBYTE(v4) = v4 & 0xFB;           // bit2 = 0

v5 = *(this + 1620) & 0xFFFE;     // bit0 = 0
v5 &= ~4u;                        // bit2 = 0
v5 &= ~2u;                        // bit1 = 0
v5 &= ~0x10u;                     // bit4 = 0
*(this + 1620) = v5 & 0x1F;       // 只保留低 5 位

v6 = *(this + 42) | 1;            // bit0 = 1
v6 &= ~2u;                        // bit1 = 0
v6 &= ~4u;                        // bit2 = 0
v6 &= ~0x10u;                     // bit4 = 0
```

- `this+41` 是主标志字（32 位），`MakeExe` 中用到 `*(this+41) ^= ... & 0x100`（bit8 表示 Native）。
- `this+42` 是次级标志字。
- `this+1620` 是另一组标志（低 5 位）。
- 构造时把这些标志设成默认状态：**未 Native、未编译、未修改、非特殊模式**。

### 5. 指针/句柄字段清零

```cpp
*(this + 813) = 0;   // CVbaExtReferences*
*(this + 800) = 0;
*(this + 816) = 0;   // LoBil 句柄
*(this + 811) = 0;   // 状态（<0 表示失败）
*(this + 814) = 0;
*(this + 815) = 0;   // buildProject*
```

- `this+813`：VBA 扩展引用管理器，析构时 `NotifyProjectDead`。
- `this+815`：`buildProject*`，`MakeExe` 里 `StartEXE`/`FinishEXE` 用。
- `this+816`：`LoBil` 句柄，`MakeExe` 里 `LoBil_Initialize` 写入。
- `this+811`：状态字段，析构时判断 `< 0`。

---

## 四、与析构函数的对称关系

| 阶段 | 构造函数 | 析构函数 |
|---|---|---|
| vtable | 先基类 → 后 `GEN_PROJECT` | 重置为基类 |
| 子对象 | 正序构造（176→3268） | 逆序析构（3268→204） |
| 全局链表 | 插入尾部 | 从链表摘除 |
| 外部引用 | 初始化为 0 | `NotifyProjectDead` |
| 构建对象 | `this+815 = 0` | `buildProject::Destroy` |
| 标志 | 设默认值 | 不再处理 |

完全对称，确认 `GEN_PROJECT` 是一个**有明确生命周期、全局注册、内嵌多个管理器的 COM 类**。

---

## 五、对象布局补全

结合析构函数，`GEN_PROJECT` 的布局大致为：

```
offset 0     : vtable ITypeLib
offset 4     : vtable ICreateTypeLib
offset 8     : vtable IVbaProject
offset 12    : prev（全局链表）
offset 16    : next（全局链表）
offset 32    : 指向 this+144 的哨兵
offset 144   : 内部链表/迭代器
offset 176   : FILTITER
offset 204   : BLK_DESC
offset 224   : BLK_DESC
offset 244   : BLKMGR32
offset 400   : VERMGR
offset 420   : GENPROJ_TYPEBIND（vtable 在 this+105）
offset 448   : UNQUAL_TYPEBIND
offset 476   : GENPROJ_BINDNAME_TABLE
offset 532   : NAMMGR
offset 3140  : BLK_DESC
offset 3212  : BLKDESC32
offset 3268  : GUIDMGR
...
offset 41    : 主标志字
offset 42    : 次级标志字
offset 85    : 0xFFFF 哨兵
offset 380   : 工程 GUID
offset 790   : BASIC_TYPEINFO*（mockup）
offset 811   : 状态
offset 813   : CVbaExtReferences*
offset 815   : buildProject*
offset 816   : LoBil 句柄
offset 1620  : 低 5 位标志
offset 3240  : 标志（bit2）
```

---

## 六、结论

`GEN_PROJECT::GEN_PROJECT` 是**三重继承 COM 类 `GEN_PROJECT` 的默认构造函数**，负责：

1. **设置三个基类 vtable**（`ITypeLib`、`ICreateTypeLib`、`IVbaProject`）；
2. **按成员顺序构造内嵌子对象**：`FILTITER`、`BLK_DESC`、`BLKMGR32`、`VERMGR`、`GENPROJ_TYPEBIND`（含 `DEFN_TYPEBIND`/`UNQUAL_TYPEBIND` 基类构造）、`GENPROJ_BINDNAME_TABLE`、`NAMMGR`、`BLKDESC32`、`GUIDMGR`；
3. **切换成 `GEN_PROJECT` 自己的 vtable**；
4. **初始化大量字段**：哨兵指针、`-1` 无效句柄、`GUID_NULL`、标志位默认值、`buildProject`/`LoBil`/扩展引用指针清零；
5. **插入全局 `GEN_PROJECT` 双向链表**（`dword_FBE9EE0` 头、`dword_FBE9EE4` 尾）。

它与析构函数完全对称，进一步证明 `GEN_PROJECT` 是一个**有明确对象布局、多 vtable、内嵌管理器、全局注册的 C++ COM 类**，而不是 namespace。
