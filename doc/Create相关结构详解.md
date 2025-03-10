## `PFILE_OBJECT`创建流程

在 Windows 系统中，当用户层调用 `CreateFile` 函数时，文件系统驱动层会处理该请求，并在这个过程中创建 `PFILE_OBJECT` 对象。具体流程如下：

---

### **1. 用户层 `CreateFile` 调用**
用户层调用 `CreateFile` 函数时，会指定要打开或创建的文件路径、访问权限、共享模式等信息。`CreateFile` 函数的原型如下：
```c
HANDLE CreateFile(
  LPCTSTR lpFileName,               // 文件名或路径
  DWORD dwDesiredAccess,            // 访问权限
  DWORD dwShareMode,                // 共享模式
  LPSECURITY_ATTRIBUTES lpSecurityAttributes, // 安全属性
  DWORD dwCreationDisposition,      // 创建方式
  DWORD dwFlagsAndAttributes,       // 文件属性
  HANDLE hTemplateFile              // 模板文件句柄
);
```
当 `CreateFile` 被调用时，Windows I/O 管理器会将请求封装为一个 `IRP_MJ_CREATE` 类型的 I/O 请求包（IRP），并将其发送到文件系统驱动栈。

---

### **2. 驱动层处理 `IRP_MJ_CREATE`**
在驱动层，`IRP_MJ_CREATE` 请求会被文件系统驱动处理。每个 IRP 都包含一个 `IO_STACK_LOCATION` 结构数组，用于描述请求的详细信息。`IO_STACK_LOCATION` 结构中的 `FileObject` 字段是一个 `PFILE_OBJECT` 指针，指向与打开的文件相关的文件对象。

---

### **3. `PFILE_OBJECT` 的创建**
`PFILE_OBJECT` 对象是由 **I/O 管理器** 创建的，而不是由文件系统驱动或用户层直接创建。具体过程如下：
1. **I/O 管理器创建 `PFILE_OBJECT`**：
   - 当 `CreateFile` 调用发生时，I/O 管理器会为打开的文件创建一个 `FILE_OBJECT` 结构，并将其初始化。
   - `FILE_OBJECT` 结构表示文件的一个打开实例，包含文件路径、访问权限、文件句柄等信息。
   
2. **关联到 IRP**：
   - I/O 管理器将创建的 `PFILE_OBJECT` 对象关联到 `IRP_MJ_CREATE` 请求的 `IO_STACK_LOCATION` 结构中。
   - 文件系统驱动在处理 `IRP_MJ_CREATE` 时，可以通过 `IoGetCurrentIrpStackLocation` 函数获取当前的 `IO_STACK_LOCATION`，并访问其中的 `FileObject` 字段。

3. **文件系统驱动的处理**：
   - 文件系统驱动会根据 `FileObject` 中的信息（如文件路径、访问权限等）执行具体的文件打开或创建操作。
   - 如果操作成功，文件系统驱动会将 `FILE_OBJECT` 与目标文件关联，并返回成功状态。

---

### **4. 返回用户层**
当 `IRP_MJ_CREATE` 请求处理完成后，I/O 管理器会将 `PFILE_OBJECT` 转换为用户层可用的文件句柄（`HANDLE`），并将其返回给用户层的 `CreateFile` 调用。

---

### **总结**
`PFILE_OBJECT` 对象是由 **Windows I/O 管理器** 在用户层调用 `CreateFile` 时创建的，并关联到 `IRP_MJ_CREATE` 请求的 `IO_STACK_LOCATION` 结构中。文件系统驱动在处理 `IRP_MJ_CREATE` 时，会通过 `FileObject` 字段访问该对象，并执行具体的文件操作。



## FILE_OBJECT详解

`FILE_OBJECT` 是 Windows 内核中用于表示文件对象的结构体，由 **I/O 管理器** 在用户层调用 `CreateFile` 时创建并初始化。以下是 `FILE_OBJECT` 结构的详细说明及其赋值过程：

---

### **1. `FILE_OBJECT` 结构概述**
`FILE_OBJECT` 结构表示文件、设备、目录或卷的打开实例。它包含文件的元数据、状态信息和操作上下文等。以下是其核心字段的说明：

```c
typedef struct _FILE_OBJECT {
  CSHORT                                Type;               // 对象类型（文件对象为5）
  CSHORT                                Size;               // 对象大小
  PDEVICE_OBJECT                        DeviceObject;       // 指向关联的设备对象
  PVPB                                  Vpb;                // 指向卷参数块
  PVOID                                 FsContext;          // 文件系统上下文（文件系统驱动使用）
  PVOID                                 FsContext2;         // 额外的文件系统上下文
  PSECTION_OBJECT_POINTERS              SectionObjectPointer; // 指向文件的内存映射信息
  PVOID                                 PrivateCacheMap;    // 缓存管理器使用的私有数据
  NTSTATUS                              FinalStatus;        // I/O 请求的最终状态
  struct _FILE_OBJECT                   *RelatedFileObject; // 指向相关文件对象（如父目录）
  BOOLEAN                               LockOperation;      // 是否执行过锁定操作
  BOOLEAN                               DeletePending;      // 文件是否待删除
  BOOLEAN                               ReadAccess;         // 是否具有读权限
  BOOLEAN                               WriteAccess;        // 是否具有写权限
  BOOLEAN                               DeleteAccess;       // 是否具有删除权限
  BOOLEAN                               SharedRead;         // 是否共享读
  BOOLEAN                               SharedWrite;        // 是否共享写
  BOOLEAN                               SharedDelete;       // 是否共享删除
  ULONG                                 Flags;              // 文件对象的标志位
  UNICODE_STRING                        FileName;           // 文件名（完整路径）
  LARGE_INTEGER                         CurrentByteOffset;  // 当前文件指针位置
  __volatile ULONG                      Waiters;            // 等待该文件的线程数
  __volatile ULONG                      Busy;               // 文件对象是否正忙
  PVOID                                 LastLock;           // 最后一次锁定的信息
  KEVENT                                Lock;               // 文件对象的锁
  KEVENT                                Event;              // 文件对象的事件
  __volatile PIO_COMPLETION_CONTEXT     CompletionContext;  // I/O 完成上下文
  KSPIN_LOCK                            IrpListLock;        // IRP 列表的锁
  LIST_ENTRY                            IrpList;            // 挂起的 IRP 列表
  __volatile _IOP_FILE_OBJECT_EXTENSION *FileObjectExtension; // 文件对象的扩展信息
} FILE_OBJECT, *PFILE_OBJECT;
```

---

### **2. `FILE_OBJECT` 的创建与赋值**
`FILE_OBJECT` 由 **I/O 管理器** 在用户层调用 `CreateFile` 时创建，并初始化其字段。以下是主要字段的赋值逻辑[1](@ref)：

1. **基本字段初始化**：
   - `Type`：设置为 `5`，表示这是一个文件对象。
   - `Size`：设置为 `sizeof(FILE_OBJECT)`，即结构体的大小。
   - `DeviceObject`：指向打开文件的设备对象（通过路径解析得到）。
   - `Vpb`：指向卷参数块（如果文件位于已挂载的卷上）。
   - `FileName`：设置为文件的完整路径（通过用户层传入的路径解析得到）。

2. **权限字段初始化**：
   - `ReadAccess`、`WriteAccess`、`DeleteAccess`：根据用户层调用 `CreateFile` 时传入的访问权限（如 `GENERIC_READ`、`GENERIC_WRITE`）设置。
   - `SharedRead`、`SharedWrite`、`SharedDelete`：根据用户层调用 `CreateFile` 时传入的共享模式（如 `FILE_SHARE_READ`）设置。

3. **上下文字段初始化**：
   - `FsContext` 和 `FsContext2`：由文件系统驱动程序初始化，用于存储文件系统特定的上下文信息（如文件控制块 FCB 或流控制块 SCB）。
   - `SectionObjectPointer`：由文件系统驱动程序设置，用于支持文件的内存映射和缓存管理。

4. **其他字段初始化**：
   - `RelatedFileObject`：如果文件是相对于另一个文件打开的（如相对于某个目录），则指向该文件对象。
   - `CurrentByteOffset`：设置为文件的初始读写位置（通常为 0）。
   - `Flags`：设置为文件对象的标志位（如 `FO_FILE_OPENED` 表示文件已成功打开）。

---

### **3. `FILE_OBJECT` 的使用**
- **文件操作**：`FILE_OBJECT` 是内核中所有文件操作的核心数据结构。例如，`ReadFile` 和 `WriteFile` 等操作都会通过 `FILE_OBJECT` 定位文件并执行 I/O 操作。
- **文件系统驱动**：文件系统驱动程序通过 `FsContext` 和 `FsContext2` 字段管理文件的内部状态（如 FCB 或 SCB）。
- **缓存管理**：`SectionObjectPointer` 和 `PrivateCacheMap` 字段用于支持文件的内存映射和缓存管理。

---

### **4. 总结**
`FILE_OBJECT` 是 Windows 内核中表示文件对象的核心数据结构，由 I/O 管理器在用户层调用 `CreateFile` 时创建并初始化。其字段包含文件的元数据、状态信息和操作上下文，是文件操作和文件系统驱动的基础。