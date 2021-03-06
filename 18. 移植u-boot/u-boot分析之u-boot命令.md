## u-boot分析之u-boot命令 (2017.12.10)
```
// Command.c (common)
__u_boot_cmd_start  //有时候定义不在.c,.h文件里面，有可能在链接文件里面(.dis)
__u_boot_cmd_end
```
1. `u-boot命令分析`
```
#define Struct_Section  __attribute__ ((unused,section (".u_boot_cmd")))

bootcmd=nand read.jffs2 0x30007FC0 kernel; 
bootm 0x30007FC0

U_BOOT_CMD(						//宏
 	bootm,	CFG_MAXARGS,	1,	do_bootm,
 	"bootm   - boot application image from memory\n",		//usage
	//以下这段之间没有 , or ; 可作为一段字符串(help)
 	"[addr [arg ...]]\n    - boot application image stored in memory\n"
 	"\tpassing arguments 'arg ...'; when booting a Linux kernel,\n"
 	"\t'arg' can be the address of an initrd image\n"
#ifdef CONFIG_OF_FLAT_TREE
	"\tWhen booting a Linux kernel which requires a flat device-tree\n"
	"\ta third argument is required which is the address of the of the\n"
	"\tdevice-tree blob. To boot that kernel without an initrd image,\n"
	"\tuse a '-' for the second argument. If you do not pass a third\n"
	"\ta bd_info struct will be passed instead\n"
#endif
);

#define U_BOOT_CMD(name,maxargs,rep,cmd,usage,help) \
cmd_tbl_t __u_boot_cmd_##name Struct_Section = {#name, maxargs, rep, cmd, usage, help}

cmd_tbl_t __u_boot_cmd_bootm __attribute__ ((unused,section (".u_boot_cmd"))) = 
{"bootm", CFG_MAXARGS, 1, do_bootm, usage, help}
/*
__u_boot_cmd_bootm ： 结构体
__attribute__  ： 结构体的属性,强制把section段属性设置为 .u_boot_cmd,跟 u-boot.dis的段属性对应起来
#	. = .;
#	__u_boot_cmd_start = .;
#	.u_boot_cmd : { *(.u_boot_cmd) }
#	__u_boot_cmd_end = .;
*/
```
##  `创建一个hello的u-boot命令`
1. `新建cmd_hello.c 保存在 /common/下 ,内容如下：`
```
#include <common.h>
#include <watchdog.h>
#include <command.h>
#include <image.h>
#include <malloc.h>
#include <zlib.h>
#include <bzlib.h>
#include <environment.h>
#include <asm/byteorder.h>
int do_hello (cmd_tbl_t *cmdtp, int flag, int argc, char *argv[])
{
	int i;
	printf ("Hello world!, %d\n",argc);

	for (i =0 ; i < argc) 
	{
		printf("argv[%d]: %s",i,argv[i]);
	}
	return 0;
}
U_BOOT_CMD(
 	hello,	CFG_MAXARGS,	1,	do_hello,
 	"hello   - just for test\n",
 	"hello,long help .............\n"
);
```
2. `vim /work/system/u-boot-1.1.6/common/Makefile 添加 cmd_hello.o` 
* 注意这里的 cmd_hello 必须与添加到 /common/目录下的文件名相同(不包括后缀)
* virtex2.o xilinx.o crc16.o xyzModem.o cmd_mac.o cmd_hello.o	
3. `cd /work/system/u-boot-1.1.6/`	
4. `make distclean`
```
make clean与make distclean的区别
make clean仅仅是清除之前编译的可执行文件及配置文件。 
而make distclean要清除所有生成的文件。
```
5. `make 100ask24x0_config `
6. `make`
7. `选择 NAND flash 启动,进入到 u-boot的界面`
```
OpenJTAG> help
hello   - just for test

OpenJTAG> hello
Hello world!, 1
argv[0]: helloOpenJTAG> hello
Hello world!, 1
argv[0]: helloOpenJTAG> help hello 
hello hello,long help .............

OpenJTAG> hello arg gag
Hello world!, 3
argv[0]: helloargv[1]: argargv[2]: gagOpenJTAG>
```
8. `实现 hello 的 u-boot命令`
## u-boot命令图解
![u-boot命令图解](https://github.com/GalenDeng/Embedded-Linux/blob/master/18.%20%E7%A7%BB%E6%A4%8Du-boot/u-boot%E5%91%BD%E4%BB%A4%E5%9B%BE%E7%89%87%E7%AC%94%E8%AE%B0/u-boot%E5%91%BD%E4%BB%A4%E5%9B%BE%E8%A7%A3.JPG)
9. `flinfo --- 查看 NOR flash 的信息` -- `flash information`
* 可以查看 NOR flash 的型号、型号、各扇区的开始地址、是否只读等信息
```
OpenJTAG> flinfo

Bank # 1: MXIC MX29LV160B FLASH (16 x 16)  Size: 2 MB in 35 Sectors
  AMD Standard command set, Manufacturer ID: 0xC2, Device ID: 0x2249
  Erase timeout: 30000 ms, write timeout: 100 ms

  Sector Start Addresses:
  00000000   RO   00004000   RO   00006000   RO   00008000   RO   00010000   RO 
  00020000   RO   00030000   RO   00040000        00050000        00060000      
  00070000        00080000        00090000        000A0000        000B0000      
  000C0000        000D0000        000E0000        000F0000        00100000      
  00110000        00120000        00130000        00140000        00150000      
  00160000        00170000        00180000        00190000        001A0000      
  001B0000        001C0000        001D0000        001E0000        001F0000      
OpenJTAG> 
```
* MX29LV160B : NOR flash 的型号
*  Size: 2 MB : NOR flash 的 大小
* RO : 只读  处于写保护状态
* 对于只读的扇区，在擦除、烧写它之前，要先解除写保护，
* 命令 ： protect off all //解除所有的 NOR Flash 的写保护
* erase : 擦除 
```
erase start end : 擦除地址范围 start - end 
erase start + len : 擦除地址范围 start ~ start + tlen - 1
erase all : 擦除所有 NOR Flash
```
* 这里若要擦除前五个分区，命令为： erase 0 0x2ffff ,而不是 erase 0 0x30000
10. `go 命令`
```
* tftp 0x30000000 test.bin or nfs 0x30000000 192.168.99.140:/work/nfs_root/test.bin // 下载可执行文件到内存中
* go 0x30000000			// 直接执行程序
```