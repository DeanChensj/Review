#<center>Linux Project3 <br>Memory management homework
**<p align = right>
	5120309491<br>陈上劼
</p>**

---
- Write a module that is called mtest 
- When module loaded, module will create a proc fs entry **/proc/mtest**
- /proc/mtest will accept 3 kind of input
 -  “listvma” will print all vma of current process in the format of
 			
			start-addr end-addr permission
			e.g
			0x10000 0x20000 rwx
			0x30000 0x40000 r—

 - “findpage addr” will find va->pa translation of address in current process’s mm context and print it. If there is not va->pa translation, prink “tranlsation not found”
 - “writeval addr val” will change an unsigned long size content in current
process’s virtual address into val. Note module should write to identity
mapping address of addr and verify it from userspace address addr.
-  All the print can be done with printk and check result with dmesg.

-------
###0. Preprocess
	#include <linux/module.h>  
	#include <linux/kernel.h>  
	#include <linux/proc_fs.h>  
	#include <linux/string.h>  
	#include <linux/vmalloc.h>  
	#include <linux/sched.h>
	#include <linux/init.h>  
	#include <linux/slab.h>  
	#include <linux/mm.h>  
	#include <linux/vmalloc.h>
	#include <linux/highmem.h> 
	#include <asm/uaccess.h> 
###1. Main module
	static struct proc_dir_entry *mtest_proc_entry;  
	  
	static int __init mtest_init(void)  
	{  
		/* 在 /proc 中创建入口  */    
		mtest_proc_entry = proc_create("mtest", 0777, NULL, &proc_mtest_operations);  
	    if (mtest_proc_entry == NULL) {  
	        printk("Error creating proc entry\nf");  
	        return -1;  
	    }  
	    printk("Succeed creating proc entry\n");    
	    return 0;  
	}  
	  
	static void __exit mtest_exit(void)  
	{  
	    printk("Good bye!\n");    
	    remove_proc_entry("mtest", NULL);  
	}  
	  
	module_init(mtest_init);  
	module_exit(mtest_exit); 
###2. Define file operation

	static struct file_operations proc_mtest_operations = {  
	    .write        = mtest_proc_write  
	};  
	
	static ssize_t mtest_proc_write(struct file *file, 
					const char __user * buffer,  
	                                size_t count, loff_t * data)  
	{        
	    char buf[128];  
	    unsigned long val, val2;  
	    if (count > sizeof(buf))  
	        return -EINVAL;  
	      
	    if (copy_from_user(buf, buffer, count))  
	        return -EINVAL;  
	      
		/* 使用 memcmp() 来比较字符串 */
	    if (memcmp(buf, "listvma", 7) == 0)   
	        mtest_list_vma();  
	
	    else if (memcmp(buf, "findpage", 8) == 0){
	        if (sscanf(buf + 8 + 1, "%lx", &val) == 1) 
	            mtest_find_page(val);
		else
		    printk("Wrong parameter for findpage\n");  
	    }
	  
	    else  if (memcmp(buf, "writeval", 8) == 0){   
	        if (sscanf(buf + 8 + 1, "%lx %lx", &val, &val2) == 2)  
	            mtest_write_val(val, val2);   
		else
		   printk("Wrong parameter for writeval");
	    }
	    else
	        printk("No such command. Nothing to be done.\n");
	    return count;  
	}  
###3. List vma
	static void mtest_list_vma(void)  
	{  
		/* 所属的内存描述符,层次最高 */
	    struct mm_struct *mm = current->mm;   
		/* 虚拟内存区域(virtual memory areas) */ 
	    struct vm_area_struct *vma;  
	     r
		/*
		*读者加锁函数down_read用于加读者锁，如果没有写者操作时，
		*等待队列为空，读者可以加读者锁，将信号量的读者计数加1。
		*如果有写在操作时，等待队列非空，读者需要等待写者操作完成。
		*/
	    down_read(&mm->mmap_sem); 
	    for (vma = mm->mmap/* 指向虚拟区间（VMA）链表 */;vma; vma = vma->vm_next) {  
	        printk("VMA 0x%lx-0x%lx ", vma->vm_start, vma->vm_end);  
	        if (vma->vm_flags & VM_READ)   printk("r"); 
		else  printk("-");
	        if (vma->vm_flags & VM_WRITE)  printk("w");  
		else  printk("-");
	        if (vma->vm_flags & VM_EXEC)   printk("x");
		else  printk("-");  
	        printk("\n");  
	    }  
	    up_read(&mm->mmap_sem); //release lock 
	}  
###4. Find page translation  
	static void mtest_find_page(unsigned long addr)  
	{    
	    struct vm_area_struct *vma;  
	    struct mm_struct *mm = current->mm;    
	    struct page *page;  
	    unsigned long kernel_addr;
	    
	    down_read(&mm->mmap_sem);  
		/*寻找第一个满足addr < vma_end的vma */
	    vma = find_vma(mm, addr);  
		/* 自定义函数： 寻找addr所在页表 *
	    page = mtest_seek_page(vma, addr); /
	  
	    if (!page)   
	    {      
	        printk("Translation not found\n");  
	        up_read(&mm->mmap_sem); 
	  	return;
	    }  
	
		 /* 翻译成物理地址 */
	    kernel_addr = (unsigned long)page_address(page);  
	    kernel_addr += (addr&~PAGE_MASK); 
	    printk("Translate 0x%lx to kernel address 0x%lx\n", addr, kernel_addr); 
	    up_read(&mm->mmap_sem);    
	}  

	static struct page* mtest_seek_page(struct vm_area_struct *vma, unsigned long addr)  
	{ 
	    pgd_t *pgd;  //Page Global Directory (页目录) 
	    pud_t *pud;  //second level page table
	    pmd_t *pmd;  //Page Middle Directory (页目录)  
	    pte_t *pte;  //Page Table Entry  (页表项)
	    spinlock_t *ptl;  
	
	    struct page *page = NULL;  
	    struct mm_struct *mm = vma->vm_mm;  
	    
	    pgd = pgd_offset(mm, addr);   
		/* 页表项是否为0 */   	/* 含有脏数据 */    
	    if (pgd_none(*pgd) || unlikely(pgd_bad(*pgd)))  return NULL;  
	          
	    pud = pud_offset(pgd, addr);  
	    if (pud_none(*pud) || unlikely(pud_bad(*pud)))  return NULL;  
	                
	    pmd = pmd_offset(pud, addr);  
	    if (pmd_none(*pmd) || unlikely(pmd_bad(*pmd)))  return NULL;
	  
	    pte = pte_offset_map_lock(mm, pmd, addr, &ptl);  
	    if (!pte)  return NULL; 
	
		/*页表项是否可用。当页在内存中但是不可读写时置此标志 */
	    if (!pte_present(*pte)){
	        pte_unmap_unlock(pte, ptl);
		return NULL;
	    }  
	          
	    page = pfn_to_page(pte_pfn(*pte));  
	    if (!page){  
	        pte_unmap_unlock(pte, ptl);
		return NULL;
	    }  
	        
	    get_page(page);  
	     /* 释放进程页面锁，同时，如果支持将页表放到高端内存，就解除对页表的映射。*/
	    pte_unmap_unlock(pte, ptl);  
	    return page;  
	}  
###5. Write val to address
	static void mtest_write_val(unsigned long addr, unsigned long val)  
	{  
	    struct vm_area_struct *vma;  
	    struct mm_struct *mm = current->mm;  
	    struct page *page;  
	    unsigned long kernel_addr;  
	  
	    down_read(&mm->mmap_sem);  
	    vma = find_vma(mm, addr);  
	    if (vma && addr >= vma->vm_start && (addr + sizeof(val)) < vma->vm_end) {  
	        if (!(vma->vm_flags & VM_WRITE)) {  
	            printk("vma is not writable for 0x%lx\n", addr);  
	            up_read(&mm->mmap_sem);
		    return; 
	        }  
	        page = mtest_seek_page(vma, addr);  
	        if (!page) {      
	            printk("Page 0x%lx not found\n", addr);  
	            up_read(&mm->mmap_sem);
		    return;
	        }  
	          
	        kernel_addr = (unsigned long)page_address(page);  
	        kernel_addr += (addr&~PAGE_MASK); 
		*(unsigned long *)kernel_addr = val; 
	        printk("Written 0x%lx to address 0x%lx\n", val, kernel_addr);  
		    
			/*      
			* put_page用来完成物理页面与一个线性地址页面的挂接，从而将一个
			* 线性地址空间内的页面落实到物理地址空间内
		    */    
			put_page(page);  
	          
	    } 
	    else  printk("Wrong vma\n");  
	    up_read(&mm->mmap_sem);
	}  
###6. In the Bash
 	
	dean@ubuntu:~/Linux/Project3$ make
	dean@ubuntu:~/Linux/Project3$ sudo insmod mtest.ko
	
	dean@ubuntu:~/Linux/Project3$ sudo echo "listvma" > /proc/mtest
	dean@ubuntu:~/Linux/Project3$ dmesg | tail -l
	[ 2567.429728] VMA 0xb77be000-0xb77c1000 rw-
	[ 2567.429735] VMA 0xb77d4000-0xb77d5000 rw-
	[ 2567.429741] VMA 0xb77d5000-0xb77d6000 r--
	[ 2567.429748] VMA 0xb77d6000-0xb77d8000 rw-
	[ 2567.429754] VMA 0xb77d8000-0xb77da000 r--
	[ 2567.429761] VMA 0xb77da000-0xb77dc000 r-x
	[ 2567.429767] VMA 0xb77dc000-0xb77fc000 r-x
	[ 2567.429774] VMA 0xb77fc000-0xb77fd000 r--
	[ 2567.429780] VMA 0xb77fd000-0xb77fe000 rw-
	[ 2567.429787] VMA 0xbfde1000-0xbfe03000 rw-

	dean@ubuntu:~/Linux/Project3$sudo echo "findpage b77d5000"  > /proc/mtest 
	dean@ubuntu:~/Linux/Project3$ dmesg | tail -1
	[ 2736.563457] Translation not found

	dean@ubuntu:~/Linux/Project3$ sudo echo "writeval ffd0a8fc 3"  > /proc/mtest 
	dean@ubuntu:~/Linux/Project3$ dmesg | tail -1
	[ 3033.790280] Written 0x3 to address 0xffd0a8fc
###7. test.c
	/*
	*程序中有一个整数型变量，比方说名字是 v，初始值是 0。程序启动后即等
	*待输入命令。输入命令“write 整数 i”，程序写/proc/mtest，从而将 v 的值改成 i。输入*命令“print”，
	*程序用 printf(“%d\n”,v)输出 v 的值。
	*/
	#include <stdio.h>
	
	int main()
	{
	  unsigned long v = 0, i;
	  unsigned long addr = (unsigned long) &v;
	  char cmd[1024];  
	  scanf("%s %lx", cmd, &i);
	  FILE *fout = fopen("/proc/mtest", "w");
	  if(strcmp(cmd, "write") == 0)
	  {
		fprintf(fout, "writeval %lx %lx", addr, i);
	   	fflush(fout);
	        printf("%lx\n",v);
	  } 
	  fclose(fout);
	  return 0;
	}
####Result
	dean@ubuntu:~/Linux/Project3$ ./test 
	write 50
	520
	dean@ubuntu:~/Linux/Project3$ dmesg | tail -1
	[  308.020991] Written 0x520 to address 0xcc064f4c


