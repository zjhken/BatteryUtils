# BatteryUtils

Thinkpad battery charge threshold utils

Exposes ACPI interface for battery controls.

1. Turn off the monitor
2. start charge threshold
3. stop charge threshold

* [Download BatteryUtils_N3.5](https://raw.githubusercontent.com/XYUU/BatteryUtils/master/BatteryUtils/bin/Release/BatteryUtils.exe)
* [Download BatteryUtils_N4](https://raw.githubusercontent.com/XYUU/BatteryUtils/master/BatteryUtils/bin/Release/BatteryUtils_N4.exe)

N4 version, win10 does not install.Net 3.5 framework can be used directly

N4 版本，win10 不安装.net 3.5 framework可直接使用


> from 作者: 更深入的研究后，使用5月2日的更新的方法可以测量出所有品牌笔记本设置电池充电阈值的方法。

This project is licensed under the GPLv3. See COPYING for details.
Copyright 2016-2017 XYUU

目前只支持Thinkpad笔记本设置电池充电阈值，将来可能会支持其他品牌笔记本，等待有兴趣的朋友一起完善。

开发思路见我的[知乎专栏](https://zhuanlan.zhihu.com/p/20706403)

知乎摘录:
# 自己开发Thinkpad电源管理程序

[![甘明](Untitled.assets/d31af46a30de54e80289a54e631315cf_xs.jpg)](https://www.zhihu.com/people/gan-ming-51)

[甘明](https://www.zhihu.com/people/gan-ming-51)

Spring Cloud中文网：https://springcloud.cc

关注他

43 人赞同了该文章

Thinkpad的电源管理程序，自Win8起就不再提供了，笔记本电池一直插在本子上，每天都会自动充满电，电池寿命每况愈下。通常如果不管的话，电池一年便损耗50%,两年之内就趋于报废。

Thinkpad官方的懒惰导致了电池更短的寿命，用户使用的成本越来越高，让本人颇为不爽。本着延长本本电池寿命，同时为保护环境节约能源贡献一份微薄力量，打算自己开发一套Thinkpad电池管理程序。

功能清单：

1、更改电池充电阈值；

2、即时调整电池充电状态；

3、调整硬盘电源状态；

4、调整显示器屏幕状态；

实现方法：

经查阅Linux下有tp_smapi项目[GitHub - evgeni/tp_smapi](https://link.zhihu.com/?target=https%3A//github.com/evgeni/tp_smapi)，可以控制电池管理，查阅源码得知其调用的是ACPI的EC控制器。关键代码是使用ASM汇编，在smapi_request方法中。

查阅Window内核驱动开发资料[Evaluating ACPI Control Methods (Windows Drivers)](https://link.zhihu.com/?target=https%3A//msdn.microsoft.com/zh-cn/library/windows/hardware/ff536139(v%3Dvs.85).aspx)，发现[IoBuildDeviceIoControlRequest routine (Windows Drivers)](https://link.zhihu.com/?target=https%3A//msdn.microsoft.com/zh-cn/library/windows/hardware/ff548318(v%3Dvs.85).aspx)方法即可直接操作ACPI方法，和硬件直接通信。但本人平时偶尔折腾Hackintosh考虑到可移植性，决定移植tp_smapi的汇编代码来实现。

今早，常松告诉我，Window在4月1日的时候发布了[“Ubuntu on Windows” 初体验_Linux新闻_Linux公社-Linux系统门户网站](https://link.zhihu.com/?target=http%3A//www.linuxidc.com/Linux/2016-04/129708.htm)和[Windows 10 Build 14316上手视频_Windows 10_cnBeta.COM](https://link.zhihu.com/?target=http%3A//www.cnbeta.com/articles/490329.htm)，约莫连移植都不用了，只用写个GUI就搞定？看来有福了。

4月20日更新

不过实在等不到Windows10的这个特性发布，而且将来发布了是否能支持这种底层硬件驱动也是不可知，原本打算改造tp_smapi的，实际实施过程中才知道tp_smapi中使用了大量linux底层驱动API，很难移植。遂决定自学一下windows内核驱动开发。

参照开源项目[GitHub - teleshoes/tpacpi-bat: ThinkPad ACPI Battery Util](https://link.zhihu.com/?target=https%3A//github.com/teleshoes/tpacpi-bat)得到了一份很不错的ACPI文档，得知thinkpad的设置电池充电阈值的几个ACPI方法分别是哪几个。

查MSDN得知[Evaluating ACPI Control Methods (Windows Drivers)](https://link.zhihu.com/?target=https%3A//msdn.microsoft.com/zh-cn/library/windows/hardware/ff536139(v%3Dvs.85).aspx)，是如何工作的。

查github得知[GitHub - reactos/reactos: ReactOS Mirror](https://link.zhihu.com/?target=https%3A//github.com/reactos/reactos)这里有示例代码。

5月2日更新

通过学习Windows内核驱动开发，得知如何调用DSDT中的Method，也参照代码文档写了一份C++的版本

```cpp
#define PciAllocateColdPoolWithTag(P,L,T)				ExAllocatePoolWithTag((POOL_TYPE)((P)|POOL_COLD_ALLOCATION),(L),T)

//
// check acpi method [checked]
//
BOOLEAN PciIsSlotPresentInParentMethod(__in PPCI_PDO_EXTENSION PdoExt,__in ULONG Method)
{
	PAGED_CODE();
	BOOLEAN Ret											=  FALSE;
	PACPI_EVAL_OUTPUT_BUFFER Output						= 0;

	__try
	{
		//
		// allocate output buffer
		//
		ULONG Length									= sizeof(ACPI_EVAL_OUTPUT_BUFFER) + sizeof(ACPI_METHOD_ARGUMENT) * 0x100;
		Output											= static_cast<PACPI_EVAL_OUTPUT_BUFFER>(PciAllocateColdPoolWithTag(PagedPool,Length,'BicP'));
		if(!Output)
			try_leave(Ret = FALSE);

		//
		// eval acpi method
		//
		ACPI_EVAL_INPUT_BUFFER Input;
		Input.MethodNameAsUlong							= Method;
		Input.Signature									= ACPI_EVAL_INPUT_BUFFER_SIGNATURE;
		PDEVICE_OBJECT DeviceObject						= PdoExt->ParentFdoExtension->PhysicalDeviceObject;
		NTSTATUS Status									= PciSendIoctl(DeviceObject,IOCTL_ACPI_EVAL_METHOD,&Input,sizeof(Input),Output,Length);
		if(!NT_SUCCESS(Status))
			try_leave(Ret = FALSE);

		//
		// search result buffer
		//
		ULONG Slot										= (PdoExt->Slot.u.bits.DeviceNumber << 16) | PdoExt->Slot.u.bits.FunctionNumber;
		for(ULONG i = 0; i < Output->Count; i ++)
		{
			if(Output->Argument[i].Type != ACPI_METHOD_ARGUMENT_INTEGER)
				try_leave(Ret = FALSE);

			if(Output->Argument[i].Argument == Slot)
				try_leave(Ret = TRUE);
		}
	}
	__finally
	{
		if(Output)
			ExFreePool(Output);
	}

	return Ret;
}

//
// send io control [checked]
//
NTSTATUS PciSendIoctl(__in PDEVICE_OBJECT DeviceObject,__in ULONG IoCode,__in PVOID Input,__in ULONG InputLength,__in PVOID Output,__in ULONG OutputLength)
{
	PAGED_CODE();

	DeviceObject										= IoGetAttachedDeviceReference(DeviceObject);
	if(!DeviceObject)
		return STATUS_INVALID_PARAMETER;

	NTSTATUS Status										= STATUS_SUCCESS;
	__try
	{
		KEVENT Event;
		KeInitializeEvent(&Event,SynchronizationEvent,FALSE);

		IO_STATUS_BLOCK Ios;
		PIRP Irp										= IoBuildDeviceIoControlRequest(IoCode,DeviceObject,Input,InputLength,Output,OutputLength,FALSE,&Event,&Ios);
		if(!Irp)
			try_leave(Status = STATUS_INSUFFICIENT_RESOURCES);

		Status											= IoCallDriver(DeviceObject,Irp);
		if(Status == STATUS_PENDING)
		{
			KeWaitForSingleObject(&Event,Executive,KernelMode,FALSE,0);
			Status										= Ios.Status;
		}
	}
	__finally
	{
		ObDereferenceObject(DeviceObject);
	}

	return Status;
}
```

但查阅示例代码时，却不知道如何得到PDEVICE_OBJECT对象；貌似非要通过内核驱动的DriverEntry驱动入口函数，让操作系统调用驱动时得到。

而Thinkpad的\_SB.PCI0.LPC.EC.HKEY设备已经使用了官方驱动，自己重写驱动并不是个好办法，自己写驱动程序虽然实现了控制电池充电阈值的功能，却会丧失我所不知道的其他功能特性。完美的解决方案应该是开发应用程序与内核驱动通信。在学习Window内核驱动开发时，知道应用程序和内核驱动通信的方法是通过调用Window中的DeviceIoControl方法来完成的。

DeviceIoControl方法有8个参数，其中最重要的参数是IoControlCode，该参数是由驱动开发者定义的，老夫没有文档呀！~

好在多年前玩过几年OD，说干就干，换块硬盘装了个Winxp，然后安装完Lenovo Power Manager应用，打开设置电池充电阈值窗口，用OD附加进程。下API断点bp DeviceIoControl，设置充电阈值为：

低于15开始充电 处于50停止充电，应用，成功进入断点。

```text
0007F3F8   00D31CC6  /CALL 到 DeviceIoControl 来自 PWRMGRIF.00D31CC0
0007F3FC   0000024C  |hDevice = 0000024C (window)
0007F400   00222638  |IoControlCode = 222638
0007F404   0007F42C  |InBuffer = 0007F42C     32 01 00 00
0007F408   00000004  |InBufferSize = 4
0007F40C   0007F424  |OutBuffer = 0007F424    00 00 00 00
0007F410   00000004  |OutBufferSize = 4
0007F414   0007F428  |pBytesReturned = 0007F428
0007F418   00000000  \pOverlapped = NULL
```

记下InBuffer处的数据为32 01 00 00，按F9执行，再次进入断点

```text
0007F3F8   00D31CC6  /CALL 到 DeviceIoControl 来自 PWRMGRIF.00D31CC0
0007F3FC   0000023C  |hDevice = 0000023C
0007F400   00222630  |IoControlCode = 222630
0007F404   0007F42C  |InBuffer = 0007F42C     0E 01 00 00
0007F408   00000004  |InBufferSize = 4
0007F40C   0007F424  |OutBuffer = 0007F424    70 F4 07 00
0007F410   00000004  |OutBufferSize = 4
0007F414   0007F428  |pBytesReturned = 0007F428
0007F418   00000000  \pOverlapped = NULL 
```

记下InBuffer处的数据为0E 01 00 00，两次进入断点的驱动句柄一致，但IoControlCode不同。

通过反复测量DeviceIoControl的数据情况：

```text
再次设置时 第一部分内容不变 第二次调InBuffer  1D 01 00 00  Outbuffer  70 F4 07 00
低于30开始充电 处于50停止充电

第三次设置
低于15开始充电 处于50停止充电 值和第一次一致

第四次设置
低于15开始充电 处于90停止充电 第一次调InBuffer 5A 01 00 00  第二次调InBuffer  0E 01 00 00

第五次设置
低于30开始充电 处于50停止充电 InBuffer 1D 01 00 00

第六次设置
低于30开始充电 处于90停止充电 第一次调InBuffer 5A 01 00 00  第二次调InBuffer  1D 01 00 00

第七次设置
低于15开始充电 处于90停止充电 第一次调InBuffer 5A 01 00 00  第二次调InBuffer  0E 01 00 00
```

5A的十进制=90，0E的十进制=14，可以确定222638是停止充电阈值控制指令，222630是开始充电阈值控制指令。InBuffer的第一位为设置的值，第二位大概是电池ID（能同时支持多块电池）。

通过这个调试的方法，再次拿到hDevice驱动句柄创建时调用的CreateFileW的文件名为“\\.\IBMPmDrv”，至此大功告成。

代码在此[GitHub - XYUU/BatteryUtils: Thinkpad battery charge threshold utils](https://link.zhihu.com/?target=https%3A//github.com/XYUU/BatteryUtils)


