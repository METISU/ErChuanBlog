# Mach-O简介

Mach-O是基于Mach内核的操作系统（例如iOS、macOS等）上的一种文件格式，包含可执行文件、目标代码或者Frameword等，可以先看下可执行文件的大致结构：

![Mach-O](https://user-images.githubusercontent.com/22512175/144608764-033425aa-f378-4cec-823a-229a56a29c16.png)

可以看到 Mach-O 主要由 3 部分组成:

* Mach Header: 主要描述了CPU架构、文件类型以及load command数量等
* Load Command: 表示文件如何加载进内存
* Data: 用于存放数据以及代码
  * Segments: Segments是具有特定权限的一组内存
  * Section: 更细维度的划分，每个Segments可能拥有0个或多个section

## Mach-O Header

在 **mach-o/loader.h** 中可见mach_header申明

``` C
/*
 * The 32-bit mach header appears at the very beginning of the object file for
 * 32-bit architectures.
 */
struct mach_header {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
};

/* Constant for the magic field of the mach_header (32-bit architectures) */
#define	MH_MAGIC	0xfeedface	/* the mach magic number */
#define MH_CIGAM	0xcefaedfe	/* NXSwapInt(MH_MAGIC) */

/*
 * The 64-bit mach header appears at the very beginning of object files for
 * 64-bit architectures.
 */
struct mach_header_64 {
	uint32_t	magic;		/* mach magic number identifier */
	cpu_type_t	cputype;	/* cpu specifier */
	cpu_subtype_t	cpusubtype;	/* machine specifier */
	uint32_t	filetype;	/* type of file */
	uint32_t	ncmds;		/* number of load commands */
	uint32_t	sizeofcmds;	/* the size of all the load commands */
	uint32_t	flags;		/* flags */
	uint32_t	reserved;	/* reserved */
};

/* Constant for the magic field of the mach_header_64 (64-bit architectures) */
#define MH_MAGIC_64 0xfeedfacf /* the 64-bit mach magic number */
#define MH_CIGAM_64 0xcffaedfe /* NXSwapInt(MH_MAGIC_64) */
```

* magic：表示当前Mach-O运行的架构，32位的是0xfeedface、0xcefaedfe，64位的是0xfeedfacf、0xcffaedfe（有大小端之分）
* cputype、cpusubtype：表示当前的Mach-O支持的CPU类型，在mach/machine.h头文件中可见
* filetype：表示当前文件的类型，如`#define	MH_OBJECT	0x1		/* relocatable object file */`等
* ncmds、sizeofcmds：load commands 的条数以及大小

下面看一个Mach-O头文件的内容

![Mach-O](https://user-images.githubusercontent.com/22512175/147305261-e011d0c3-b6c8-409a-af86-83a431ed7cf6.png)

可见这个Mach-O为64位，可运行于x86_64 cpu架构，是一个可执行文件

### Fat Header

Fat Header是对多架构的二进制文件的打包集合，比如同时集成32位以及64位的文件，在**mach-o/fat.h**头文件进行查看

``` C
#define FAT_MAGIC	0xcafebabe
#define FAT_CIGAM	0xbebafeca	/* NXSwapLong(FAT_MAGIC) */

struct fat_header {
	uint32_t	magic;		/* FAT_MAGIC or FAT_MAGIC_64 */
	uint32_t	nfat_arch;	/* number of structs that follow */
};

struct fat_arch {
	cpu_type_t	cputype;	/* cpu specifier (int) */
	cpu_subtype_t	cpusubtype;	/* machine specifier (int) */
	uint32_t	offset;		/* file offset to this object file */
	uint32_t	size;		/* size of this object file */
	uint32_t	align;		/* alignment as a power of 2 */
};

/*
 * The support for the 64-bit fat file format described here is a work in
 * progress and not yet fully supported in all the Apple Developer Tools.
 *
 * When a slice is greater than 4mb or an offset to a slice is greater than 4mb
 * then the 64-bit fat file format is used.
 */
#define FAT_MAGIC_64	0xcafebabf
#define FAT_CIGAM_64	0xbfbafeca	/* NXSwapLong(FAT_MAGIC_64) */

struct fat_arch_64 {
	cpu_type_t	cputype;	/* cpu specifier (int) */
	cpu_subtype_t	cpusubtype;	/* machine specifier (int) */
	uint64_t	offset;		/* file offset to this object file */
	uint64_t	size;		/* size of this object file */
	uint32_t	align;		/* alignment as a power of 2 */
	uint32_t	reserved;	/* reserved */
};
```

fat header有自己的magic，fat header后跟着多个fat_arch结构体，nfat_arch表示这些结构体的数量（同样区分32位、64位）
fat_arch则描述了这个fat Mach-O里面有哪几种CPU架构等，以及该架构相对于当前文件的偏移量、大小等（即对应的Mach-O的偏移量）

![Fat Header](https://user-images.githubusercontent.com/22512175/147314456-56723b12-3f4e-49bf-bb92-d9d59f2e6ac1.png)

图片该Far Header集成了两种Mach-O，一种是x86_64的，还有ARM64的（ARM的cupsubtype是80000002）

## Load Commands
Load Commands在Mach-O Header之后，提供了如何将文件加载如内存的指令，可在**mach-o/loader.h**中进行查看

每个load command前两个字段必须是 cmd 和 cmdsize

``` C
struct load_command {
	uint32_t cmd;		/* type of load command */
	uint32_t cmdsize;	/* total size of command in bytes */
};
```

> 每个load command都有自己的结构体，但是Mach-O的作者提供的一个通用的结构体

从注释中可以看出，如果知道了cmd，就能知道这是哪个类型的load command，也就能转换成响应的load command了

## Segments && Section

首先看几个相对来说接触的比较多个Segment（同样位于mach/loader.h）

``` C
#define	SEG_PAGEZERO	"__PAGEZERO"	/* the pagezero segment which has no */
					/* protections and catches NULL */
					/* references for MH_EXECUTE files */

#define	SEG_TEXT	"__TEXT"	/* the tradition UNIX text segment */

#define	SEG_DATA	"__DATA"	/* the tradition UNIX data segment */

#define	SEG_OBJC	"__OBJC"	/* objective-C runtime segment */

#define	SEG_LINKEDIT	"__LINKEDIT"	/* the segment containing all structs */
					/* created and maintained by the link */
					/* editor.  Created with -seglinkedit */
					/* option to ld(1) for MH_EXECUTE and */
					/* FVMLIB file types only */
```

* SEG_PAGEZERO：当为MH_EXECUTE文件时存在，该区域没有读、写、执行权限，所以指针引向这边会发生crash，一般是64位程序的低32位
* SEG_TEXT：可读可执行区域，一般用于存放代码以及一些不可变数据
* SEG_DATA：可读可写数据段
* SEG_OBJC：Objective-C runtime 段
* SEG_LINKEDIT：只读权限，存放符号表、entitlements等


