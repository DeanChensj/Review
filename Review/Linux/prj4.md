#<center>Linux Project4 <br>File System homework
**<p align = right>
	5120309491<br>陈上劼
</p>**

---
- Modify the Super.c/Makefile (kernel source of romfs)
- Create a directory which contains some subdir and file, then use genromfs to generate the image
- Mount that image to some empty directory
- Loaded the module, which has 3 kind of characteristics.
 -  'Hide file name'	 

			There are aa and fo/aa in the origin directory.
			>ls t
			>ls t/fo
			No 'aa' can be found
 - 'Encrypte file'

			Say aa’s original content is ‘bbbbbbb
			With the change, cat t/bb output ‘ccccccccc’
 - 'Make file executable'

			Without code changes ‘ls –l t’, output is ‘-rw-r--r--’
			With the change, output is ‘-rwxr-xr-x’

-------
###1. Modify super.c
####1.1 Add params
	static char *hided_file_name="null";
	module_param(hided_file_name, charp, S_IRUGO);
	
####1.2. Modify the  .iterate file_operation 
ls指令涉及到文件操作符中的遍历操作，于是我们需对此进行修改。

	static int romfs_readdir(struct file *file, struct dir_context *ctx)
	{
		...
		for (;;) {
			// 越界返回
			if (!offset || offset >= maxoff) {
				...
				goto out;
			}
			ctx->pos = offset; //调整文件当前位置
	
			//获取inode信息
			ret = romfs_dev_read(i->i_sb, offset, &ri, ROMFH_SIZE);
			if (ret < 0)
				goto out;
	
			// 获取路径名长度
			j = romfs_dev_strnlen(i->i_sb, offset + ROMFH_SIZE,
					      sizeof(fsname) - 1);
			if (j < 0)
				goto out;

			// 遍历的终点
			ret = romfs_dev_read(i->i_sb, offset + ROMFH_SIZE, fsname, j);
			if (ret < 0)
				goto out;
			fsname[j] = '\0';
	
			ino = offset;
			nextfh = be32_to_cpu(ri.next);  /* convert cpu's byte order to big endian*/ 
	
			// 若当前文件名与参数文件名一致，跳过，不予显示	
	+		if(strcmp(fsname, hided_file_name)==0){
	+			goto dean;
	+		}		
			
			// 处理hard link
			if ((nextfh & ROMFH_TYPE) == ROMFH_HRD)
				ino = be32_to_cpu(ri.spec);

			/* Passing a dentry to parent_ino() */
			if (!dir_emit(ctx, fsname, j, ino,
				    romfs_dtype_table[nextfh & ROMFH_TYPE]))
				goto out;
	+ dean:
			//更新调整目录偏移量，并和硬件进行连接，方便下次读取
			offset = nextfh & ROMFH_MASK; 
		}
	  out:
		return 0;
	}
####1.3. Modify romfs_readpage
为了使cat 文件时输出加密信息，我们需对读文件操作符进行修改。

	static int romfs_readpage(struct file *file, struct page *page)
	{
		//获取当前inode
		struct inode *inode = page->mapping->host;
		loff_t offset, size; //long long 类型
		unsigned long fillsize, pos;
		void *buf;
		int ret;
		
		//对一个物理页进行分配
		buf = kmap(page);
		if (!buf)
			return -ENOMEM;
		
		/* 32 bit warning -- but not for us :) */
		offset = page_offset(page);
		size = i_size_read(inode);
		fillsize = 0;
		ret = 0;
		if (offset < size) {
			size -= offset;
			fillsize = size > PAGE_SIZE ? PAGE_SIZE : size;
	
			pos = ROMFS_I(inode)->i_dataoffset + offset;
	
			ret = romfs_dev_read(inode->i_sb, pos, buf, fillsize);
			if (ret < 0) {
				SetPageError(page);
				fillsize = 0;
				ret = -EIO;
			}
		}
		
		//获取当前文件名
	+	char *fname = file->f_path.dentry->d_iname;
	+	int i;
	+	printk(KERN_INFO"filename: %s \n", fname);
		
		如果文件名匹配，修改输出的文件内容
	+	if(strcmp(fname, hided_file_name)==0){
	+		for(i = 0; i < fillsize; i++)
	+			*((char *)buf + i) = 'c';
	+	}
	
		if (fillsize < PAGE_SIZE)
			memset(buf + fillsize, 0, PAGE_SIZE - fillsize);
		if (ret == 0)
			SetPageUptodate(page);
	
		flush_dcache_page(page);
		kunmap(page);
		unlock_page(page);
		return ret;
	}
####1.4. Modify romfs_dir_inode_operations
文件的权限信息存储于Inode中，因此我们需对inode操作符进行修改。

	static struct dentry *romfs_lookup(struct inode *dir, struct dentry *dentry,
					   unsigned int flags)
	{
		unsigned long offset, maxoff;
		struct inode *inode;
		struct romfs_inode ri; //romfs文件头，包含4个区块
		const char *name;		/* got from dentry */
		int len, ret;
	
		offset = dir->i_ino & ROMFH_MASK; //获取偏移量
		ret = romfs_dev_read(dir->i_sb, offset, &ri, ROMFH_SIZE);
		if (ret < 0)
			goto error;
	
		/* search all the file entries in the list starting from the one
		 * pointed to by the directory's special data */
		maxoff = romfs_maxsize(dir->i_sb);
		offset = be32_to_cpu(ri.spec) & ROMFH_MASK;
	
		name = dentry->d_name.name;
		len = dentry->d_name.len;
	
		for (;;) {
			...
		}
	
		...

		//获取当前inode
		inode = romfs_iget(dir->i_sb, offset);
	
		// 若当前文件名匹配， 修改文件权限
	+	if(strcmp(name, hided_file_name)==0){
	+		inode->i_mode |= S_IXUGO;
	+		printk(KERN_INFO"file: %s executable \n", name);
	+	}
	
		...

	}
###2. Modify Makefile
	obj-$(CONFIG_ROMFS_FS) += romfs.o
	
	romfs-y := storage.o super.o
	
	KDIR := /lib/modules/$(shell uname -r)/build
	PWD := $(shell pwd)
	
	all:
		make -C $(KDIR) SUBDIRS=$(PWD) modules
	
	clean:
		rm *.o *.ko *.mod.c Module.symvers Module.markers modules.order -f

###3. Mount the image and load the module
		dean@ubuntu:$ mkdir dir
		dean@ubuntu:$ cd dir
		...
		# 在目录下创建各种目录与文件...		
		...
		dean@ubuntu:$ sudo apt-get install genromfs 
		dean@ubuntu:$ sudo genromfs -f test.img  #将当前目录制作为img
		dean@ubuntu:$ Mount –o loop test.img <some empty dir>
		dean@ubuntu:$ sudo insmod romfs hided_file_name="aa“
###4. Result
 	
	dean@ubuntu:~/Desktop/tmp$  find tmp
	tmp
	tmp/test.img
	tmp/fo
	tmp/bb
	dean@ubuntu:~/Desktop/tmp$ ls
	bb  fo  test.img
	dean@ubuntu:~/Desktop/tmp$ ls aa
	dean@ubuntu:~/Desktop/tmp$ cat fo/aa
	cccccccccccccccccccc

	dean@ubuntu:~/Desktop/tmp$ ls -l fo/aa
	-rwxr-xr-x 1 root root 2 Jun  4  2015 fo/aa



