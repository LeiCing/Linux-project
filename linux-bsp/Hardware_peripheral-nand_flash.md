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
      
diff --git a/drivers/mtd/mtdpart.c b/drivers/mtd/mtdpart.c
old mode 100644
new mode 100755
index 30149338..62797764
--- a/drivers/mtd/mtdpart.c
+++ b/drivers/mtd/mtdpart.c
@@ -33,10 +33,25 @@
 
 #include "mtdcore.h"
 
+#include <linux/mtd/nand.h> 
+
+#define MAX_PARTITION_MAPPING   2  
+struct part_map{  
+    struct mtd_info *part_mtd;  /* Mapping partition mtd */  
+    unsigned *map_table;        /* Mapping from logic block to phys block */  
+    unsigned nBlock;            /* Logic block number */  
+};  
+  
+static struct part_map *part_mapping[MAX_PARTITION_MAPPING];  
+static int part_mapping_count = -1;  
+
 /* Our partition linked list */
 static LIST_HEAD(mtd_partitions);
 static DEFINE_MUTEX(mtd_partitions_mutex);
 
+
+
+
 /* Our partition node structure */
 struct mtd_part {
 	struct mtd_info mtd;
@@ -52,6 +67,107 @@ struct mtd_part {
 #define PART(x)  ((struct mtd_part *)(x))
 
 
+/* nanzhong:
+ * This function create a partition mapping 
+ */
+static int part_create_partition_mapping ( struct mtd_info *part_mtd )
+{
+    struct mtd_part *part = PART(part_mtd);
+    struct nand_chip *this = part->master->priv;    
+    struct part_map *map_part;    
+    int index;
+    unsigned offset;
+    int logical_b, phys_b;
+
+    if ( !part_mtd || !this )
+    {
+        printk("null mtd or it is no nand chip!");
+        return -1;
+    }    
+    
+    if ( part_mapping_count < 0 )
+    {
+        /* Init the part mapping table when this function called first time */
+        memset(part_mapping, 0, sizeof(struct part_map *)*MAX_PARTITION_MAPPING);
+        part_mapping_count = 0;
+    }
+
+    for ( index=0; index<MAX_PARTITION_MAPPING; index++ )
+    {
+        if ( part_mapping[index] == NULL )
+            break;
+    }
+
+    if ( index >= MAX_PARTITION_MAPPING )
+    {
+        printk("partition mapping is full!");
+        return -1;
+    }
+
+    map_part = kmalloc(sizeof(struct part_map), GFP_KERNEL);
+    if ( !map_part )
+    {
+        printk ("memory allocation error while creating partitions mapping for %s/n",
+                part_mtd->name);     
+        return -1;
+    }
+
+    map_part->map_table = kmalloc(sizeof(unsigned)*(part_mtd->size>>this->bbt_erase_shift),
+                                  GFP_KERNEL);
+    if ( !map_part->map_table )
+    {
+        printk ("memory allocation error while creating partitions mapping for %s/n",
+                part_mtd->name);
+        kfree(map_part);
+        return -1;
+    }
+    memset(map_part->map_table, 0xFF, sizeof(unsigned)*(part_mtd->size>>this->bbt_erase_shift));
+
+    /* Create partition mapping table */
+    logical_b = 0;
+    for ( offset=0; offset<part_mtd->size; offset+=part_mtd->erasesize )
+    {
+        if ( part_mtd->_block_isbad &&
+             part_mtd->_block_isbad(part_mtd, offset) )
+             continue;        
+
+        phys_b = offset >> this->bbt_erase_shift;
+        map_part->map_table[logical_b] = phys_b;
+        printk("part[%s]: logic[%u]=phys[%u]\n", 
+              part_mtd->name, logical_b, phys_b);            
+        logical_b++;
+    }
+    map_part->nBlock = logical_b;
+    map_part->part_mtd = part_mtd;
+    
+    part_mapping[index] = map_part;
+    part_mapping_count++;
+    return 0;
+}
+
+static void part_del_partition_mapping( struct mtd_info *part_mtd )
+{
+    int index;
+    struct part_map *map_part;    
+
+    if ( part_mapping_count > 0 )
+    {
+        for ( index=0; index<MAX_PARTITION_MAPPING; index++ )
+        {
+            map_part = part_mapping[index];        
+            if ( map_part && map_part->part_mtd==part_mtd )
+            {
+                kfree(map_part->map_table);
+                kfree(map_part);
+                part_mapping[index] = NULL;                   
+                part_mapping_count--;
+            }
+        }
+    }
+}
+
+
+
 /*
  * MTD methods which simply translate the effective address and pass through
  * to the _real_ device.
@@ -63,6 +179,37 @@ static int part_read(struct mtd_info *mtd, loff_t from, size_t len,
 	struct mtd_part *part = PART(mtd);
 	struct mtd_ecc_stats stats;
 	int res;
+    
+    /* nanzhong: calucate physical address */
+    struct nand_chip *this = part->master->priv;
+    unsigned logic_b, phys_b;
+    unsigned i;
+
+    
+    if ( part_mapping_count > 0 )
+    {
+        for ( i=0; i<MAX_PARTITION_MAPPING; i++ )
+        {
+            if ( part_mapping[i] && part_mapping[i]->part_mtd==mtd )
+            {
+                /* remap from logic block to physical block */
+                logic_b = from >> this->bbt_erase_shift;
+                if ( logic_b < part_mapping[i]->nBlock )
+                {
+                    phys_b = part_mapping[i]->map_table[logic_b];
+                    from = phys_b << this->bbt_erase_shift | (from&(mtd->erasesize-1));
+                }
+                else
+                {
+                   /* the offset is bigger than good block range, don't read data */
+                    *retlen = 0;
+                    return -EINVAL;
+                }
+            }
+        }
+    }
+    
+
 
 	stats = part->master->ecc_stats;
 	res = part->master->_read(part->master, from + part->offset, len,
@@ -326,6 +473,7 @@ int del_mtd_partitions(struct mtd_info *master)
 	mutex_lock(&mtd_partitions_mutex);
 	list_for_each_entry_safe(slave, next, &mtd_partitions, list)
 		if (slave->master == master) {
+    		part_del_partition_mapping(&slave->mtd);
 			ret = del_mtd_device(&slave->mtd);
 			if (ret < 0) {
 				err = ret;
@@ -643,6 +791,12 @@ int add_mtd_partitions(struct mtd_info *master,
 
 		add_mtd_device(&slave->mtd);
 
+        /* nanzhong: Build partition mapping for squashfs */            
+        if ( slave->mtd.name && 0==strcmp(slave->mtd.name, "rootfs") )
+        {                                
+            part_create_partition_mapping(&slave->mtd);
+        }
+
 		cur_offset = slave->offset + slave->mtd.size;
 	}
 

解决参考网站：https://blog.csdn.net/sloan6/article/details/12620975
    





