# `GEN_PROJECT::~GEN_PROJECT` 析构函数分析

这个析构函数进一步**坐实了 `GEN_PROJECT` 是一个多重继承的 COM 类**，而且其布局信息很清楚：三个 vtable、大量子对象、全局链表、多块子管理器。

---

## 一、整体职责

析构函数做五件事：

1. **重置三个 vtable**（多重继承的 COM 接口）；
2. **通知扩展引用/清理引用列表**；
3. **清理类型/符号/调试/构建相关资源**；
4. **从全局 `GEN_PROJECT` 链表中摘除自己**；
5. **逆序析构内嵌子对象**（GUIDMGR、BLKDESC32、NAMMGR、VERMGR 等）。

---

## 二、逐段分析

### 1. 重置三个 vtable（多重继承证据）

```cpp
*this = &GEN_PROJECT::`vftable'{for `ITypeLib'};
*(this + 1) = &GEN_PROJECT::`vftable'{for `ICreateTypeLib'};
*(this + 2) = &GEN_PROJECT::`vftable'{for `IVbaProject'};
```

- `this`、`this+1`、`this+2` 是**三个 vtable 指针**，位于对象最前面（即三个基类子对象的起始）。
- 分别对应：
  - `ITypeLib`（类型库读取接口）
  - `ICreateTypeLib`（类型库创建接口）
  - `IVbaProject`（VBA 工程接口）
- 析构时重置 vtable 是 C++ 析构的典型做法，保证后续虚调用落到正确（基类）实现。
- 这也解释了之前函数清单里的 `[thunk]:GEN_PROJECT::QueryInterface`adjustor{4}/{8}``：`ITypeLib` 在 offset 0，`ICreateTypeLib` 在 offset 4，`IVbaProject` 在 offset 8，多重继承需要 adjustor thunk 调整 `this`。

**结论：`GEN_PROJECT` 同时实现 `ITypeLib`、`ICreateTypeLib`、`IVbaProject` 三个 COM 接口。**

---

### 2. 通知扩展引用

```cpp
v2 = *(this + 813);
if ( v2 )
    CVbaExtReferences::NotifyProjectDead(v2);
```

- `this+813` 是 `CVbaExtReferences*`（VBA 扩展引用管理器）。
- 析构时通知它“工程已死”，避免悬空引用。
- 对应函数清单中的 `NotifyBinderOfRefLib`、`RebuildRefList` 等引用管理逻辑。

---

### 3. 条件清理（大块）

```cpp
if ( (*(this + 3240) & 4) != 0 || *(this + 811) < 0 )
{
    ...
}
```

- `this+3240` 是标志字段，bit2（`4`）表示某种“需要清理/已初始化”状态。
- `this+811 < 0` 是另一个条件（可能是错误/失败状态）。
- 只有满足条件才做下面的清理，避免对未初始化对象做析构。

#### 3.1 引用列表

```cpp
v3 = *(this + 43);
if ( v3 )
{
    CRefList::~CRefList(*(this + 43));
    ProfMemFree(v3);
}
GEN_PROJECT::ReleaseQualCache(this);
```

- `this+43` 是 `CRefList*`（引用库列表），析构并释放。
- `ReleaseQualCache` 释放限定类型绑定缓存（对应函数清单里的 `QualCacheOf*`、`InitQualCache`、`AddQualCache` 等）。

#### 3.2 遍历释放链表

```cpp
if ( (*(this + 3240) & 4) != 0 )
{
    v4 = this + 3164;
    if ( *(this + 791) )
    {
        do
        {
            v5 = *v4;
            *v4 = *(*v4 + 4);
            (*(**v5 + 8))(*v5);   // 虚函数调用析构
            ProfMemFree(v5);
        }
        while ( *v4 );
    }
}
```

- `this+3164` 是某个链表头，`this+791` 是链表非空标志。
- 逐个取出节点 `v5`，先调用其虚函数（vtable+8，通常是 `Release`/析构），再 `ProfMemFree` 释放。
- 这是**对象链表的手工析构**，典型于 COM 引用计数对象或类型信息节点。

#### 3.3 通知 BIL/符号变更

```cpp
BilNotifySymbolChanged();
BilNotifyLargeInfrequentChange();
```

- 通知 BIL 层符号/大变更，让缓存失效（对应 EXMGR 的哈希/绑定表）。
- 对应函数清单中的 `InvalidateBindingTables`、`InvalidateNameCache` 等逻辑。

#### 3.4 释放 WATCH 对象

```cpp
if ( (*(this + 165) & 8) == 0 && dword_FBE9EF4 )
    WATCHMGR::ReleaseObjects(dword_FBE9EF4);
```

- 如果 `this+165` 的 bit3（`8`）未置位，且全局 WATCHMGR 存在，释放调试监视对象。
- 对应函数清单中的 `NotifyWatchType`、`WATCHMGR` 相关。

#### 3.5 销毁 mockup 类型信息

```cpp
v6 = *(this + 790);
if ( v6 )
{
    BASIC_TYPEINFO::Destroy(v6);
    *(this + 790) = 0;
}
```

- `this+790` 是 `BASIC_TYPEINFO*`（mockup 类型信息，对应 `GetTypeInfoMockup`）。
- 析构并清零。

#### 3.6 释放字符串/缓冲

```cpp
if ( *(this + 794) )
    ProfMemFree(*(this + 794));
if ( *(this + 795) )
    ProfMemFree(*(this + 795));
SysFreeString(*(this + 797));
```

- `this+794`、`this+795`：两个内存缓冲（可能是名称缓存/路径缓存）。
- `this+797`：BSTR 字符串（工程名/路径/文档字符串），用 `SysFreeString` 释放。

#### 3.7 销毁 buildProject

```cpp
v7 = *(this + 815);
if ( v7 )
    buildProject::Destroy(v7);
```

- `this+815` 是 `buildProject*`（构建工程对象，`MakeExe` 中 `StartEXE`/`FinishEXE` 用的那个）。
- 调用 `buildProject::Destroy` 销毁。
- 对应函数清单中 `MakeExe` 里 `*(this + 815)` 的用法。

---

### 4. 从全局链表摘除自己

```cpp
v8 = *(this + 4);
v9 = this + 12;
if ( v8 )
    *(v8 + 12) = *(this + 3);
else
    dword_FBE9EE0 = *(this + 3);
if ( *v9 )
    *(*v9 + 16) = v8;
else
    dword_FBE9EE4 = v8;
```

- `this+3`（prev）、`this+4`（next）、`this+12`（??）构成一个**双向链表节点**。
- 全局头 `dword_FBE9EE0`（first）、尾 `dword_FBE9EE4`（last）。
- 这是把当前 `GEN_PROJECT` 从**全局工程链表**中摘除的标准双向链表操作。
- 说明存在一个全局的 `GEN_PROJECT` 列表，所有工程对象挂在这条链上。

---

### 5. 逆序析构内嵌子对象

```cpp
GUIDMGR::~GUIDMGR((this + 3268));
BLKDESC32::~BLKDESC32((this + 3212));
BLK_DESC::~BLK_DESC((this + 3140));
NAMMGR::~NAMMGR((this + 532));
GENPROJ_TYPEBIND::~GENPROJ_TYPEBIND((this + 420));
VERMGR::~VERMGR((this + 400));
BLKDESC32::~BLKDESC32((this + 244));
BLK_DESC::~BLK_DESC((this + 224));
BLK_DESC::~BLK_DESC((this + 204));
```

按地址从高到低逆序析构内嵌子对象（C++ 析构顺序与构造相反）：

| 偏移 | 子对象类型 | 作用 |
|---|---|---|
| `this+3268` | `GUIDMGR` | GUID 管理器（类型/接口 GUID 分配） |
| `this+3212` | `BLKDESC32` | 32 位块描述符（某块内存/数据区） |
| `this+3140` | `BLK_DESC` | 块描述符 |
| `this+532` | `NAMMGR` | 名称管理器（工程名/模块名/名称缓存） |
| `this+420` | `GENPROJ_TYPEBIND` | 工程类型绑定管理器（QualCache/绑定表） |
| `this+400` | `VERMGR` | 版本管理器 |
| `this+244` | `BLKDESC32` | 另一块 32 位块描述 |
| `this+224` | `BLK_DESC` | 块描述 |
| `this+204` | `BLK_DESC` | 块描述 |

这些子对象对应函数清单中的：
- `NAMMGR` → `AddNameToCache`、`GetIndexOfName`、`InvalidateNameCache`、`ReadNameCacheArray`；
- `GENPROJ_TYPEBIND` → `GetDefnTypeBind`、`GetQualCache`、`BuildBindingTables`、`FixProjBindTable`；
- `VERMGR` → `SetVersion`、`UpdateVersion`；
- `GUIDMGR` → `SetGuid`、`HteOfHguid`、`AssignClassIID`；
- `BLK_DESC`/`BLKDESC32` → `QtrOfHchunk32`、`QHteRef`、`CreateHsz`、`ResetHsz` 等块/HSZ 管理。

---

## 三、对象布局推断

从析构函数可以看出 `GEN_PROJECT` 的大致布局：

```
offset 0     : vtable ITypeLib
offset 4     : vtable ICreateTypeLib
offset 8     : vtable IVbaProject
offset 12    : prev（全局链表）
offset 16    : next（全局链表）
offset 12?   : 另一个链表指针（this+12）
offset 43    : CRefList*
offset 165   : 标志（bit3）
offset 204   : BLK_DESC
offset 224   : BLK_DESC
offset 244   : BLKDESC32
offset 400   : VERMGR
offset 420   : GENPROJ_TYPEBIND
offset 532   : NAMMGR
offset 790   : BASIC_TYPEINFO*（mockup）
offset 791   : 链表非空标志
offset 794   : 内存缓冲
offset 795   : 内存缓冲
offset 797   : BSTR
offset 811   : 某状态（<0 表示失败）
offset 813   : CVbaExtReferences*
offset 815   : buildProject*
offset 3164  : 链表头
offset 3212  : BLKDESC32
offset 3240  : 标志（bit2=4）
offset 3268  : GUIDMGR
```

这与之前 `MakeExe` 里用到的 `this+51`（模块表）、`this+180`（模块数）、`this+815`（buildProject）、`this+816`（LoBil 句柄）等偏移一致，进一步确认是同一个类。

---

## 四、结论

`GEN_PROJECT::~GEN_PROJECT` 是**多重继承 COM 类 `GEN_PROJECT` 的析构函数**，负责：

1. **重置三个 COM vtable**（`ITypeLib`、`ICreateTypeLib`、`IVbaProject`），证明它是多接口 COM 对象；
2. **通知 `CVbaExtReferences` 工程已死**；
3. **条件清理**：引用列表、QualCache、对象链表、BIL 通知、WATCH 对象、mockup 类型信息、缓冲/BSTR、`buildProject`；
4. **从全局工程双向链表中摘除自己**（`dword_FBE9EE0`/`dword_FBE9EE4` 为全局头尾）；
5. **逆序析构内嵌子管理器**：`GUIDMGR`、`BLKDESC32`、`BLK_DESC`、`NAMMGR`、`GENPROJ_TYPEBIND`、`VERMGR`。

这完全不是 namespace 能有的结构。它是一个**有明确对象布局、多个 vtable、内嵌子对象、全局链表节点、COM 接口实现**的 C++ 类实例。
