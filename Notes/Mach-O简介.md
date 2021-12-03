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

首先进入 **mach-o/loader.h** 

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

可见有两种类型的Mach-O Header，通过magic区分
