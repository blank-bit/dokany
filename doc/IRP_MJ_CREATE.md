

### DokanDispatchCreate 代码分析文档

---

#### **1. 代码概述**
本代码是 Dokan 用户模式文件系统库的核心部分，处理 Windows 文件系统的创建请求（IRP_MJ_CREATE）。主要功能包括文件控制块（FCB/CCB）管理、共享访问检查、Oplock 处理、路径解析和异步操作处理。代码涉及内核模式下的资源管理、同步机制及 Windows 文件系统特性支持。

---

#### **2. 关键数据结构**
1. **CCB (Context Control Block)**  
   - 由 `DokanAllocateCCB` 分配，存储与文件句柄相关的上下文（如进程ID、挂载ID）。
   - 通过链表与 FCB 关联，释放时需操作链表节点（`RemoveEntryList`）。
2. **FCB (File Control Block)**  
   - 由 `DokanGetFCB` 分配或查找，表示文件或目录的元数据（如文件名、共享状态、Oplock）。
   - 包含 `ShareAccess` 字段管理共享权限，通过锁（`DokanFCBLockRW`）保证线程安全。
3. **EVENT_CONTEXT**  
   - 用户模式事件上下文，传递创建请求的详细信息（如路径、安全描述符）。

---

#### **3. 核心函数分析**

##### **3.1 `DokanDispatchCreate`**
- **功能**：处理 IRP_MJ_CREATE 请求，包括路径解析、FCB/CCB 分配、共享权限检查、Oplock 处理。

- **流程**：
  1. **参数检查**：验证 `FileObject` 和 `RelatedFileObject` 的有效性。
  
  2. **路径拼接**：结合 `RelatedFileObject` 和当前文件名生成绝对路径。
  
  3. **FCB 和 CCB 分配**：通过 `DokanGetFCB` 获取或创建文件控制块；通过`DokanAllocateCCB`获取或创建上下文控制块。
  
     > DokanAllocateCCB must be called with exlusive Fcb lock held.
  
  4. **共享访问检查**：调用 `DokanCheckShareAccess` 验证共享权限，失败时触发 Oplock 中断。
  
  5. **Oplock 处理**：  
     - 使用 `FsRtlCheckOplock` 和 `FsRtlOplockBreakH` 处理机会锁的申请与中断。
     - 支持异步中断（返回 `STATUS_PENDING`）和原子操作（`FILE_OPEN_REQUIRING_OPLOCK`）。
  
  6. **事件上下文创建**：构建 `EVENT_CONTEXT` 并注册为挂起 IRP，等待用户模式响应。
  
- **关键逻辑**：
  - **错误回滚**：失败时需释放 CCB/FCB，并回滚共享权限（`IoRemoveShareAccess`）。
  - **只读设备处理**：若设备只读且请求写入操作，返回 `STATUS_MEDIA_WRITE_PROTECTED`。

##### **3.2 `DokanCompleteCreate`**
- **功能**：完成创建请求的后处理，更新 FCB/CCB 状态，处理删除标记和通知。
- **流程**：
  1. **上下文绑定**：将用户模式返回的上下文（`EventInfo->Context`）绑定到 CCB。
  2. **状态更新**：  
     - 设置文件目录标志（`DOKAN_FILE_DIRECTORY`）。
     - 处理 `FILE_DELETE_ON_CLOSE` 标记。
  3. **通知发送**：通过 `DokanNotifyReportChange` 发送文件创建/删除通知。
  4. **资源清理**：失败时释放 CCB/FCB，并移除共享访问。

---

#### **4. 重难点分析**

##### **4.1 同步与锁机制**
- **FCB 锁**：使用读写锁（`DokanFCBLockRW`）保护 FCB 的并发访问，确保共享状态和 Oplock 的一致性。
- **原子操作回滚**：若创建失败，需通过 `DokanMaybeBackOutAtomicOplockRequest` 回滚已申请的 Oplock。

##### **4.2 Oplock 处理**
- **异步中断**：通过 `FsRtlCheckOplock` 触发异步 Oplock 中断，IRP 被挂起（`STATUS_PENDING`）后由回调函数 `DokanRetryCreateAfterOplockBreak` 重试。
- **原子请求**：`FILE_OPEN_REQUIRING_OPLOCK` 标记的请求需保证原子性，失败时需回滚。

##### **4.3 路径处理**
- **父目录解析**：`DokanGetParentDir` 从完整路径中提取父目录，处理末尾斜杠和根目录。
- **UNC 路径处理**：移除 UNC 前缀（如 `\\server\share`），确保路径格式正确。

##### **4.4 安全描述符**
- **权限继承**：通过 `SeAssignSecurity` 创建新文件的安全描述符，继承父目录权限（用户模式需处理）。

---

#### **5. 错误处理**
- **内存不足**：分配失败（如 `fileName = DokanAllocZero`）返回 `STATUS_INSUFFICIENT_RESOURCES`。
- **共享冲突**：`DokanCheckShareAccess` 返回 `STATUS_SHARING_VIOLATION`，触发 Oplock 中断或直接失败。
- **只读设备**：写入操作在只读设备上返回 `STATUS_MEDIA_WRITE_PROTECTED`。

---

#### **6. 关键代码片段**
```c
// 创建事件上下文并注册 IRP
eventContext = AllocateEventContext(RequestContext, eventLength, ccb);
status = DokanRegisterPendingIrp(RequestContext, eventContext);

// Oplock 中断处理
status = FsRtlOplockBreakH(oplock, Irp, 0, DeviceObject, DokanRetryCreateAfterOplockBreak, DokanPrePostIrp);

// 共享访问检查
status = DokanCheckShareAccess(RequestContext, fileObject, fcb, desiredAccess, shareAccess);
if (!NT_SUCCESS(status)) {
    // 处理中断或失败
}
```

---

#### **7. 总结**
- **核心逻辑**：通过 FCB/CCB 管理文件元数据，结合 Oplock 和共享检查确保并发安全。
- **难点**：异步 Oplock 中断、路径解析的边界条件、内核资源管理。
- **扩展性**：支持用户模式回调（`EVENT_CONTEXT`）实现灵活的文件操作处理。

此代码展示了 Windows 内核驱动开发中典型的 IRP 处理流程，适合作为文件系统开发的参考案例。



### DokanDispatchCreate关键流程

文件名解析，根据需要添加relatedObjectfilename

```c
NTSTATUS
DokanDispatchCreate(__in PREQUEST_CONTEXT RequestContext)
{
	// ...
    
    // RelatedFileObject：如果文件是相对于另一个文件打开的（如相对于某个目录），则指向该文件对象。
    relatedFileObject = fileObject->RelatedFileObject;
    
    // 构建fileName
    if (relatedFileName != NULL && !alternateDataStreamOfRootDir)
    {
        DOKAN_LOG_FINE_IRP(RequestContext, "RelatedFileName=\"%wZ\"", relatedFileName);

        // copy the file name of related file object
        RtlCopyMemory(fileName, relatedFileName->Buffer, relatedFileName->Length);

        if (needBackSlashAfterRelatedFile)
        {
            ((PWCHAR)fileName)[relatedFileName->Length / sizeof(WCHAR)] = '\\';
        }
        // copy the file name of fileObject
        RtlCopyMemory((PCHAR)fileName + relatedFileName->Length +
            (needBackSlashAfterRelatedFile ? sizeof(WCHAR) : 0),
            fileObject->FileName.Buffer, fileObject->FileName.Length);
        DOKAN_LOG_FINE_IRP(RequestContext, "Absolute FileName=\"%ls\"", fileName);
    }
    else
    {
        // if related file object is not specified, copy the file name of file
        // object
        RtlCopyMemory(fileName, fileObject->FileName.Buffer,
            fileObject->FileName.Length);
    }
    
    // ...
    
    // 分配FCB
    if (allocateCcb)
    {
        // SL_OPEN_TARGET_DIRECTORY 是 Windows 文件系统驱动中用于处理 IRP_MJ_CREATE 请求的一个标志位，
        // 主要用于在文件系统操作中检查目标文件所在的目录是否允许某些特定操作，例如文件创建或删除
        // Allocate an FCB or find one in the open list.
        if (RequestContext->IrpSp->Flags & SL_OPEN_TARGET_DIRECTORY)
        {
            status = DokanGetParentDir(fileName, &parentDir, &parentDirLength);
            if (status != STATUS_SUCCESS)
            {
                ExFreePool(fileName);
                fileName = NULL;
                __leave;
            }
            fcb = DokanGetFCB(RequestContext, parentDir, parentDirLength);
        }
        else
        {
            fcb = DokanGetFCB(RequestContext, fileName, fileNameLength);
        }
        if (fcb == NULL)
        {
            status = STATUS_INSUFFICIENT_RESOURCES;
            __leave;
        }
        if (fcb->BlockUserModeDispatch)
        {
            DokanLogInfo(&logger,
                L"Opened file with user mode dispatch blocked: \"%wZ\"",
                &fileObject->FileName);
        }
        DOKAN_LOG_FINE_IRP(RequestContext, "Use FCB=%p", fcb);
    }
    
    // ...
}
```



创建分配FCB

```c
PDokanFCB GetOrCreateUninitializedFcb(__in PREQUEST_CONTEXT RequestContext,
    __in PUNICODE_STRING FileName,
    __in PBOOLEAN NewElement)
{
    PDokanFCB fcb = NULL;

    fcb = ExAllocateFromLookasideListEx(&g_DokanFCBLookasideList);
    // Try again if garbage collection frees up space. This is a no-op when
    // garbage collection is disabled.
    if (fcb == NULL && DokanForceFcbGarbageCollection(RequestContext->Vcb))
    {
        fcb = ExAllocateFromLookasideListEx(&g_DokanFCBLookasideList);
    }
    if (fcb == NULL)
    {
        return NULL;
    }
    RtlZeroMemory(fcb, sizeof(DokanFCB));
    fcb->FileName = *FileName;

    PDokanFCB* fcbInTable = (PDokanFCB*)RtlInsertElementGenericTableAvl(
        &RequestContext->Vcb->FcbTable, &fcb, sizeof(PDokanFCB), NewElement);
    if (!fcbInTable)
    {
        ExFreeToLookasideListEx(&g_DokanFCBLookasideList, fcb);
        return NULL;
    }
    if (!(*NewElement))
    {
        ExFreeToLookasideListEx(&g_DokanFCBLookasideList, fcb);
    }
    return *fcbInTable;
}
```

