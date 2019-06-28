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

## 2.CD-Record\(copy from Beginning Linux Programming\)

```
#!/bin/bash

#注意shell中默认定义的变量为global，也就是全局的
#要想定义局部变量，就显示的在变量前写local

menu_choice=""
current_cd=""
title_file="title.cdb"
tracks_file="tracks.cdb"
temp_file=/tmp/cdb.$$
trap 'rm -f $temp_file' EXIT

# Now we define our functions, so that the script, executing from the top line, can find
# all the function definitions before we attempt to call any of them for the first time.
# To avoid rewriting the same code in several places, the first two functions are simple
# utilities.

get_return() {
  echo -e "Press return \c"
  read x
  return 0
}

get_confirm() {
  echo -e "Are you sure? \c"
  while true
  do
    read x
    case "$x" in
      y | yes | Y | Yes | YES ) 
        # shell脚本中0代表的是true
        return 0;;
      n | no  | N | No  | NO ) 
        echo 
        echo "Cancelled"
        return 1;;
      *) echo "Please enter yes or no" ;;
    esac
  done
}

# Here, we come to the main menu function, set_menu_choice.
# The contents of the menu vary dynamically, with extra options being added if a CD entry
# has been selected. Note that echo -e may not be portable to some shells.

set_menu_choice() {
  clear
  echo "Options :-"
  echo
  echo "   a) Add new CD"
  echo "   f) Find CD"
  echo "   c) Count the CDs and tracks in the catalog"
  if [ "$cdcatnum" != "" ]; then
    echo "   l) List tracks on $cdtitle"
    echo "   r) Remove $cdtitle"
    echo "   u) Update track information for $cdtitle"
  fi
  echo "   q) Quit"
  echo
  echo -e "Please enter choice then press return \c"
  read menu_choice
  return
}

# Two more very short functions, insert_title and insert_track for adding to the database files.
# Though some people hate one-liners like these, they help make other functions clearer
# They are followed by the larger add_record_track function that uses them.
# This function uses pattern matching to ensure no commas are entered (since we're using commas
# as a field separator), and also arithmetic operations to increment the current track number
# as tracks are entered.

insert_title() {
  #注意这里的$*代表的是传入的所有参数,参数在函数调用的时候用空格隔开因为默认IFS=“ ”,此时的$*为  A B，而$1为A
  echo $* >> $title_file
  return
}

insert_track() {
  echo $* >> $tracks_file
  return
}

add_record_tracks() {
  echo "Enter track information for this CD"
  echo "When no more tracks enter q"
  cdtrack=1
  cdttitle=""
  while [ "$cdttitle" != "q" ]
  do
      echo -e "Track $cdtrack, track title? \c"
      read tmp
      cdttitle=${tmp%%,*}
      if [ "$tmp" != "$cdttitle" ]; then
        echo "Sorry, no commas allowed"
        continue
      fi
      #-n代表字符串不为空则为true
      if [ -n "$cdttitle" ] ; then
        if [ "$cdttitle" != "q" ]; then
          insert_track $cdcatnum,$cdtrack,$cdttitle
        fi
      else
        cdtrack=$((cdtrack-1))
      fi
    cdtrack=$((cdtrack+1))
  done
}

# The add_records function allows entry of the main CD information for a new CD.

add_records() {
  # Prompt for the initial information

  echo -e "Enter catalog name \c"
  read tmp
  #这一步对主键进行判断
  myTmp=$(grep "^${tmp}," $title_file)
  if [ -n "$myTmp" ]; then
    echo
    echo "fuck,you can't type the same catalog"
    echo
    get_return
    return
  fi

  cdcatnum=${tmp%%,*}

  echo -e "Enter title \c"
  read tmp
  cdtitle=${tmp%%,*}

  echo -e "Enter type \c"
  read tmp
  cdtype=${tmp%%,*}

  echo -e "Enter artist/composer \c"
  read tmp
  cdac=${tmp%%,*}

  # Check that they want to enter the information

  echo About to add new entry
  echo "$cdcatnum $cdtitle $cdtype $cdac"

  # If confirmed then append it to the titles file

  if get_confirm ; then
    insert_title $cdcatnum,$cdtitle,$cdtype,$cdac
    add_record_tracks
  else
    remove_records
  fi 

  return
}

# The find_cd function searches for the catalog name text in the CD title file, using the
# grep command. We need to know how many times the string was found, but grep only returns
# a value telling us if it matched zero times or many. To get around this, we store the
# output in a file, which will have one line per match, then count the lines in the file.
# The word count command, wc, has whitespace in its output, separating the number of lines,
# words and characters in the file. We use the $(wc -l $temp_file) notation to extract the
# first parameter from the output to set the linesfound variable. If we wanted another,
# later parameter we would use the set command to set the shell's parameter variables to
# the command output.
# We change the IFS (Internal Field Separator) to a , (comma), so we can separate the
# comma-delimited fields. An alternative command is cut.

find_cd() {
  if [ "$1" = "n" ]; then
    asklist=n
  else
    asklist=y
  fi
  cdcatnum=""
  echo -e "Enter a string to search for in the CD titles \c"
  read searchstr
  if [ "$searchstr" = "" ]; then
    return 0
  fi
  #通过字符串（也就是cd唱片中一个数据项的一个关键词去找文件title_file中匹配的项然后重定向到一个temp_file）
  grep "$searchstr" $title_file > $temp_file
  # wc命令的-l选项就是显示行数 
  set $(wc -l $temp_file)
  #这里的$1代表的就是上面set的结果
  linesfound=$1

  case "$linesfound" in
  0)    echo "Sorry, nothing found"
        get_return
        return 0
        ;;
  1)    ;;
  2)    echo "Sorry, not unique."
        echo "Found the following"
        cat $temp_file
        get_return
        return 0
  esac
  #用,作为分隔，因为temp_file中的每一项都是利用,分割
  IFS=","
  read cdcatnum cdtitle cdtype cdac < $temp_file
  #恢复为空格
  IFS=" "

  if [ -z "$cdcatnum" ]; then
    #无法提取目录字段
    echo "Sorry, could not extract catalog field from $temp_file"
    get_return 
    return 0
  fi

  echo
  echo Catalog number: $cdcatnum
  echo Title: $cdtitle
  echo Type: $cdtype
  echo Artist/Composer: $cdac
  echo
  get_return

  if [ "$asklist" = "y" ]; then
    echo -e "View tracks for this CD? \c"
      read x
    if [ "$x" = "y" ]; then
      echo
      list_tracks
      echo
    fi
  fi
  return 1
}

# update_cd allows us to re-enter information for a CD. Notice that we search (grep)
# for lines that start (^) with the $cdcatnum followed by a ,, and that we need to wrap
# the expansion of $cdcatnum in {} so we can search for a , with no whitespace between
# it and the catalogue number. This function also uses {} to enclose multiple statements
# to be executed if get_confirm returns true.

update_cd() {
  if [ -z "$cdcatnum" ]; then
    echo "You must select a CD first"
    find_cd n
  fi
  if [ -n "$cdcatnum" ]; then
    echo "Current tracks are :-"
    list_tracks
    echo
    echo "This will re-enter the tracks for $cdtitle"
    get_confirm && {
      #-v表示的是对匹配模式取反，也就是不是这个匹配模式的行（这样就能删去cdcatnum对应的行）
      grep -v "^${cdcatnum}," $tracks_file > $temp_file
      mv $temp_file $tracks_file
      echo
      add_record_tracks
    }
  fi
  return
}

# count_cds gives us a quick count of the contents of our database.

count_cds() {
  set $(wc -l $title_file)
  num_titles=$1
  set $(wc -l $tracks_file)
  num_tracks=$1
  echo found $num_titles CDs, with a total of $num_tracks tracks
  get_return
  return
}

# remove_records strips entries from the database files, using grep -v to remove all
# matching strings. Notice we must use a temporary file.
# If we tried to do this,
# grep -v "^$cdcatnum" > $title_file
# the $title_file would be set to empty by the > output redirection before the grep
# had chance to execute, so grep would read from an empty file.

remove_records() {
  # -z代表字符串为null（空字符串）为真
  if [ -z "$cdcatnum" ]; then
    echo You must select a CD first
    find_cd n
  fi
  if [ -n "$cdcatnum" ]; then
    echo "You are about to delete $cdtitle"
    get_confirm && {
      grep -v "^${cdcatnum}," $title_file > $temp_file
      mv $temp_file $title_file
      grep -v "^${cdcatnum}," $tracks_file > $temp_file
      mv $temp_file $tracks_file
      cdcatnum=""
      echo Entry removed
    }
    get_return
  fi
  return
}

# List_tracks again uses grep to extract the lines we want, cut to access the fields
# we want and then more to provide a paginated output. If you consider how many lines
# of C code it would take to re-implement these 20-odd lines of code, you'll appreciate
# how powerful a tool the shell can be.

list_tracks() {
  if [ "$cdcatnum" = "" ]; then
    echo no CD selected yet
    return
  else
    #这里引号中的是一个正则表达式^代表一行的开头,${cdcatnum},表示的是一个变量扩展，可以看书中的p59的${i}_tmp
    grep "^${cdcatnum}," $tracks_file > $temp_file
    num_tracks=$(wc -l $temp_file)
    if [ "$num_tracks" = "0" ]; then
      echo no tracks found for $cdtitle
    else { 
      echo
      echo "$cdtitle :-"
      echo 
      #-f和-d连用，指定哪一片区域-d说明分割符为,-f说明第二个以后的区域
      cut -f 2- -d , $temp_file 
      echo 
      #注意这里是和管道命令一起执行的,前面的作为后面的输入，当PAGER为空，那么就用more来显示内容
    } | ${PAGER:-more}
    fi
  fi
  get_return
  return
}

# 主程序开始

rm -f $temp_file
if [ ! -f $title_file ]; then
  touch $title_file
fi
if [ ! -f $tracks_file ]; then
  touch $tracks_file
fi

# Now the application proper

clear
echo
echo
echo "Mini CD manager"
sleep 1

quit=n
while [ "$quit" != "y" ];
do
  set_menu_choice
  case "$menu_choice" in
    a) add_records;;
    r) remove_records;;
    f) find_cd y;;
    u) update_cd;;
    c) count_cds;;
    l) list_tracks;;
    b) 
      echo
      more $title_file
      echo
      get_return;;
    q | Q ) quit=y;;
    *) echo "Sorry, choice not recognized";;
  esac
done

# Tidy up and leave

rm -f $temp_file
echo "Finished"

exit 0
```

## 3.用户交互以及重定向输出不想让用户看见的小demo（利用termios结构以及/dev/tty文件实现）

```c
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <termios.h>


char *menu[] = {
    "a - add new record",
    "d - delete record",
    "q - quit",
    NULL,
};

int getchoice(char *greet, char *choice[],FILE *in,FILE *out);

int main()
{
    int choice = 0;
    /* 这里定义两个文件流，与标准输入输出区分开来 */
    FILE *input;
    FILE *output;
    struct termios initial_settings,new_settings;
    /* 函数isatty()通过检测文件描述符(通过fileno(FILE *stream)转换)是否和终端关联,从而判断检测标准输出是否重定向了 */
    if(!isatty(fileno(stdout))){
        fprintf(stderr,"You are not a terminal, OK.\n");
    }
    /* 两个不同的文件输入输出流 */
    input = fopen("/dev/tty","r");
    output = fopen("/dev/tty","w");
    if(!input || !output){
        fprintf(stderr,"Unable to open /dev/tty\n");
        exit(1);
    }
    tcgetattr(fileno(input),&initial_settings);
    new_settings=initial_settings;
    /* 关闭标准输入处理 */
    new_settings.c_lflag &= ~ICANON;
    /* 关闭回显 */
    new_settings.c_lflag &= ~ECHO;
    /* 由于前面设置了非标准模式，所以，可以设置这里的MIN=1，TIME=0，含义是:read调用将一直等待,直到有MIN个字符可以读取时才返回,返回值是读取的字符数量,到达文件尾返回0 */
    new_settings.c_cc[VMIN] = 1;
    new_settings.c_cc[VTIME] = 0;
    /* 本地模式下的启用信号，这里是取消启用信号，这里就是通过清楚这个标志来禁用对像ctrl+c(终止程序)这种特殊字符的处理 */
    new_settings.c_lflag &= ~ISIG;
    if(tcsetattr(fileno(input),TCSANOW,&new_settings) != 0){
        fprintf(stderr,"Could not set attributes\n");
    }
    do
    {
        choice = getchoice("Please select an action",menu,input,output);
        /* 会输出到标准输出 stdout，所以这里的输出会被重定向到文件file */
        printf("You have chosen: %c\n",choice);
    }while(choice != 'q');
    /* 恢复原来的终端接口状态 */
    tcsetattr(fileno(input),TCSANOW,&initial_settings);
    exit(0);
}

int getchoice(char *greet, char *choice[],FILE *in,FILE *out)
{
    int chosen = 0;
    int selected;
    char **option;
    do{
        fprintf(out,"Choice: %s\n",greet);
        option = choice;    
        while(*option){
            fprintf(out,"%s\n",*option);
            option++;
        }
        do{
            selected = fgetc(in);
            /* 多了一个'\r'字符的检测，是因为前面设置了非标准模式，此时的回车不和换行符映射了，所以需要检测回车符 */
        }while(selected == '\n' || selected == '\r');
        option = choice;
        while(*option){
            if(selected == *option[0]){
                chosen = 1;
                break;
            }
            option++;
        }
        if(!chosen){
            fprintf(out,"Incorrect choice, select again\n");
        }
    }while(!chosen);
    return selected;
}
```



