### *Linux*

学习的几个阶段：

- 基本操作（文件操作、vim工具、用户管理） -- 基本掌握
- 各种配置（环境变量、网络配置、服务配置） --基本掌握
- 搭建环境（大数据组件、JavaEE、Python）    -- 学习中
- 维护服务器（计算机核心原理）



### 1. 基本操作

#### 1.1 文件操作

##### 1.1.1 基本属性

![image-20201231114832243](C:\Users\f30001582\AppData\Roaming\Typora\typora-user-images\image-20201231114832243.png)

​       首先，明确字母含义及其代表数字：r：read表示读权限，数字4表示；w：write表示写权限，数字2表示；x：excute表示执行权限，数字1表示；

​		然后，`ll`输出结果的第一字段为10个“-”组成的文件属性字段，第一个表示文件类型：文件（-表示）、文件夹（d表示，dirtectory目录的缩写）、连接文件（l表示，link软链接）；后面9个按照三个一组，表示User、Group、Other的权限，如上图 drwxr-x--- 表示此文件是文件夹，拥有者有rwx权限，同组用户有rx权限，其他用户没有权限，也可表示为750权限；

​		第二字段为该文件所占用的节点数，属于软链接，或为目录内所含子目录的个数，若为空目录，则该字段为2，因为每个目录都有一个指向它的子目录“.” 和指向它上级目录的“..”；

​		第三四列为文件的属主和属组，chown(change ownerp)修改属主或属组 / chmod(change mode)修改文件属性，其中：

- 更改属主或属组信息：
  chgrp: 更改文件属组 chgrp [-R] 属组名 文件名
  chown:更改文件属主 chowm [-R] 属主名 文件名
  	         同时更改文件属主属组 chown [-R] 属主名：属组名 文件名
- 更改文件属性信息：

```shell
  # chmod：更改文件属性 chmod [-R] u=rwx,g=rwx,o=rwx 文件名
  # 或者：
  chmod 750 test.sh
```

##### 1.1.2 解压缩文件

​	  压缩文件夹：     zip  -r   zk.zip  zk          tar -zcvf zk.tar /opt/zk
​      解压文件：         unzip   zk.zip                tar  -xvf  zk.tar



#### 1.2 vim工具

##### 1.2.1 查看文件内容

常用查看命令：  cat, tac, nl, more, less, head, tail;      通过man [命令] 可查看命令使用说明；

1）cat 是由第一行开始显示文件内容，-n 参数是带行号显示内容；tac 刚好相反，由最后一行开始显示内容；

2）nl 是带行号显示内容，跳过空行；

3）more与less都是一页页翻动； 空白键向下翻一页，回车键向下翻一行，q退出，这三个指令对man同样有效；

4）head取文件前几行，tail取文件后几行；默认取10行，-n [number] 参数表示特定行数，如tail -n 50 /tmp/test.txt；

5）tail -f -n /tmp/test.txt ，-f参数表示循环读取，适用于不断刷新的日志文件，总能显示最后十行；

##### 1.2.2 文件传输

 跨机传输：如将本机/tmp下的文件传到其他机子用户qige上：

 `scp /tmp/test.txt  qige@10.10.10.169:/tmp/` 

##### 1.2.3 写入文件

- 覆盖写入： echo "内容" > "文件名" ， 若文件不存在则新生成；
- 追加写入：  echo "内容" >> "文件名"；

##### 1.2.4 常用快捷键

- gg 移动到首行，shift+gg移动到尾行；
- 光标移动到要删除的列：dd是删除该行， 5dd是删除自该行起向下数的5行；
- i、o都是写入模式，o是向下新建行写入（常用）；
- `:wq`保存退出，`:q!`不保存退出；



#### 1.3 用户管理

##### 1.3.1 普通用户

1）/etc/group  

​		文件里存放着所有用户组信息，形如：  qige：x：2201:mike,john  ，其中，qige为用户组名称，x为隐藏的用户组密码，2201为GID号（指定新用户组的组标识号），0是超级用户root的标识号，1～99由系统保留，作为管理账号，普通用户的标识号从1000（不定）开始；mike和John为用户组其他成员；

2）/etc/passwd

 		文件里存放用户的密码属性，形如： qige：x：2201:2201::/home/qige:/bin/bash     ，每行记录对应一个用户，被：分为7个部分，具体含义为：（用户名:口令:用户标识号:组标识号:注释性描述:主目录:登录Shell）

3）/etc/shadow

​		由于/etc/passwd文件是所有用户都可读的，如果用户的密码太简单或规律比较明显的话，一台普通的计算机就能够很容易地将它破解，因此对安全性要求较高的Linux系统都把加密后的口令字分离出来，单独存放在/etc/shadow文件。 有超级用户才拥有该文件读权限，这就保证了用户密码的安全性。它由pwconv命令根据/etc/passwd中的数据自动产生；

管理用户的常用命令：

- 添加用户  adduser + [user name]； 然后设置密码；

- 用户间切换 su - [user name],  Crtl+D 退出当前用户；

##### 1.3.2 超级用户

​		sudo命令允许普通用户拥有超级用户root一样的权限，但前提是该普通用户需要被添加到/etc/group 中的sudo用户组中，在该行记录中添加该用户即可；
注：每次使用sudo时会需要用户密码，若不想每次输入密码，可在/etc/sudoers中添加
your_user_name ALL=(ALL) NOPASSWD: ALL

%admin ALL=(ALL) NOPASSWD: ALL   # 若只添加用户免密，可能会被group中的设置覆盖，因此需要将该用户所在用户组设置为免密

#### 1.4 目录操作

- 绝对路径：由根目录写起：   /home/qige
- 相对路径：相对本级目录而言的下级目录和上级目录分别为：.和..       
- 完整目录为  /home/qige/tmp , 若此时处于/home/qige ，则 cd .. 为回到/home
-  涉及操作目录的相关命令：ls, cd, pwd, mkdir, rmdir, cp, rm, mv；    通过man [命令] 可查看命令使用说明；

```shell
1）pwd -P , 如果当前目录为软链接的话，P参数能够显示出其指向的真正目录；
2）多层次创建目录mkdir -P /tmp/test1/test2 ， 同层次创建多个目录 mkdir tmp/test1 tmp/test2 ；
3）rmdir 只能删除空目录，  非空目录需要rm删除；
4）切换到当前用户家目录： cd，切换到普通用户（qige）家目录： cd  -> cd ~qige (波浪线扩展)；
5) 查找目录：find / -name [目录名称] -type d
6) 统计目录下所有目录个数 ls  -lR  | grep "^d" | wc -l
7) 创建软链接：创建软链接 ln -s [源文件] [链接名称]
	如：ln -s /home/qige/photo.txt /home/qishushu/introduction
```



### 2. 环境配置

#### 2.1 shell操作

1.1 变量名：使用定义过的变量，需要在变量名前加`$`符号，最好将变量写在`{}`里，如

```shell
my_name = "uncle_qi"
echo ${my_name}
```

1.2 字符串：尽量使用双引号，因为里面可以有变量和转义字符`\`

```shell
# 获取字符串长度
string = "abcd"
echo ${#string}  输出 4
```

1.3 数组：用括号表示数组，元素用空格分隔

```shell
array_name = (value0 value1 value2)
echo ${array_name[@]}  # 获取数组所有元素

```

