# First

## 1.目录浏览器

```c
#include <unistd.h>
#include <stdio.h>
#include <dirent.h>
#include <string.h>
#include <sys/stat.h>
#include <stdlib.h>

/* struct dirent
 * 成员:
 * 1.ino_t d_ino:文件的inode节点号
 * 2.char d_name[]:文件的名字
*/

/* struct stat
 * 成员
 * 1.st_mode:文件权限和文件类型信息
 * 2.st_ino:与该文件关联的inode
 * 3.st_dev:保存文件的设备
 * 4.st_uid:文件属主的uid号
 * 5.st_gid:文件属组的gid号
 * 6.st_atime:文件上一次被访问的时间
 * 7.st_ctime:文件的权限，属主，组或内容上一次被修改的时间
 * st_mtime:文件的内容上一次被修改的时间
 * st_nlink:该文件上硬链接的个数
*/

/* S_ISDIR(宏定义)
 * 判断是否为一个目录，为目录则返回true，否则返回false（自己的猜测）
*/ 



void printdir(char *dir,int depth)
{
	DIR *dp;
	struct dirent *entry;
	struct stat statbuf;
	/* 这里是建立一个目录流，失败时返回一个空指针 */	
	if((dp = opendir(dir)) == NULL){
		/* stderr为标准错误输出，这里是将fprintf的输出输出到标准错误输出流 */
		fprintf(stderr,"cannot open directory: %s\n",dir);
	}
	/* 跳转到对应目录,因为后面的dir都是目录名字而不是完整路径，在这里使用这个函数，我们就可以跳到对应目录，之后输入目录名字也可以找到 */
	chdir(dir);
	/* 根据目录流读取其中的一个目录项 */
	while((entry = readdir(dp)) != NULL){
		/* 根据目录项中的文件名字将文件的状态信息赋值给statbuf */
		lstat(entry->d_name,&statbuf);
		if(S_ISDIR(statbuf.st_mode)){
			/* 这里是为了除去.和..目录 */
			if(strcmp(".",entry->d_name) == 0 || strcmp("..",entry->d_name) == 0){
				continue;
			}
			/* 见书中p96 */
			printf("%*s%s/\n",depth,"",entry->d_name);
			/* if is Dir ，then recurse. a new indent with depth+4 */
			printdir(entry->d_name,depth+4);
		}
		else printf("%*s%s\n",depth,"",entry->d_name);
	}
	chdir("..");
	/* 切断目录流与目录之间的关联，从而释放目录流，方便后续使用 */
	closedir(dp);
}

int main(int argc,char* argv[])
{
	char *topdir = ".";
	if(argc >= 2){
		topdir = argv[1];
	}
	printf("Directory scan of %s :\n",topdir);
	printf("=================================================================\n");
	printdir(topdir,0);
	printf("=================================================================\n");
	printf("done.\n");
	exit(0);
}
```







