```
// 前提：
// - Project 至少 0x2FC = 764 字节；
// - 继承关系：DblLstElm + VBGlobalObject + virtual CProjProfiles；
// - 使用 g_hHeapPrimary 分配；VBProject 使用 g_hHeapWiz；
// - 所有字符串/句柄字段的释放方式按析构函数标注。

struct Project {
    /* ===== 基类 / 虚表 / 虚基表 ===== */
    /* +0x000 */ void **vftable_ProjectContext;   // ProjectContext -> Project 主虚表
    /* +0x004 */ void *vbtable;                   // Project::vbtable
    /* +0x008 */ ...                              // DblLstElm 基类数据（待确认）
    /* +0x028 */ HINSTANCE hmodThun;              // +40  Rby_hmodThun，线程 thunk 模块句柄
    /* +0x02C */ tagNTBL *ntbl1;                  // +44  名称表 1；析构 NtblDestroy
    /* +0x030 */ tagNTBL *ntbl2;                  // +48  名称表 2；析构 NtblDestroy
    /* +0x034 */ DWORD field_34;                  // +52  构造置 0，待确认
    /* +0x038 */ ...                              // 待确认
    /* +0x03C */ DWORD pStartupDesc;              // +60  启动描述对象；-1=无，0=空，非0=含 +0x4C Projitem*
    /* +0x040 */ ...                              // 待确认
    /* +0x058 */ VBGlobalObject vbGlobal;         // +88  VBGlobalObject 子对象；析构 DestroyThreadInstance
    /* +0x07C */ BYTE flags_7C;                   // +124 构造 |=2；待确认其他位
    /* +0x07D */ BYTE pad_7D;
    /* +0x07E */ char projTitle[40];              // +126 工程标题/描述；首字节 0 表示未设置
    /* +0x0A6 */ BYTE pad_A6[2];

    /* ===== 路径与文件 ===== */
    /* +0x0A8 */ CFileRep filePath;               // +168 工程文件全路径；构造/析构
    /* +0x0B4 */ DWORD hProj;                     // +180 工程句柄 HProj；Tip* 工具链键
    /* +0x0B8 */ IUnknown *pVbaProject;           // +184 VBA 项目对象；vtable+140 = SetTypeFlags；析构 Release
    /* +0x0BC */ Projitem *pStartupProjitem;      // +188 启动对象；Get/SetCompileInfo
    /* +0x0C0 */ DWORD compileInfo;               // +192 编译信息/启动模式
    /* +0x0C4 */ ...                              // 待确认
    /* +0x0C8 */ BSTR bstrField200;               // +200 宽字符字符串；析构 SysFreeString
    /* +0x0CC */ DWORD field_204;                 // +204 快速加载/跳过文件加载标志；LoadProjitem 非 0 时跳过 LoadFromFile
    /* +0x0D0 */ ...                              // 待确认
    /* +0x0D4 */ CFileRep projectDir;             // +212 工程目录/基路径；构造 SetDirectory(上次路径)
    /* +0x0E0 */ CVBServer *pServer;              // +224 IDE 服务器；+0x0C 字段按 flags&1 置 1/0；析构 ~CVBServer + HeapFree
    /* +0x0E4 */ ...                              // 待确认
    /* +0x0EC */ DWORD field_EC;                  // +236 构造置 0
    /* +0x0F0 */ DWORD field_F0;                  // +240 构造置 1
    /* +0x0F4 */ VBProject *pVBProject;           // +244 VBProject 自动化包装；GetVBProject 懒加载；析构 Release
    /* +0x0F8 */ DblLstBase projitemList;         // +248 工程项列表；AddFile/AddOCX/lppi* /HrKillProjitem/析构
    /* +0x104 */ void *pProjitemList;             // +260 工程项列表指针；构造 = this+248；fAddProjitem 调 vtable+4 Append
    /* +0x108 */ DWORD preferredBase;             // +264 构造 = 0x11000000，首选加载地址
    /* +0x10C */ DWORD field_10C;                 // +268 待确认
    /* +0x110 */ FILEREF *pFileRef;               // +272 工程文件引用；FReadOnly/GetProjPathname/ProcessType/析构
    /* +0x114 */ CFileRep exeName;                // +276 输出 EXE 名；GetEXEName/SetEXEName/析构
    /* +0x120 */ WORD flags_288;                  // +288 bit0 = 工程脏标志；析构 ExtIDEAfterCloseFile 读 bit0
    /* +0x122 */ BYTE pad_122[2];                 // +290 对齐/待确认
    /* +0x124 */ IUnknown *pUnk292;               // +292 COM 对象；析构 Release
    /* +0x128 */ CProfferService *pProfferService; // +296 proffer service；GetProfferService 懒加载；析构 Release
    /* +0x12C */ CHxmodList hxmodList;            // +300 hxmod 列表；构造/析构/NewDesigner AddHxmod

    /* ===== 日志 / 其他 COM ===== */
    /* +0x144 */ CProjectLog *pLog;               // +324 日志；析构 CloseLog + HeapFree
    /* +0x148 */ IUnknown *pUnk328;               // +328 COM 对象；析构 Release
    /* +0x14C */ ...                              // 待确认
    /* +0x154 */ DWORD field_154;                 // +340 构造置 0
    /* +0x158 */ DWORD field_158;                 // +344 构造置 0

    /* ===== 版本信息 ===== */
    /* +0x15C */ DWORD verMajor;                  // +348 构造默认 1
    /* +0x160 */ DWORD verMinor;                  // +352
    /* +0x164 */ DWORD verRevision;               // +356
    /* +0x168 */ DWORD verBuild;                  // +360
    /* +0x16C */ HSZ verStr[6];                   // +364..+384 VERSIONINFO 字符串；第 1 个 HszOrgName()；析构逐个 HeapFree

    /* ===== 命令行参数 ===== */
    /* +0x184 */ char cmdLineArgs[256];           // +388 命令行参数；SetCommandLineArgs 写；构造清首字节

    /* ===== 虚基类 CProjProfiles ===== */
    /* +0x284 */ void *vbtable_ProjProfiles;      // +644 CProjProfiles::vbtable；析构 RemoveAllSections
    /* +0x288 */ DWORD field_288;                 // +648 构造置 0
    /* +0x28C */ char projPath[?];                // +652 工程路径缓存；SetProjectName/OpenText 写回；≤44 字节

    /* ===== 工程状态 / 类型 / 线程模型 / 编译选项 ===== */
    /* +0x2B8 */ DWORD flags_696;                 // +696 工程关闭/失效标志；析构置 1；非 0 时 HrKillProjitem 跳过 AddIn 通知
    /* +0x2BC */ BYTE projectType;                // +700 工程类型；0xFF=未设置；3=ActiveX 类；索引 g_rgprojtypeinfo[7*type]
    /* +0x2BD */ BYTE pad_2BD[3];
    /* +0x2C0 */ DWORD flags_2C0;                 // +704 bit1 = fUseVB5ThreadModel；GetVB5ThreadModel/SetProjKeyValue(33)
    /* +0x2C4 */ DWORD requiredThreadModel;       // +708 必需线程模型；0xFF=未设置；ProcessType 初始化；键 36
    /* +0x2C8 */ DWORD field_2C8;                 // +712 计数/对象数；构造默认 1；键 35
    /* +0x2CC */ DWORD options_2CC;               // +716 bit0=键34，bit1=键23
    /* +0x2D0 */ DWORD threadModel;               // +720 线程模型枚举；ExeMake 写链接标志；键 24
    /* +0x2D4 */ DWORD compileOptions_2D4;        // +724 bit0~bit7 = 编译选项；键 25~32
    /* +0x2D5 */ BYTE makeInfoFlags;              // +725 bit0 = fMakeInfoValid；SetMakeInfo 置位
    /* +0x2D6 */ BYTE pad_2D6[2];

    /* ===== MakeInfo（组件注册信息） ===== */
    /* +0x2D8 */ GUID componentGuid;              // +728 SetMakeInfo 写
    /* +0x2E8 */ WORD makeInfoVerMajor;           // +744 SetMakeInfo 写
    /* +0x2EA */ WORD makeInfoVerMinor;           // +746 SetMakeInfo 写
    /* +0x2EC */ DWORD makeInfoExtra;             // +748 SetMakeInfo 写（可能 LCID）
    /* +0x2F0 */ HSZ makeInfoString;              // +752 SetMakeInfo 写；析构 HeapFree

    /* ===== 类型库 / 文件名锁 ===== */
    /* +0x2F4 */ InMemTypeLib *pTypeLib;          // +756 内存类型库；含 +0x0C BSTR 名称缓存；SetProjectName 写；析构 UnRegisterTypeLib + Release
    /* +0x2F8 */ void *hFileNameLock;             // +760 文件名锁；SetFileNameLock 写；析构 ReleaseFileNameLock
};
```
