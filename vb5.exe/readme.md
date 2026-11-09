struct Project {
    /* +0x000 */ void **vftable_ProjectContext;      // 构造：ProjectContext -> Project
    /* +0x004 */ void *vbtable;                      // 构造：Project::vbtable
    /* +0x00C */ DWORD field_0C;                     // 未确认
    /* +0x028 */ HINSTANCE hmodThun;                 // +40 构造：Rby_hmodThun
    /* +0x02C */ tagNTBL *ntbl1;                     // +44 析构：NtblDestroy
    /* +0x030 */ tagNTBL *ntbl2;                     // +48 析构：NtblDestroy
    /* +0x034 */ DWORD field_34;                     // +52 构造：0
    /* +0x03C */ DWORD pStartupDesc;                 // +60 启动描述对象；-1=无
    /* +0x058 */ VBGlobalObject vbGlobal;            // +88 构造：VBGlobalObject vftable；析构：DestroyThreadInstance
    /* +0x07C */ BYTE flags_7C;                      // +124 构造：|=2
    /* +0x07E */ char projTitle[40];                 // +126 构造未写；析构清首字节；SetCompileInfo 写
    /* +0x0A8 */ CFileRep filePath;                  // +168 构造；析构
    /* +0x0B4 */ DWORD hProj;                        // +180 构造未写；析构 EbCloseProject
    /* +0x0B8 */ IUnknown *pVbaProject;              // +184 析构 Release
    /* +0x0BC */ Projitem *pStartupProjitem;         // +188 构造未写；Get/SetCompileInfo
    /* +0x0C0 */ DWORD compileInfo;                  // +192 构造未写；GetCompileInfo 读
    /* +0x0C8 */ BSTR bstrField200;                  // +200 析构 SysFreeString
    /* +0x0D4 */ CFileRep projectDir;                // +212 构造；析构
    /* +0x0E0 */ CVBServer *pServer;                 // +224 构造未写；析构 ~CVBServer + HeapFree
    /* +0x0EC */ DWORD field_EC;                     // +236 构造：0
    /* +0x0F0 */ DWORD field_F0;                     // +240 构造：1
    /* +0x0F4 */ VBProject *pVBProject;              // +244 构造：0；GetVBProject 懒加载；析构 Release
    /* +0x0F8 */ DblLstBase projitemList;            // +248 构造；析构
    /* +0x104 */ void *pProjitemList;                // +260 构造：this+248
    /* +0x108 */ DWORD preferredBase;                // +264 构造：0x11000000
    /* +0x10C */ DWORD field_10C;                    // +268 未确认
    /* +0x110 */ FILEREF *pFileRef;                  // +272 构造未写；析构 FrefDelete
    /* +0x114 */ CFileRep exeName;                   // +276 构造；析构
    /* +0x120 */ WORD flags_288;                     // +288 析构读 bit0
    /* +0x124 */ IUnknown *pUnk292;                  // +292 构造：0；析构 Release
    /* +0x128 */ CProfferService *pProfferService;   // +296 析构 Release；GetProfferService 懒加载
    /* +0x12C */ CHxmodList hxmodList;               // +300 构造；析构
    /* +0x144 */ CProjectLog *pLog;                  // +324 析构 CloseLog + HeapFree
    /* +0x148 */ IUnknown *pUnk328;                  // +328 析构 Release
    /* +0x154 */ DWORD field_154;                    // +340 构造：0
    /* +0x158 */ DWORD field_158;                    // +344 构造：0
    /* +0x15C */ DWORD verMajor;                     // +348 构造：1
    /* +0x160 */ DWORD verMinor;                     // +352
    /* +0x164 */ DWORD verRevision;                  // +356
    /* +0x168 */ DWORD verBuild;                     // +360
    /* +0x16C */ HSZ verStr[6];                      // +364..+384 构造；析构 HeapFree
    /* +0x184 */ char cmdLineArgs[256];              // +388 构造清首字节
    /* +0x284 */ void *vbtable_ProjProfiles;         // +644 构造；析构 RemoveAllSections
    /* +0x288 */ DWORD field_288;                    // +648 构造：0
    /* +0x28C */ char projPath[?];                   // +652 SetProjectName 写
    /* +0x2B8 */ DWORD flags_696;                    // +696 构造：0；析构置 1
    /* +0x2BC */ BYTE projectType;                   // +700 构造：0xFF；SetType 写
    /* +0x2C0 */ DWORD flags_2C0;                    // +704 构造清 bit1
    /* +0x2C4 */ DWORD requiredThreadModel;          // +708 构造：0xFF；键 36
    /* +0x2C8 */ DWORD field_2C8;                    // +712 构造：1；键 35
    /* +0x2CC */ DWORD options_2CC;                  // +716 构造；键 23/34
    /* +0x2D0 */ DWORD threadModel;                  // +720 构造：0；键 24
    /* +0x2D4 */ DWORD compileOptions_2D4;           // +724 构造清低 8 位；键 25~32
    /* +0x2D5 */ BYTE makeInfoFlags;                 // +725 SetMakeInfo 置 bit0
    /* +0x2D6 */ BYTE pad_2D6[2];
    /* +0x2D8 */ GUID componentGuid;                 // +728 SetMakeInfo 写
    /* +0x2E8 */ WORD makeInfoVerMajor;              // +744 SetMakeInfo 写
    /* +0x2EA */ WORD makeInfoVerMinor;              // +746 SetMakeInfo 写
    /* +0x2EC */ DWORD makeInfoExtra;                // +748 SetMakeInfo 写
    /* +0x2F0 */ HSZ makeInfoString;                 // +752 SetMakeInfo 写；析构 HeapFree
    /* +0x2F4 */ InMemTypeLib *pTypeLib;             // +756 析构 UnRegisterTypeLib + Release；SetProjectName 写 +12 BSTR
    /* +0x2F8 */ void *hFileNameLock;                // +760 SetFileNameLock 写；析构 ReleaseFileNameLock
};
