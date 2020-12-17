## Shell命令

## 一、文件路径

### 1. ls 基本查看

查看文件夹内的所有的内容，默认情况下不能看到隐藏文件

| 序号 | 选项 | 作用                                                         |
| ---- | ---- | ------------------------------------------------------------ |
| 1    | -a   | 查看文件夹内所有的内容，包括隐藏的文件，隐藏文件时文件名前带着"." |
| 2    | -l   | 以列表的形式列出文件的详细信息，包括文件所属的用户和组，文件的权限以及时间 |

```
ls -a
ls -a -l
ls -la
ls -al
ll
```

清屏的命令：ctrl+l



### 2. ll 详细查看

以列表的形式查看文件的内容

### 3. 访问文件路径
- cd 绝对路径
```
cd /etc/sysconfig/network-scripts/
```

- cd 相对路径
```
cd d1  #进入到当前路径下的d1文件夹内
```

- cd ..
```
cd .. # 回到上一层目录
```

- cd .
```
cd . #处在当前路径下，不会发生变化
```

- cd /
```
cd / #进入到根目录下
```

- cd ~
```
cd ~ #进入到当前用户的家，如果当前是root用户，那么家在/root，如果当前是普通用户，家在/home/普通用户文件夹
```

- cd -
```
cd - #回到改变路径之前的那一次位置
```



## 二、文件管理

### 1.创建文件

- touch file1.txt
- touch file2 file3
- touch /home/file2 file3
- touch /home/{file2,file3}
- touch /home/file{1..10}

### 2.创建目录

- mkdir dir
- mkdir /home/dir
- mkdir /home/{dir1,dir2}
- mkdir -v /home/dir3
- mkdir -p /home/d/dir4
- mkdir -pv /home/d/dir5

### 3.复制文件

cp 源文件 目标目录

- cp file2 /home/div2/

- cp -Rv /etc  /home/div2


  使用递归拷贝etc文件夹内的所有文件到目标目录

### 4.移动文件（剪切）

- mv file1 /home/dir3  移动至dir3中
- mv file1/home/dir3/file2 移动至dir3中并重命名成file2
- mv file1file2 当前路径下直接重命名成file2



### 5.删除文件

- rm -rf 目标文件

  - r: 递归删除，对于文件夹的删除来说，需要使用r选项
  - f:强制删除


```
rm -rf 目标文件/文件夹
rm -rf /   #不要用
```



- 使用通配符* 来删除文件

  - rm -rf /home/dir10/file*   

    注意：*是不包含隐藏文件的

  - rm -rf /home/dir10/.file3

    删除隐藏文件

  - rm -rf/home/dir10/*.pdf

## 三、查看文件内容

键盘上的Tab键可以自动补全

### 1.cat命令

```
cat /etc/hosts
```

cat命令不适合看长的文件，适合看短的文件。

### 2.head命令

```
head /etc/passwd  默认看文件的前10行 head -5 /etc/passwd  默认看文件的前10行
```

### 3.tail命令

```
tail/etc/passwd  默认看文件的后10行 
```

### 4.less命令

```
less /etc/passwd 可以分页显示，按q退出
```

### 5.more命令

```
more /etc/passwd 按回车往下翻，不能往上翻
```

### 6.grep命令

```
grep ‘root’/etc/passwd 条件：搜索root的行

grep ‘^root’ /etc/passwd 条件：搜索root开头的行

grep ‘bash$’ /etc/passwd 条件：搜索bash结尾的行

ps -l | grep w

```



## 四、编辑文件



### vi编辑器（vim=增强版的vi）

vim编辑器有三种模式

- 普通模式： 使用vim打开一个文本文件，即进入到普通模式
- 编辑模式： 键盘输入"i"，当输入esc键，回到普通模式
- 命令模式：键盘输入":"

| 命令 |    作用    |
| :--: | :--------: |
|  q   |    退出    |
|  w   |    保存    |
|  !   |  强制执行  |
|  wq  | 保存并退出 |
|  q!  |  强制退出  |













