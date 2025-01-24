1：nand flash基础知识


2：linux kernel 支持新的nand flash

在rv1106平台下，支持华邦windbond的nand flash芯片
sfc_nand.c中找到spi_nand_tbl这个支持flash列表

结构体变量含义：
static struct nand_info spi_nand_tbl[] = {
    // ... 现有芯片配置 ...
    
    /* 新增Winbond芯片 */
    { 
        .id0 = 0xEF,              /* Winbond的厂商ID */
        .id1 = 0xXX,              /* 具体型号ID */
        .id2 = 0xXX,              /* 可选的第三个ID */
        
        /* 基本参数配置 */
        .sec_size = 4,            /* 每个扇区的大小(KB) */
        .pages_per_sec = 0x40,    /* 每个扇区的页数 */
        .sec_per_page = 1,        /* 每页的扇区数 */
        .plane_per_die = 1024,    /* plane大小 */
        
        /* 状态寄存器配置 */
        .sta_reg_offs = 0x0C,     /* 状态寄存器偏移 */
        
        /* 容量相关 */
        .page_size_shift = 18,    /* 页大小偏移 */
        .ecc_bits = 0x8,          /* ECC位数 */
        
        /* 特性标志 */
        .feature = 1,             /* 特性标志 */
        
        /* 保护位配置 */
        .protection = { 0x04, 0x08, 0xFF, 0xFF },
        
        /* ECC状态获取函数 */
        .ecc_status = &sfc_nand_get_ecc_status0,  // 选择合适的ECC状态检查函数
    },
};

支持的windbond nand flash
![image](https://github.com/user-attachments/assets/f4c1f00c-7682-4b1d-b964-bac43bb29896)


如果芯片有特殊的ECC状态检查方式，需要添加新的ECC状态检查函数
如果芯片需要特殊的初始化配置，修改初始化相关代码
如果芯片有特殊的读写时序要求，可能需要修改读写函数
如果需要特殊的坏块管理或ECC处理，可能需要修改相关函数










XT26 nand flash坏块导致squashfs rootfs挂载失败
xt26 nand flash 有坏块，使用的只读文件根系统squashfs，不支持跳过坏块，导致挂载rootfs失败
![image](https://github.com/user-attachments/assets/bc7fc3e9-d1d7-4d6d-b36a-bfa54acc83e6)
![image](https://github.com/user-attachments/assets/d985bb39-dba0-489c-9730-6b511fd44499)

解决方案：建立物理块和逻辑块映射表
![image](https://github.com/user-attachments/assets/597b4110-497b-4be8-bcf6-8ab081d7fdb8)

code patch：
jump_badblock.patch

 


解决参考网站：https://blog.csdn.net/sloan6/article/details/12620975
    





