## `IoRegisterDeviceInterface` 和 `IoCreateSymbolicLink` 
都是为了让用户模式应用程序能够访问内核模式设备，但它们的作用和使用场景有所不同。以下是两者的详细比较：

### 1. `IoCreateSymbolicLink`

- **目的**：创建一个符号链接，使得用户模式应用程序可以通过该符号链接名称打开设备。
- **使用场景**：适用于需要直接创建一个已知名称的符号链接的情况。
- **灵活性**：符号链接名称是固定的，由驱动程序指定。
- **适用范围**：适用于不需要动态发现设备的情况，或者设备名称不需要遵循特定的标准。

#### 示例代码

```c
#include <ntddk.h>

NTSTATUS MyDriverAddDevice(PDRIVER_OBJECT DriverObject, PDEVICE_OBJECT PhysicalDeviceObject) {
    NTSTATUS status;
    PDEVICE_OBJECT fdo;
    PDEVICE_EXTENSION deviceExtension;
    UNICODE_STRING deviceNameUnicodeString;
    UNICODE_STRING symLinkUnicodeString;

    // 初始化设备名称和符号链接名称
    RtlInitUnicodeString(&deviceNameUnicodeString, L"\\Device\\MyDevice");
    RtlInitUnicodeString(&symLinkUnicodeString, L"\\??\\MyDevice");

    // 创建 FDO
    status = IoCreateDevice(
        DriverObject,                   // 驱动对象
        sizeof(DEVICE_EXTENSION),       // 设备扩展大小
        &deviceNameUnicodeString,       // 设备名称
        FILE_DEVICE_UNKNOWN,            // 设备类型
        0,                            // 设备特性
        FALSE,                        // 是否独占
        &fdo                         // 新设备对象
    );

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create device object: 0x%X\n", status));
        return status;
    }

    // 获取设备扩展
    deviceExtension = (PDEVICE_EXTENSION)fdo->DeviceExtension;
    deviceExtension->PhysicalDeviceObject = PhysicalDeviceObject;

    // 创建符号链接
    status = IoCreateSymbolicLink(&symLinkUnicodeString, &deviceNameUnicodeString);
    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create symbolic link: 0x%X\n", status));
        IoDeleteDevice(fdo);
        return status;
    }

    // 设置设备对象标志
    fdo->Flags |= DO_BUFFERED_IO;
    fdo->Flags &= ~DO_DEVICE_INITIALIZING;

    return STATUS_SUCCESS;
}
```

### 2. `IoRegisterDeviceInterface`

- **目的**：注册一个设备接口，允许用户模式应用程序通过设备接口类 GUID 发现设备，并动态生成符号链接名称。
- **使用场景**：适用于需要动态发现设备的情况，或者设备名称需要遵循特定的标准。
- **灵活性**：符号链接名称由系统生成，通常是唯一的，确保不会与其他设备冲突。
- **适用范围**：适用于需要支持多个相同类型设备的情况，或者设备名称需要动态分配。

#### 示例代码

```c
#include <ntddk.h>

// 定义设备接口类 GUID
DEFINE_GUID(GUID_DEVINTERFACE_WANG_DEVICE, 0x12345678, 0x1234, 0x1234, 0x12, 0x34, 0x12, 0x34, 0x56, 0x78, 0x9A, 0xBC);

NTSTATUS MyDriverAddDevice(PDRIVER_OBJECT DriverObject, PDEVICE_OBJECT PhysicalDeviceObject) {
    NTSTATUS status;
    PDEVICE_OBJECT fdo;
    PDEVICE_EXTENSION deviceExtension;
    UNICODE_STRING deviceNameUnicodeString;
    UNICODE_STRING symLinkUnicodeString;
    UNICODE_STRING interfaceName;

    // 初始化设备名称和符号链接名称
    RtlInitUnicodeString(&deviceNameUnicodeString, L"\\Device\\MyDevice");
    RtlInitUnicodeString(&symLinkUnicodeString, L"\\??\\MyDevice");

    // 创建 FDO
    status = IoCreateDevice(
        DriverObject,                   // 驱动对象
        sizeof(DEVICE_EXTENSION),       // 设备扩展大小
        &deviceNameUnicodeString,       // 设备名称
        FILE_DEVICE_UNKNOWN,            // 设备类型
        0,                            // 设备特性
        FALSE,                        // 是否独占
        &fdo                         // 新设备对象
    );

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create device object: 0x%X\n", status));
        return status;
    }

    // 获取设备扩展
    deviceExtension = (PDEVICE_EXTENSION)fdo->DeviceExtension;
    deviceExtension->PhysicalDeviceObject = PhysicalDeviceObject;

    // 初始化符号链接名称
    RtlZeroMemory(&deviceExtension->SymbolicLinkName, sizeof(UNICODE_STRING));

    // 创建符号链接
    status = IoCreateSymbolicLink(&symLinkUnicodeString, &deviceNameUnicodeString);
    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create symbolic link: 0x%X\n", status));
        IoDeleteDevice(fdo);
        return status;
    }

    // 复制符号链接名称到设备扩展
    RtlCopyUnicodeString(&deviceExtension->SymbolicLinkName, &symLinkUnicodeString);

    // 注册设备接口
    status = IoRegisterDeviceInterface(
        PhysicalDeviceObject,           // 物理设备对象
        &GUID_DEVINTERFACE_WANG_DEVICE, // 设备接口类 GUID
        NULL,                           // 引用字符串（可选）
        &interfaceName                  // 符号链接名称
    );

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to register device interface: 0x%X\n", status));
        IoDeleteSymbolicLink(&symLinkUnicodeString);
        IoDeleteDevice(fdo);
        return status;
    }

    // 启用设备接口
    status = IoSetDeviceInterfaceState(&interfaceName, TRUE);
    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to set device interface state: 0x%X\n", status));
        IoDeleteSymbolicLink(&symLinkUnicodeString);
        IoDeleteDevice(fdo);
        return status;
    }

    // 设置设备对象标志
    fdo->Flags |= DO_BUFFERED_IO;
    fdo->Flags &= ~DO_DEVICE_INITIALIZING;

    return STATUS_SUCCESS;
}
```

### 3. 主要区别

- **符号链接名称**：
  - **`IoCreateSymbolicLink`**：符号链接名称由驱动程序指定，固定不变。
  - **`IoRegisterDeviceInterface`**：符号链接名称由系统生成，通常是唯一的，确保不会与其他设备冲突。

- **设备发现**：
  - **`IoCreateSymbolicLink`**：用户模式应用程序需要知道符号链接名称才能打开设备。
  - **`IoRegisterDeviceInterface`**：用户模式应用程序可以通过设备接口类 GUID 动态发现设备，并获取符号链接名称。

- **适用场景**：
  - **`IoCreateSymbolicLink`**：适用于设备名称不需要动态分配的情况。
  - **`IoRegisterDeviceInterface`**：适用于需要动态发现设备的情况，或者设备名称需要遵循特定的标准。

### 4. 总结

- **`IoCreateSymbolicLink`**：适用于需要直接创建一个已知名称的符号链接的情况，符号链接名称固定。
- **`IoRegisterDeviceInterface`**：适用于需要动态发现设备的情况，符号链接名称由系统生成，确保唯一性。


## **PDO和FDO**
在 Windows 驱动程序模型中，功能设备对象（FDO，Functional Device Object）和物理设备对象（PDO，Physical Device Object）是两种不同类型的设备对象，它们在设备栈中扮演不同的角色。以下是 FDO 和 PDO 的主要区别：

### 1. 功能设备对象（FDO）

- **定义**：FDO 是由驱动程序创建的，用于表示设备的功能。它负责处理设备的 I/O 请求和控制操作。
- **创建**：FDO 通常在 `AddDevice` 回调函数中创建，使用 `IoCreateDevice` 或 `IoCreateDeviceSecure` 函数。
- **角色**：FDO 是设备栈中的顶层对象，负责处理来自用户模式应用程序或系统的服务请求。它实现了设备的具体功能，如读写操作、控制命令等。
- **示例**：一个 USB 存储设备的 FDO 可能负责处理文件系统的读写请求。

### 2. 物理设备对象（PDO）

- **定义**：PDO 是由总线驱动程序创建的，用于表示物理设备的存在。它主要用于报告设备的状态和属性。
- **创建**：PDO 通常由总线驱动程序（如 PCI 总线驱动程序）在检测到新设备时自动创建。
- **角色**：PDO 是设备栈中的底层对象，负责向操作系统报告设备的存在、状态和属性。它不直接处理 I/O 请求，而是将请求转发给上层的 FDO。
- **示例**：一个 USB 设备的 PDO 可能只是报告设备的存在和基本属性，而不处理具体的读写操作。

### 3. 主要区别

- **创建者**：
  - **FDO**：由功能驱动程序（如类驱动程序）创建。
  - **PDO**：由总线驱动程序创建。

- **职责**：
  - **FDO**：处理设备的 I/O 请求和控制操作。
  - **PDO**：报告设备的存在和属性。

- **位置**：
  - **FDO**：位于设备栈的顶层。
  - **PDO**：位于设备栈的底层。

- **交互**：
  - **FDO**：与用户模式应用程序或系统交互，处理 I/O 请求。
  - **PDO**：与总线驱动程序和操作系统交互，报告设备状态。

### 4. 示例代码

以下是一个简单的示例，展示了如何在 `AddDevice` 回调函数中创建 FDO：

```c
#include <ntddk.h>

#define DEVICE_NAME L"\\Device\\MyDevice"
#define SYM_LINK_NAME L"\\??\\MyDevice"

NTSTATUS MyDriverAddDevice(PDRIVER_OBJECT DriverObject, PDEVICE_OBJECT PhysicalDeviceObject) {
    NTSTATUS status;
    PDEVICE_OBJECT fdo;
    PDEVICE_EXTENSION deviceExtension;
    UNICODE_STRING deviceNameUnicodeString;
    UNICODE_STRING symLinkUnicodeString;

    // 初始化设备名称和符号链接名称
    RtlInitUnicodeString(&deviceNameUnicodeString, DEVICE_NAME);
    RtlInitUnicodeString(&symLinkUnicodeString, SYM_LINK_NAME);

    // 创建 FDO
    status = IoCreateDevice(
        DriverObject,                   // 驱动对象
        sizeof(DEVICE_EXTENSION),       // 设备扩展大小
        &deviceNameUnicodeString,       // 设备名称
        FILE_DEVICE_UNKNOWN,            // 设备类型
        0,                            // 设备特性
        FALSE,                        // 是否独占
        &fdo                         // 新设备对象
    );

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create device object: 0x%X\n", status));
        return status;
    }

    // 获取设备扩展
    deviceExtension = (PDEVICE_EXTENSION)fdo->DeviceExtension;
    deviceExtension->PhysicalDeviceObject = PhysicalDeviceObject;

    // 设置设备对象标志
    fdo->Flags |= DO_BUFFERED_IO;
    fdo->Flags &= ~DO_DEVICE_INITIALIZING;

    // 创建符号链接
    status = IoCreateSymbolicLink(&symLinkUnicodeString, &deviceNameUnicodeString);
    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create symbolic link: 0x%X\n", status));
        IoDeleteDevice(fdo);
        return status;
    }

    return STATUS_SUCCESS;
}
```

### 5. 总结

- **FDO**：功能设备对象，由功能驱动程序创建，处理设备的 I/O 请求和控制操作。
- **PDO**：物理设备对象，由总线驱动程序创建，报告设备的存在和属性。

## **WDM驱动程序基本结构**

好的，让我更清晰地解释一下在创建驱动程序时，驱动程序如何创建设备对象的过程。创建驱动程序并不自动创建设备对象，而是通过驱动程序中的特定函数来实现这一点。以下是详细的步骤和解释：

### 1. 驱动程序的基本结构

一个典型的 WDM 驱动程序包含以下几个部分：
- **DriverEntry**: 驱动程序的入口点。
- **AddDevice**: 创建设备对象的回调函数。
- **IRP 处理函数**: 处理各种 I/O 请求，如创建、关闭、读取、写入等。
- **Unload**: 驱动程序卸载时的清理工作。

### 2. DriverEntry 函数

`DriverEntry` 是驱动程序的入口点，在驱动程序加载时被调用。在这个函数中，你需要注册 `AddDevice` 函数以及其他主要的 IRP 处理函数。

#### 示例代码

```c
#include <ntddk.h>

#define DEVICE_NAME L"\\Device\\VirtualDevice"
#define SYM_LINK_NAME L"\\??\\VirtualDevice"

NTSTATUS VirtualDeviceAddDevice(PDRIVER_OBJECT DriverObject, PDEVICE_OBJECT PhysicalDeviceObject);
NTSTATUS VirtualDeviceCreate(PDEVICE_OBJECT DeviceObject, PIRP Irp);
NTSTATUS VirtualDeviceClose(PDEVICE_OBJECT DeviceObject, PIRP Irp);
NTSTATUS VirtualDeviceDispatch(PDEVICE_OBJECT DeviceObject, PIRP Irp);
VOID VirtualDeviceUnload(PDRIVER_OBJECT DriverObject);

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);

    // 注册 AddDevice 函数
    DriverObject->DriverExtension->AddDevice = VirtualDeviceAddDevice;

    // 注册其他主要函数
    DriverObject->MajorFunction[IRP_MJ_CREATE] = VirtualDeviceCreate;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = VirtualDeviceClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = VirtualDeviceDispatch;
    DriverObject->DriverUnload = VirtualDeviceUnload;

    return STATUS_SUCCESS;
}
```

### 3. AddDevice 函数

`AddDevice` 函数是当系统检测到一个新的硬件设备实例时被调用的回调函数。在这个函数中，你可以创建设备对象并设置其属性。

#### 示例代码

```c
NTSTATUS VirtualDeviceAddDevice(PDRIVER_OBJECT DriverObject, PDEVICE_OBJECT PhysicalDeviceObject) {
    NTSTATUS status;
    PDEVICE_OBJECT deviceObject;
    PDEVICE_EXTENSION deviceExtension;
    UNICODE_STRING deviceNameUnicodeString;
    UNICODE_STRING symLinkUnicodeString;

    // 初始化设备名称和符号链接名称
    RtlInitUnicodeString(&deviceNameUnicodeString, DEVICE_NAME);
    RtlInitUnicodeString(&symLinkUnicodeString, SYM_LINK_NAME);

    // 创建设备对象
    status = IoCreateDevice(
        DriverObject,                   // 驱动对象
        sizeof(DEVICE_EXTENSION),       // 设备扩展大小
        &deviceNameUnicodeString,       // 设备名称
        FILE_DEVICE_UNKNOWN,            // 设备类型
        0,                            // 设备特性
        FALSE,                        // 是否独占
        &deviceObject                 // 新设备对象
    );

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create device object: 0x%X\n", status));
        return status;
    }

    // 获取设备扩展
    deviceExtension = (PDEVICE_EXTENSION)deviceObject->DeviceExtension;
    deviceExtension->PhysicalDeviceObject = PhysicalDeviceObject;

    // 设置设备对象标志
    deviceObject->Flags |= DO_BUFFERED_IO;
    deviceObject->Flags &= ~DO_DEVICE_INITIALIZING;

    // 创建符号链接
    status = IoCreateSymbolicLink(&symLinkUnicodeString, &deviceNameUnicodeString);
    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create symbolic link: 0x%X\n", status));
        IoDeleteDevice(deviceObject);
        return status;
    }

    return STATUS_SUCCESS;
}
```

### 4. IRP 处理函数

IRP 处理函数用于处理来自用户模式或其他驱动程序的 I/O 请求。常见的 IRP 处理函数包括 `Create`、`Close` 和 `Dispatch`。

#### 示例代码

```c
NTSTATUS VirtualDeviceCreate(PDEVICE_OBJECT DeviceObject, PIRP Irp) {
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

NTSTATUS VirtualDeviceClose(PDEVICE_OBJECT DeviceObject, PIRP Irp) {
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

NTSTATUS VirtualDeviceDispatch(PDEVICE_OBJECT DeviceObject, PIRP Irp) {
    PIO_STACK_LOCATION irpStack;
    NTSTATUS status = STATUS_SUCCESS;
    ULONG ioControlCode;
    PVOID inputBuffer;
    PVOID outputBuffer;
    ULONG inputBufferLength;
    ULONG outputBufferLength;

    irpStack = IoGetCurrentIrpStackLocation(Irp);
    ioControlCode = irpStack->Parameters.DeviceIoControl.IoControlCode;
    inputBuffer = Irp->AssociatedIrp.SystemBuffer;
    outputBuffer = Irp->AssociatedIrp.SystemBuffer;
    inputBufferLength = irpStack->Parameters.DeviceIoControl.InputBufferLength;
    outputBufferLength = irpStack->Parameters.DeviceIoControl.OutputBufferLength;

    switch (ioControlCode) {
        case IOCTL_VIRTUAL_DEVICE_REQUEST:
            // 处理 IOCTL 请求
            status = VirtualDeviceHandleRequest(inputBuffer, inputBufferLength, outputBuffer, outputBufferLength);
            break;
        default:
            status = STATUS_INVALID_DEVICE_REQUEST;
            break;
    }

    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

NTSTATUS VirtualDeviceHandleRequest(PVOID inputBuffer, ULONG inputBufferLength, PVOID outputBuffer, ULONG outputBufferLength) {
    // 处理请求的具体逻辑
    // 例如，复制数据到输出缓冲区
    RtlCopyMemory(outputBuffer, inputBuffer, min(inputBufferLength, outputBufferLength));
    return STATUS_SUCCESS;
}
```

### 5. Unload 函数

`Unload` 函数用于在驱动程序卸载时清理资源。

#### 示例代码

```c
VOID VirtualDeviceUnload(PDRIVER_OBJECT DriverObject) {
    UNICODE_STRING symLinkName = RTL_CONSTANT_STRING(SYM_LINK_NAME);
    PDEVICE_OBJECT deviceObject = DriverObject->DeviceObject;

    IoDeleteSymbolicLink(&symLinkName);
    IoDeleteDevice(deviceObject);
}
```

### 6. 定义 IOCTL 代码

定义 IOCTL 代码以便用户模式程序可以与驱动程序通信。

#### 示例代码

```c
#define FILE_DEVICE_VIRTUAL_DEVICE 0x8000

#define IOCTL_VIRTUAL_DEVICE_REQUEST CTL_CODE(FILE_DEVICE_VIRTUAL_DEVICE, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)
```

### 7. 创建 `.inf` 文件

创建一个 `.inf` 文件来安装驱动程序。

#### 示例 `.inf` 文件

```inf
[Version]
Signature="$Windows NT$"
Class=System
ClassGuid={4D36E97D-E325-11CE-BFC1-08002BE10318}
Provider=%ProviderName%
DriverVer=01/01/2023,1.0.0.0

[Manufacturer]
%ManufacturerName%=DeviceList

[DeviceList]
%DeviceDesc%=Device_Install, Root\VirtualDevice

[Device_Install.NT]
CopyFiles=Device_CopyFiles

[Device_Install.NT.Services]
AddService=%ServiceName%,%SPSVCINST_ASSOCSERVICE%,Device_Service_Inst

[Device_CopyFiles]
VirtualDevice.sys

[Device_Service_Inst]
DisplayName    = %ServiceDesc%
ServiceType    = %SERVICE_KERNEL_DRIVER%
StartType      = %SERVICE_DEMAND_START%
ErrorControl   = %SERVICE_ERROR_NORMAL%
ServiceBinary  = %12%\VirtualDevice.sys
LoadOrderGroup = Base

[Strings]
ProviderName = "My Company"
ManufacturerName = "My Company"
DeviceDesc = "Virtual Device"
ServiceName = "VirtualDevice"
ServiceDesc = "Virtual Device Driver"
SPSVCINST_ASSOCSERVICE = 0x02
SERVICE_KERNEL_DRIVER = 0x1
SERVICE_DEMAND_START = 0x3
SERVICE_ERROR_NORMAL = 0x1
```

### 8. 编写用户模式程序

编写一个用户模式程序来与驱动程序通信。

#### 示例代码

```c
#include <windows.h>
#include <stdio.h>

#define IOCTL_VIRTUAL_DEVICE_REQUEST CTL_CODE(0x8000, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)

int main() {
    HANDLE hDevice;
    DWORD bytesReturned;
    char inputBuffer[] = "Hello from user mode!";
    char outputBuffer[256];

    hDevice = CreateFile(
        TEXT("\\\\.\\VirtualDevice"),    // 设备名称
        GENERIC_READ | GENERIC_WRITE, // 访问权限
        0,                            // 共享模式
        NULL,                         // 安全属性
        OPEN_EXISTING,                // 打开方式
        0,                            // 文件属性
        NULL                          // 模板文件句柄
    );

    if (hDevice == INVALID_HANDLE_VALUE) {
        printf("Failed to open device: %d\n", GetLastError());
        return 1;
    }

    if (!DeviceIoControl(
        hDevice,                      // 设备句柄
        IOCTL_VIRTUAL_DEVICE_REQUEST, // IOCTL 代码
        inputBuffer,                  // 输入缓冲区
        sizeof(inputBuffer),          // 输入缓冲区大小
        outputBuffer,                 // 输出缓冲区
        sizeof(outputBuffer),         // 输出缓冲区大小
        &bytesReturned,               // 返回的字节数
        NULL                          // 重叠结构
    )) {
        printf("DeviceIoControl failed: %d\n", GetLastError());
        CloseHandle(hDevice);
        return 1;
    }

    printf("Received from driver: %s\n", outputBuffer);

    CloseHandle(hDevice);
    return 0;
}
```

### 总结

1. **DriverEntry**: 注册 `AddDevice` 和其他 IRP 处理函数。
2. **AddDevice**: 创建设备对象并设置其属性。
3. **IRP 处理函数**: 处理各种 I/O 请求。
4. **Unload**: 清理驱动程序资源。
5. **IOCTL 代码**: 定义用户模式程序与驱动程序通信的接口。
6. **`.inf` 文件**: 安装驱动程序。
7. **用户模式程序**: 与驱动程序通信。

### 关键点

- **DriverEntry**: 不会自动创建设备对象，需要手动注册 `AddDevice` 函数。
- **AddDevice**: 在系统检测到新设备或驱动程序加载时被调用，用于创建设备对象。
- **设备对象**: 表示系统中的一个设备实例，是驱动程序与设备交互的基础。

## INITCODEG和PAGECODE

在 Windows 驱动开发中，`#pragma PAGEDCODE` 和 `#pragma INITCODE` 是两个常用的编译指示符，它们用于控制代码段的分页属性和初始化属性。下面是它们的作用和区别：

### `#pragma PAGEDCODE`

#### 作用
- **分页代码**：将标记的代码段标记为可分页的。这意味着这些代码可以在系统内存不足时被换出到磁盘，以释放物理内存。
- **性能影响**：由于代码可以被换出到磁盘，因此在频繁访问这些代码时可能会导致性能下降，因为需要从磁盘读取代码。

#### 使用场景
- **非关键路径**：通常用于那些不经常执行或对性能要求不高的代码，例如设备的配置、状态查询等。
- **减少内存占用**：有助于减少驱动程序在内存中的占用，特别是在内存紧张的情况下。

#### 示例
```c
#pragma PAGEDCODE
NTSTATUS DefaultPnpHandler(PDEVICE_EXTENSION pdx, PIRP Irp)
{
    PAGED_CODE();
    L_INFO("Enter DefaultPnpHandler\n");
    IoSkipCurrentIrpStackLocation(Irp);
    L_INFO("Leave DefaultPnpHandler\n");
    return IoCallDriver(pdx->NextStackDevice, Irp);
}
```

### `#pragma INITCODE`

#### 作用
- **初始化代码**：将标记的代码段标记为初始化代码。这些代码在驱动加载时执行一次，之后不再执行。
- **内存管理**：初始化代码在执行后可以被系统释放，从而节省内存。

#### 使用场景
- **驱动加载时**：通常用于驱动加载时的初始化操作，例如注册设备、分配资源等。
- **一次性任务**：适用于只需要在驱动加载时执行一次的任务。

#### 示例
```c
#pragma INITCODE
extern "C"
NTSTATUS DriverEntry(IN PDRIVER_OBJECT pDriverObject,
                     IN PUNICODE_STRING pRegistryPath)
{
    L_INFO("Enter DriverEntry\n");
    pDriverObject->DriverExtension->AddDevice = HelloWDMAddDevice;
    pDriverObject->MajorFunction[IRP_MJ_PNP] = HelloWDMPnp;
    pDriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] =
    pDriverObject->MajorFunction[IRP_MJ_CREATE] =
    pDriverObject->MajorFunction[IRP_MJ_READ] =
    pDriverObject->MajorFunction[IRP_MJ_WRITE] = HelloWDMDispatchRoutine;
    pDriverObject->DriverUnload = HelloWDMUnload;

    L_INFO("Leave DriverEntry\n");
    return STATUS_SUCCESS;
}
```

### 区别

1. **代码分页**：
   - `#pragma PAGEDCODE`：代码可以被分页到磁盘，适合非关键路径的代码。
   - `#pragma INITCODE`：代码在驱动加载时执行一次，之后可以被释放，适合初始化代码。

2. **内存管理**：
   - `#pragma PAGEDCODE`：有助于减少内存占用，但在频繁访问时可能会影响性能。
   - `#pragma INITCODE`：有助于节省内存，因为初始化代码在执行后可以被释放。

3. **使用场景**：
   - `#pragma PAGEDCODE`：适用于不经常执行的代码，如设备配置、状态查询等。
   - `#pragma INITCODE`：适用于驱动加载时的初始化操作，如注册设备、分配资源等。

### 总结
- `#pragma PAGEDCODE` 用于标记可以被分页的代码，适合非关键路径的代码。
- `#pragma INITCODE` 用于标记初始化代码，适合驱动加载时的初始化操作。

通过合理使用这两个编译指示符，可以优化驱动程序的内存管理和性能。


`IRP_MJ_PNP` 是 Windows 驱动程序模型中用于处理 Plug and Play (PnP) 操作的主要 IRP 类型。每个 `IRP_MJ_PNP` 请求都包含一个次要功能代码（Minor Function Code），这些次要功能代码用于指定具体的 PnP 操作。以下是一些常见的 `IRP_MJ_PNP` 次要功能代码及其触发时机：

### 常见的 `IRP_MJ_PNP` 次要功能代码

1. **IRP_MN_START_DEVICE**
   - **触发时机**：当设备第一次插入系统或从挂起状态恢复时。
   - **作用**：通知驱动程序设备已准备好进行操作。

2. **IRP_MN_QUERY_STOP_DEVICE**
   - **触发时机**：当设备即将停止时，系统会发送此 IRP 以询问驱动程序是否允许停止设备。
   - **作用**：驱动程序可以拒绝停止请求，或者进行必要的清理工作。

3. **IRP_MN_STOP_DEVICE**
   - **触发时机**：在 `IRP_MN_QUERY_STOP_DEVICE` 成功后，系统会发送此 IRP 以实际停止设备。
   - **作用**：驱动程序应释放资源并停止设备的正常操作。

4. **IRP_MN_CANCEL_STOP_DEVICE**
   - **触发时机**：如果在 `IRP_MN_QUERY_STOP_DEVICE` 之后，系统决定取消停止设备，会发送此 IRP。
   - **作用**：驱动程序应撤销 `IRP_MN_QUERY_STOP_DEVICE` 中所做的任何准备工作。

5. **IRP_MN_QUERY_REMOVE_DEVICE**
   - **触发时机**：当设备即将被移除时，系统会发送此 IRP 以询问驱动程序是否允许移除设备。
   - **作用**：驱动程序可以拒绝移除请求，或者进行必要的清理工作。

6. **IRP_MN_REMOVE_DEVICE**
   - **触发时机**：在 `IRP_MN_QUERY_REMOVE_DEVICE` 成功后，系统会发送此 IRP 以实际移除设备。
   - **作用**：驱动程序应释放所有资源并完成设备的移除操作。

7. **IRP_MN_CANCEL_REMOVE_DEVICE**
   - **触发时机**：如果在 `IRP_MN_QUERY_REMOVE_DEVICE` 之后，系统决定取消移除设备，会发送此 IRP。
   - **作用**：驱动程序应撤销 `IRP_MN_QUERY_REMOVE_DEVICE` 中所做的任何准备工作。

8. **IRP_MN_QUERY_ID**
   - **触发时机**：当系统需要获取设备的标识信息时。
   - **作用**：驱动程序应返回设备的标识信息，如硬件 ID、兼容 ID 等。

9. **IRP_MN_QUERY_CAPABILITIES**
   - **触发时机**：当系统需要获取设备的能力信息时。
   - **作用**：驱动程序应返回设备的能力信息，如支持的电源状态、DMA 能力等。

10. **IRP_MN_FILTER_RESOURCE_REQUIREMENTS**
    - **触发时机**：当系统需要获取设备的资源需求时。
    - **作用**：驱动程序可以修改或过滤设备的资源需求。

11. **IRP_MN_READ_CONFIG**
    - **触发时机**：当系统需要读取设备的配置信息时。
    - **作用**：驱动程序应返回设备的配置信息。

12. **IRP_MN_WRITE_CONFIG**
    - **触发时机**：当系统需要写入设备的配置信息时。
    - **作用**：驱动程序应更新设备的配置信息。

13. **IRP_MN_EJECT**
    - **触发时机**：当用户请求弹出或移除设备时。
    - **作用**：驱动程序应执行必要的操作以安全地弹出或移除设备。

14. **IRP_MN_SET_LOCK**
    - **触发时机**：当系统需要锁定或解锁设备时。
    - **作用**：驱动程序应执行必要的操作以锁定或解锁设备。

15. **IRP_MN_QUERY_DEVICE_RELATIONS**
    - **触发时机**：当系统需要获取设备的关系信息时。
    - **作用**：驱动程序应返回设备的关系信息，如子设备列表、总线关系等。

16. **IRP_MN_QUERY_INTERFACE**
    - **触发时机**：当系统需要查询设备的接口信息时。
    - **作用**：驱动程序应返回设备的接口信息。

17. **IRP_MN_QUERY_DEVICE_TEXT**
    - **触发时机**：当系统需要获取设备的文本信息时。
    - **作用**：驱动程序应返回设备的文本信息，如设备描述、友好名称等。

### 示例代码

以下是一个简单的示例，展示了如何在驱动程序中处理 `IRP_MJ_PNP` 请求。

```c
#include <ntddk.h>

VOID UnloadDriver(PDRIVER_OBJECT DriverObject) {
    UNREFERENCED_PARAMETER(DriverObject);
    KdPrint(("Driver unloaded\n"));
}

NTSTATUS AddDevice(PDRIVER_OBJECT DriverObject, PDEVICE_OBJECT PhysicalDeviceObject) {
    UNREFERENCED_PARAMETER(DriverObject);
    UNREFERENCED_PARAMETER(PhysicalDeviceObject);
    KdPrint(("AddDevice called\n"));
    return STATUS_SUCCESS;
}

NTSTATUS DispatchPnp(PDEVICE_OBJECT DeviceObject, PIRP Irp) {
    PIO_STACK_LOCATION irpStack = IoGetCurrentIrpStackLocation(Irp);
    NTSTATUS status = STATUS_SUCCESS;

    switch (irpStack->MinorFunction) {
    case IRP_MN_START_DEVICE:
        KdPrint(("IRP_MN_START_DEVICE\n"));
        // 处理设备启动
        break;

    case IRP_MN_QUERY_STOP_DEVICE:
        KdPrint(("IRP_MN_QUERY_STOP_DEVICE\n"));
        // 处理设备停止查询
        break;

    case IRP_MN_STOP_DEVICE:
        KdPrint(("IRP_MN_STOP_DEVICE\n"));
        // 处理设备停止
        break;

    case IRP_MN_QUERY_REMOVE_DEVICE:
        KdPrint(("IRP_MN_QUERY_REMOVE_DEVICE\n"));
        // 处理设备移除查询
        break;

    case IRP_MN_REMOVE_DEVICE:
        KdPrint(("IRP_MN_REMOVE_DEVICE\n"));
        // 处理设备移除
        break;

    case IRP_MN_QUERY_ID:
        KdPrint(("IRP_MN_QUERY_ID\n"));
        // 处理设备标识查询
        break;

    case IRP_MN_QUERY_CAPABILITIES:
        KdPrint(("IRP_MN_QUERY_CAPABILITIES\n"));
        // 处理设备能力查询
        break;

    default:
        KdPrint(("Unknown IRP_MJ_PNP minor function: %x\n", irpStack->MinorFunction));
        status = STATUS_INVALID_DEVICE_REQUEST;
        break;
    }

    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);

    DriverObject->DriverUnload = UnloadDriver;
    DriverObject->DriverExtension->AddDevice = AddDevice;
    DriverObject->MajorFunction[IRP_MJ_PNP] = DispatchPnp;

    KdPrint(("Driver loaded successfully\n"));
    return STATUS_SUCCESS;
}
```

### 解释

1. **`DispatchPnp` 函数**：处理 `IRP_MJ_PNP` 请求的派发函数。
2. **`irpStack->MinorFunction`**：获取当前 IRP 的次要功能代码。
3. **`switch` 语句**：根据不同的次要功能代码处理相应的 PnP 操作。
4. **`Irp->IoStatus.Status` 和 `Irp->IoStatus.Information`**：设置 IRP 的状态和信息字段。
5. **`IoCompleteRequest`**：完成 IRP 请求。

### 总结

- **`IRP_MJ_PNP`**：用于处理 PnP 操作的主要 IRP 类型。
- **次要功能代码**：每个次要功能代码对应一个具体的 PnP 操作，触发时机各不相同。
- **处理方法**：在驱动程序中通过 `DispatchPnp` 函数处理 `IRP_MJ_PNP` 请求，并根据不同的次要功能代码执行相应的操作。


挂载子设备
在 Windows 驱动程序中，使用 `FILE_DEVICE_UNKNOWN` 设备类型创建的设备对象仍然可以支持子设备的挂载。关键在于如何正确处理 `IRP_MJ_PNP` 请求中的 `IRP_MN_QUERY_DEVICE_RELATIONS` 次要功能代码。

### `IRP_MN_QUERY_DEVICE_RELATIONS`

`IRP_MN_QUERY_DEVICE_RELATIONS` 用于查询设备的关系信息，特别是子设备列表。当系统需要了解某个设备的子设备时，会发送此 IRP。驱动程序需要构建一个 `DEVICE_RELATIONS` 结构，并返回给系统。

### 示例代码

以下是一个示例，展示了如何在驱动程序中处理 `IRP_MN_QUERY_DEVICE_RELATIONS` 请求，以返回子设备列表。

#### 驱动程序入口点

```c
#include <ntddk.h>

typedef struct _DEVICE_EXTENSION {
    PDEVICE_OBJECT Self;
    PDEVICE_OBJECT ChildDevice;
} DEVICE_EXTENSION, *PDEVICE_EXTENSION;

VOID UnloadDriver(PDRIVER_OBJECT DriverObject) {
    UNREFERENCED_PARAMETER(DriverObject);
    KdPrint(("Driver unloaded\n"));
}

NTSTATUS AddDevice(PDRIVER_OBJECT DriverObject, PDEVICE_OBJECT PhysicalDeviceObject) {
    PDEVICE_OBJECT pDeviceObject;
    PDEVICE_EXTENSION pDeviceExtension;
    NTSTATUS status;
    UNICODE_STRING usDeviceName;

    RtlInitUnicodeString(&usDeviceName, L"\\Device\\MyParentDevice");

    status = IoCreateDevice(
        DriverObject,
        sizeof(DEVICE_EXTENSION),
        &usDeviceName,
        FILE_DEVICE_UNKNOWN,
        0,
        FALSE,
        &pDeviceObject
    );

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create parent device object: %x\n", status));
        return status;
    }

    pDeviceExtension = (PDEVICE_EXTENSION)pDeviceObject->DeviceExtension;
    pDeviceExtension->Self = pDeviceObject;

    // 创建子设备
    UNICODE_STRING usChildDeviceName;
    RtlInitUnicodeString(&usChildDeviceName, L"\\Device\\MyChildDevice");

    status = IoCreateDevice(
        DriverObject,
        sizeof(DEVICE_EXTENSION),
        &usChildDeviceName,
        FILE_DEVICE_UNKNOWN,
        0,
        FALSE,
        &pDeviceExtension->ChildDevice
    );

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create child device object: %x\n", status));
        IoDeleteDevice(pDeviceObject);
        return status;
    }

    KdPrint(("Devices created successfully\n"));
    return STATUS_SUCCESS;
}

NTSTATUS DispatchPnp(PDEVICE_OBJECT DeviceObject, PIRP Irp) {
    PDEVICE_EXTENSION pDeviceExtension = (PDEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpStack = IoGetCurrentIrpStackLocation(Irp);
    NTSTATUS status = STATUS_SUCCESS;

    switch (irpStack->MinorFunction) {
    case IRP_MN_QUERY_DEVICE_RELATIONS: {
        PDEVICE_RELATIONS deviceRelations;
        PDEVICE_OBJECT childDevice = pDeviceExtension->ChildDevice;
        ULONG size = sizeof(DEVICE_RELATIONS) + sizeof(PDEVICE_OBJECT);

        deviceRelations = (PDEVICE_RELATIONS)ExAllocatePoolWithTag(NonPagedPool, size, 'DEVR');
        if (!deviceRelations) {
            status = STATUS_INSUFFICIENT_RESOURCES;
            break;
        }

        deviceRelations->Count = 1;
        deviceRelations->Objects[0] = childDevice;
        ObReferenceObject(childDevice);

        Irp->IoStatus.Information = (ULONG_PTR)deviceRelations;
        break;
    }

    default:
        KdPrint(("Unknown IRP_MJ_PNP minor function: %x\n", irpStack->MinorFunction));
        status = STATUS_INVALID_DEVICE_REQUEST;
        break;
    }

    Irp->IoStatus.Status = status;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);

    DriverObject->DriverUnload = UnloadDriver;
    DriverObject->DriverExtension->AddDevice = AddDevice;
    DriverObject->MajorFunction[IRP_MJ_PNP] = DispatchPnp;

    KdPrint(("Driver loaded successfully\n"));
    return STATUS_SUCCESS;
}
```

### 解释

1. **设备扩展结构**：定义了一个 `DEVICE_EXTENSION` 结构，用于存储设备对象和子设备对象的指针。
2. **`AddDevice` 函数**：
   - 创建父设备对象。
   - 创建子设备对象，并将其指针存储在父设备的设备扩展中。
3. **`DispatchPnp` 函数**：
   - 处理 `IRP_MN_QUERY_DEVICE_RELATIONS` 请求。
   - 分配 `DEVICE_RELATIONS` 结构，并填充子设备对象的指针。
   - 将 `DEVICE_RELATIONS` 结构的指针设置为 `Irp->IoStatus.Information`。
   - 引用子设备对象，以防止其在 IRP 处理过程中被删除。
4. **`DriverEntry` 函数**：
   - 设置卸载回调函数。
   - 设置 `AddDevice` 回调函数。
   - 设置 `IRP_MJ_PNP` 的派发函数。

### 关键点

- **`DEVICE_RELATIONS` 结构**：用于存储子设备对象的指针列表。
- **引用计数**：使用 `ObReferenceObject` 增加子设备对象的引用计数，以防止其在 IRP 处理过程中被删除。
- **内存分配**：使用 `ExAllocatePoolWithTag` 分配 `DEVICE_RELATIONS` 结构的内存，并在 IRP 完成后释放内存。

### 总结

- **`FILE_DEVICE_UNKNOWN`**：使用 `FILE_DEVICE_UNKNOWN` 设备类型创建的设备对象仍然可以支持子设备的挂载。
- **`IRP_MN_QUERY_DEVICE_RELATIONS`**：通过正确处理 `IRP_MN_QUERY_DEVICE_RELATIONS` 请求，可以返回子设备列表。

在 Windows 驱动开发中，读写操作通常分为两种主要模式：**驱动缓冲区读写** 和 **直接读写**。这两种模式在数据传输方式和内存管理上有所不同。下面分别详细介绍这两种模式：

### 驱动缓冲区读写 (Buffered I/O)

#### 特点
- **内核缓冲区**：系统为每个 I/O 操作分配一个内核缓冲区。
- **数据复制**：数据从用户缓冲区复制到内核缓冲区，然后再从内核缓冲区复制到设备或从设备复制到内核缓冲区，最后从内核缓冲区复制回用户缓冲区。
- **简单实现**：驱动程序只需处理内核缓冲区中的数据，不需要关心用户缓冲区的地址。

#### 优点
- **安全性**：由于数据复制，避免了用户缓冲区和内核缓冲区之间的直接访问，提高了系统的安全性。
- **简化驱动开发**：驱动程序只需要处理内核缓冲区中的数据，减少了复杂性。

#### 缺点
- **性能开销**：额外的数据复制操作增加了 CPU 负载和内存带宽消耗。
- **内存占用**：每次 I/O 操作都需要额外的内核缓冲区，增加了内存占用。

#### 使用场景
- 适用于小规模数据传输，或者对性能要求不高的场景。

### 直接读写 (Direct I/O)

#### 特点
- **用户缓冲区**：系统直接使用用户提供的缓冲区进行数据传输。
- **无数据复制**：数据直接从用户缓冲区传输到设备或从设备传输到用户缓冲区，没有中间的内核缓冲区。
- **内存映射**：用户缓冲区被映射到内核地址空间，驱动程序可以直接访问这些缓冲区。

#### 优点
- **高性能**：减少了数据复制操作，降低了 CPU 负载和内存带宽消耗。
- **低延迟**：数据传输路径更短，减少了数据传输的延迟。

#### 缺点
- **复杂性**：驱动程序需要处理用户缓冲区的地址，增加了驱动开发的复杂性。
- **安全性**：直接访问用户缓冲区可能会导致安全问题，需要谨慎处理。

#### 使用场景
- 适用于大规模数据传输，或者对性能要求较高的场景。

### 总结

- **驱动缓冲区读写**：适合小规模数据传输，对性能要求不高，但对安全性和简单性有较高要求的场景。
- **直接读写**：适合大规模数据传输，对性能要求高，能够承受复杂性的场景。



在 Windows 驱动开发中，内存管理是一个非常重要的部分。驱动程序需要有效地分配和回收内存，以确保系统的稳定性和性能。以下是一些常见的内存分配与回收方法及其注意事项：

### 内存分配

#### 1. **ExAllocatePoolWithTag**
这是最常用的内存分配函数，适用于大多数情况下的内存分配。

```c
PVOID ExAllocatePoolWithTag(
  POOL_TYPE PoolType,
  SIZE_T NumberOfBytes,
  ULONG Tag
);
```

- **PoolType**：指定内存池类型，常见的有 `NonPagedPool`（非分页池）和 `PagedPool`（分页池）。
- **NumberOfBytes**：请求分配的字节数。
- **Tag**：一个 4 字节的标记，用于调试和跟踪内存分配。

**示例**：
```c
PVOID buffer = ExAllocatePoolWithTag(NonPagedPool, 1024, 'ABCD');
if (buffer == NULL) {
    // 处理内存分配失败的情况
}
```

#### 2. **MmAllocateContiguousMemory**
用于分配连续的物理内存，适用于需要连续物理地址的场景。

```c
PVOID MmAllocateContiguousMemory(
  SIZE_T NumberOfBytes,
  PHYSICAL_ADDRESS HighestAcceptableAddress
);
```

- **NumberOfBytes**：请求分配的字节数。
- **HighestAcceptableAddress**：最高可接受的物理地址。

**示例**：
```c
PVOID contiguousBuffer = MmAllocateContiguousMemory(1024, (PHYSICAL_ADDRESS)-1);
if (contiguousBuffer == NULL) {
    // 处理内存分配失败的情况
}
```

### 内存回收

#### 1. **ExFreePoolWithTag**
用于释放通过 `ExAllocatePoolWithTag` 分配的内存。

```c
VOID ExFreePoolWithTag(
  PVOID P,
  ULONG Tag
);
```

- **P**：要释放的内存指针。
- **Tag**：与分配时使用的标记相同。

**示例**：
```c
ExFreePoolWithTag(buffer, 'ABCD');
```

#### 2. **MmFreeContiguousMemory**
用于释放通过 `MmAllocateContiguousMemory` 分配的内存。

```c
VOID MmFreeContiguousMemory(
  PVOID BaseAddress
);
```

- **BaseAddress**：要释放的内存指针。

**示例**：
```c
MmFreeContiguousMemory(contiguousBuffer);
```

### 注意事项

1. **内存泄漏**：确保在不再需要内存时及时释放，避免内存泄漏。
2. **错误处理**：在分配内存时检查返回值，确保分配成功。
3. **内存池选择**：根据内存使用场景选择合适的内存池类型。非分页池适用于频繁访问且不能分页的内存，分页池适用于可以分页的内存。
4. **调试标记**：使用 `Tag` 参数可以帮助调试和跟踪内存分配，建议使用有意义的标记。

### 总结

- **ExAllocatePoolWithTag** 和 **ExFreePoolWithTag** 是最常用的内存分配和回收函数，适用于大多数情况。
- **MmAllocateContiguousMemory** 和 **MmFreeContiguousMemory** 用于分配和释放连续的物理内存，适用于特定场景。
- 注意内存管理和错误处理，避免内存泄漏和系统不稳定。


在 Windows 驱动开发中，链表是一种常用的数据结构，用于动态地存储和管理一系列数据项。Windows 提供了一些内置的链表操作函数和宏，使得在驱动程序中使用链表变得相对简单和高效。以下是关于如何在驱动程序中使用链表的详细说明。

## 1. 基本概念

### 单向链表 vs 双向链表

- **单向链表**：每个节点只有一个指向下一个节点的指针。
- **双向链表**：每个节点有两个指针，一个指向下一个节点，另一个指向前一个节点。

在 Windows 驱动开发中，通常使用双向链表，因为它们提供了更好的灵活性和效率。

### LIST_ENTRY 结构体

Windows 提供了一个通用的双向链表节点结构体 `LIST_ENTRY`，定义如下：

```c
typedef struct _LIST_ENTRY {
    struct _LIST_ENTRY *Flink;
    struct _LIST_ENTRY *Blink;
} LIST_ENTRY, *PLIST_ENTRY;
```

- **Flink**：指向下一个节点的指针。
- **Blink**：指向上一个节点的指针。

## 2. 初始化链表

在使用链表之前，必须先初始化它。可以使用 `InitializeListHead` 宏来初始化一个空的链表头。

```c
#include <ntddk.h>

LIST_ENTRY myList;

void InitializeMyList() {
    InitializeListHead(&myList);
}
```

## 3. 插入节点

Windows 提供了多个宏来插入节点到链表中。常用的宏包括：

- **InsertTailList**: 在链表尾部插入节点。
- **InsertHeadList**: 在链表头部插入节点。
- **InsertAfter**: 在指定节点之后插入新节点。
- **InsertBefore**: 在指定节点之前插入新节点。

### 示例：在链表尾部插入节点

```c
typedef struct _MY_NODE {
    LIST_ENTRY entry;
    int data;
} MY_NODE, *PMY_NODE;

void InsertNodeAtTail(int value) {
    PMY_NODE newNode = (PMY_NODE)ExAllocatePoolWithTag(PagedPool, sizeof(MY_NODE), 'NODE');
    if (newNode == NULL) {
        // 处理内存分配失败的情况
        return;
    }

    newNode->data = value;
    InsertTailList(&myList, &newNode->entry);
}
```

### 示例：在链表头部插入节点

```c
void InsertNodeAtHead(int value) {
    PMY_NODE newNode = (PMY_NODE)ExAllocatePoolWithTag(PagedPool, sizeof(MY_NODE), 'NODE');
    if (newNode == NULL) {
        // 处理内存分配失败的情况
        return;
    }

    newNode->data = value;
    InsertHeadList(&myList, &newNode->entry);
}
```

### 示例：在指定节点之后插入节点

```c
void InsertNodeAfter(PMY_NODE existingNode, int value) {
    PMY_NODE newNode = (PMY_NODE)ExAllocatePoolWithTag(PagedPool, sizeof(MY_NODE), 'NODE');
    if (newNode == NULL) {
        // 处理内存分配失败的情况
        return;
    }

    newNode->data = value;
    InsertAfter(&existingNode->entry, &newNode->entry);
}
```

### 示例：在指定节点之前插入节点

```c
void InsertNodeBefore(PMY_NODE existingNode, int value) {
    PMY_NODE newNode = (PMY_NODE)ExAllocatePoolWithTag(PagedPool, sizeof(MY_NODE), 'NODE');
    if (newNode == NULL) {
        // 处理内存分配失败的情况
        return;
    }

    newNode->data = value;
    InsertBefore(&existingNode->entry, &newNode->entry);
}
```

## 4. 删除节点

Windows 提供了 `RemoveEntryList` 宏来删除链表中的节点。

```c
BOOLEAN RemoveEntryList(
    PLIST_ENTRY EntryToRemove
);
```

- 返回值：如果链表为空，则返回 `FALSE`；否则返回 `TRUE`。

### 示例：删除指定节点

```c
void DeleteNode(PMY_NODE nodeToDelete) {
    RemoveEntryList(&nodeToDelete->entry);
    ExFreePoolWithTag(nodeToDelete, 'NODE');
}

// 或者遍历链表找到并删除特定值的节点
void DeleteNodeByValue(int value) {
    PLIST_ENTRY currentEntry = myList.Flink;
    while (currentEntry != &myList) {
        PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
        if (currentNode->data == value) {
            RemoveEntryList(&currentNode->entry);
            ExFreePoolWithTag(currentNode, 'NODE');
            break; // 如果需要删除所有匹配的节点，可以移除此行
        }
        currentEntry = currentEntry->Flink;
    }
}
```

## 5. 遍历链表

遍历链表通常使用 `IsListEmpty` 宏来检查链表是否为空，并使用 `CONTAINING_RECORD` 宏将 `LIST_ENTRY` 结构转换为自定义节点结构。

### 示例：遍历链表并打印节点数据

```c
void TraverseAndPrintList() {
    if (IsListEmpty(&myList)) {
        DbgPrint("The list is empty.\n");
        return;
    }

    PLIST_ENTRY currentEntry = myList.Flink;
    while (currentEntry != &myList) {
        PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
        DbgPrint("Node Data: %d\n", currentNode->data);
        currentEntry = currentEntry->Flink;
    }
}
```

## 6. 清空链表

清空链表意味着删除链表中的所有节点并释放相应的内存。

### 示例：清空链表

```c
void ClearList() {
    while (!IsListEmpty(&myList)) {
        PLIST_ENTRY currentEntry = RemoveHeadList(&myList);
        PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
        ExFreePoolWithTag(currentNode, 'NODE');
    }
}
```

## 7. 注意事项

1. **内存管理**：
   - **分配内存**：在插入节点前，确保正确分配内存。
   - **释放内存**：在删除节点后，确保释放内存，避免内存泄漏。

2. **同步机制**：
   - 如果链表在多个线程之间共享，需要使用适当的同步机制（如自旋锁、互斥锁）来保护链表操作，防止并发访问带来的问题。

3. **错误处理**：
   - 在分配内存时，始终检查返回值，确保分配成功。
   - 在删除节点时，确保节点确实存在于链表中，避免非法操作。

4. **调试**：
   - 使用调试工具（如 WinDbg）来检查链表的状态，确保链表操作的正确性。

## 8. 示例代码汇总

以下是一个完整的示例，展示了如何在驱动程序中使用链表进行基本的操作。

```c
#include <ntddk.h>

typedef struct _MY_NODE {
    LIST_ENTRY entry;
    int data;
} MY_NODE, *PMY_NODE;

LIST_ENTRY myList;

void InitializeMyList() {
    InitializeListHead(&myList);
}

void InsertNodeAtTail(int value) {
    PMY_NODE newNode = (PMY_NODE)ExAllocatePoolWithTag(PagedPool, sizeof(MY_NODE), 'NODE');
    if (newNode == NULL) {
        DbgPrint("Failed to allocate memory for new node.\n");
        return;
    }

    newNode->data = value;
    InsertTailList(&myList, &newNode->entry);
}

void InsertNodeAtHead(int value) {
    PMY_NODE newNode = (PMY_NODE)ExAllocatePoolWithTag(PagedPool, sizeof(MY_NODE), 'NODE');
    if (newNode == NULL) {
        DbgPrint("Failed to allocate memory for new node.\n");
        return;
    }

    newNode->data = value;
    InsertHeadList(&myList, &newNode->entry);
}

void DeleteNode(PMY_NODE nodeToDelete) {
    RemoveEntryList(&nodeToDelete->entry);
    ExFreePoolWithTag(nodeToDelete, 'NODE');
}

void TraverseAndPrintList() {
    if (IsListEmpty(&myList)) {
        DbgPrint("The list is empty.\n");
        return;
    }

    PLIST_ENTRY currentEntry = myList.Flink;
    while (currentEntry != &myList) {
        PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
        DbgPrint("Node Data: %d\n", currentNode->data);
        currentEntry = currentEntry->Flink;
    }
}

void ClearList() {
    while (!IsListEmpty(&myList)) {
        PLIST_ENTRY currentEntry = RemoveHeadList(&myList);
        PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
        ExFreePoolWithTag(currentNode, 'NODE');
    }
}

VOID DriverUnload(PDRIVER_OBJECT DriverObject) {
    ClearList();
    DbgPrint("Driver unloaded.\n");
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);

    InitializeMyList();

    // 测试插入节点
    InsertNodeAtTail(10);
    InsertNodeAtHead(20);
    InsertNodeAtTail(30);

    // 打印链表
    TraverseAndPrintList();

    // 删除节点
    PLIST_ENTRY currentEntry = myList.Flink;
    while (currentEntry != &myList) {
        PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
        if (currentNode->data == 20) {
            DeleteNode(currentNode);
            break;
        }
        currentEntry = currentEntry->Flink;
    }

    // 再次打印链表
    TraverseAndPrintList();

    // 设置卸载回调函数
    DriverObject->DriverUnload = DriverUnload;

    DbgPrint("Driver loaded successfully.\n");
    return STATUS_SUCCESS;
}
```

### 代码说明

1. **初始化链表**：
   ```c
   void InitializeMyList() {
       InitializeListHead(&myList);
   }
   ```

2. **插入节点**：
   ```c
   void InsertNodeAtTail(int value) {
       PMY_NODE newNode = (PMY_NODE)ExAllocatePoolWithTag(PagedPool, sizeof(MY_NODE), 'NODE');
       if (newNode == NULL) {
           DbgPrint("Failed to allocate memory for new node.\n");
           return;
       }

       newNode->data = value;
       InsertTailList(&myList, &newNode->entry);
   }

   void InsertNodeAtHead(int value) {
       PMY_NODE newNode = (PMY_NODE)ExAllocatePoolWithTag(PagedPool, sizeof(MY_NODE), 'NODE');
       if (newNode == NULL) {
           DbgPrint("Failed to allocate memory for new node.\n");
           return;
       }

       newNode->data = value;
       InsertHeadList(&myList, &newNode->entry);
   }
   ```

3. **删除节点**：
   ```c
   void DeleteNode(PMY_NODE nodeToDelete) {
       RemoveEntryList(&nodeToDelete->entry);
       ExFreePoolWithTag(nodeToDelete, 'NODE');
   }
   ```

4. **遍历链表**：
   ```c
   void TraverseAndPrintList() {
       if (IsListEmpty(&myList)) {
           DbgPrint("The list is empty.\n");
           return;
       }

       PLIST_ENTRY currentEntry = myList.Flink;
       while (currentEntry != &myList) {
           PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
           DbgPrint("Node Data: %d\n", currentNode->data);
           currentEntry = currentEntry->Flink;
       }
   }
   ```

5. **清空链表**：
   ```c
   void ClearList() {
       while (!IsListEmpty(&myList)) {
           PLIST_ENTRY currentEntry = RemoveHeadList(&myList);
           PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
           ExFreePoolWithTag(currentNode, 'NODE');
       }
   }
   ```

6. **驱动卸载**：
   ```c
   VOID DriverUnload(PDRIVER_OBJECT DriverObject) {
       ClearList();
       DbgPrint("Driver unloaded.\n");
   }
   ```

7. **驱动入口**：
   ```c
   NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
       UNREFERENCED_PARAMETER(RegistryPath);

       InitializeMyList();

       // 测试插入节点
       InsertNodeAtTail(10);
       InsertNodeAtHead(20);
       InsertNodeAtTail(30);

       // 打印链表
       TraverseAndPrintList();

       // 删除节点
       PLIST_ENTRY currentEntry = myList.Flink;
       while (currentEntry != &myList) {
           PMY_NODE currentNode = CONTAINING_RECORD(currentEntry, MY_NODE, entry);
           if (currentNode->data == 20) {
               DeleteNode(currentNode);
               break;
           }
           currentEntry = currentEntry->Flink;
       }

       // 再次打印链表
       TraverseAndPrintList();

       // 设置卸载回调函数
       DriverObject->DriverUnload = DriverUnload;

       DbgPrint("Driver loaded successfully.\n");
       return STATUS_SUCCESS;
   }
   ```

## 9. 进一步阅读

为了更好地掌握链表在驱动开发中的应用，可以参考以下资源：

- **Microsoft 文档**：[Windows 驱动程序文档](https://docs.microsoft.com/en-us/windows-hardware/drivers/)
- **书籍**：《Programming the Microsoft Windows Driver Model》 by Walter Oney
- **博客和教程**：许多技术博客和在线教程提供了详细的链表操作示例和最佳实践。
