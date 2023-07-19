---
title: "BUAA 2023 OS Lab6 Challenge"
description: 我的2023OS课程的lab6挑战性任务的实验报告
date: 2023-07-19T17:42:39+08:00
image: mos.jpg 
math: 
license: 
hidden: false
comments: true
categories:
    - OS
---

# Lab6-Challenge实验报告


## 实现思路

### 添加命令

1. 在`user`目录下，添加`xxx.c`
2. 在`user\include.mk`的`USERAPPS`中，添加`xxx.b`

### 一行多命令

在`parsecmd`函数中的`switch-case`语句中直接再添加`case ';'`，直接`fork`，子进程直接执行`；`左边的命令，父进程等待子进程运行命令结束后再运行`；`右边的命令

```c
case ';':
	fktmp = fork();
	if (fktmp) {
		wait(fktmp);
		return parsecmd(argv, rightpipe);
	} else {
		return argc;
	}

	break;
```

### 后台任务

类似于一行多命令的实现，直接在`switch-case`中添加`caes '&'`，直接`fork`，子进程直接执行`&`左边的命令，与实现一行多命令不一样的是，此时父进程不需要等待子进程执行完毕，直接执行右边的命令即可

```c
case '&':
	fktmp = fork();
	if (fktmp) {
		return parsecmd(argv, rightpipe);
	} else {
		return argc;
	}

	break;
```

需要注意的是，`readline`中调用`read`从控制台读入字符，当没有字符输入时，进程将阻塞在内核态，无法进行调度，因此为了实现后台任务，需要改变进程阻塞的位置，使其在用户态阻塞，修改`sys_cgetc`和`cons_read`即可

```c
int sys_cgetc(void) {
	int ch;
	// while ((ch = scancharc()) == 0) {
	// }
	ch = scancharc();
	return ch;
}
```

```c
while ((c = syscall_cgetc()) == 0) {
		syscall_yield();
	}
```

### 引号支持

为了实现引号支持，需要修改token的解析方式，即修改`_gettoken`函数，在函数中，当遇到左双引号时，解析出双引号内的整个字符串，并返回`w`，将其视为普通字符串而不解析为命令。

```c
if (*s == '\"') {
		*p1 = ++s;
		while (*s && *s != '\"') {
			s++;
		}

		*s = 0;
		*p2 = s;
		return 'w';
	}
```

### 键入命令时任意位置的修改

首先要明确键盘上的方向键也对应一个字符，当按下上下左右任意按键时，代码中的`readline`函数也能读取到对应字符，四个按键的字符映射如下：

- `UP:   \033[A`
- `DOWN:  \033[B`
- `Left:  \033[D`
- `Right: \033[C`

我们需要修改`readline`函数，使之能够处理相应按键的触发，代码的主要逻辑是维护当前光标的位置和`buf`字符串，每次输入普通字符或者输入退格符时，保存光标之后的字符串，依次输出光标前的字符串、输入字符或退格符、光标后的字符串。同时要注意光标左右移动的边界，在此我通过维护`buf`字符串的长度来实现。

```c
void readline(char *buf, u_int n) {
	int r;
	int len = 0;	       // buf长度
	int i = 0;	       // 光标位置
	int hisindex = hislen; // 历史命令位置
	char temp;
	char histmp[128];

	while (i < n) {
		if ((r = read(0, &temp, 1)) != 1) {
			if (r < 0) {
				debugf("read error: %d\n", r);
			}
			exit();
		}

		if (temp == '\b' || temp == 0x7f) {
			if (i > 0) {
				if (i == len) {
					buf[--i] = 0;
					MOVELEFT(1);
					printf(" ");
					MOVELEFT(1);
				} else {
					for (int j = i - 1; j < len - 1; j++) {
						buf[j] = buf[j + 1];
					}
					buf[len - 1] = 0;
					MOVELEFT(i--); // 往前挪i个位置到最左
					printf("%s ", buf);
					MOVELEFT(len - i); // 复位
				}
				len -= 1;
			}
		} else if (temp == '\033') { // 方向键
			switch (getDir()) {
			case 1: // up
				MOVEDOWN(1);
				if (hisindex > 0) {
					hisindex--;
					getHis(hisindex, histmp);
					strcpy(buf, histmp);
					flushLine(len, i);
					printf("%s", buf);
					len = strlen(buf);
					i = len;
				}

				break;
			case 2: // down
				if (hisindex < hislen) {
					hisindex++;
					getHis(hisindex, histmp);
					strcpy(buf, histmp);
					flushLine(len, i);
					printf("%s", buf);
					len = strlen(buf);
					i = len;
				}
				break;
			case 3: // left
				if (i > 0) {
					i -= 1;
				} else {
					MOVERIGHT(1); // 抵消
				}
				break;
			case 4: // right
				if (i < len) {
					i += 1;
				} else {
					MOVELEFT(1); // 抵消
				}
				break;
			default:
				break;
			}
		} else if (temp == '\r' || temp == '\n') {

			buf[len] = 0;
			return;
		} else { // 普通字符
			if (i == len) {
				buf[i++] = temp;
			} else { // i < len
				for (int j = len; j > i; j--) {
					buf[j] = buf[j - 1];
				}
				buf[i] = temp;
				buf[len + 1] = 0;
				MOVELEFT(++i);
				printf("%s", buf);
				MOVELEFT(len - i + 1);
			}
			len += 1;
		}

		if (len >= n) {
			break;
		}
	}
	debugf("line too long\n");
	while ((r = read(0, buf, 1)) == 1 && buf[0] != '\r' && buf[0] != '\n') {
		;
	}
	buf[0] = 0;
}
```

### 程序名称中 `.b` 的省略

这个功能实现起来比较简单，当输入的程序路径无法打开时，在后面追加`.b`再次尝试打开即可

```c
	int fd;
	char progTmp[MAXPATHLEN];
	char after[] = ".b";
	if ((fd = open(prog, O_RDONLY)) < 0) {
		strcpy(progTmp, prog);
		int len = strlen(prog);
		strcpy(progTmp + len, after);
		if ((fd = open(progTmp, O_RDONLY)) < 0) {
			return fd;
		}
	}
```

### tree命令

`tree [-adf] [directory...]`

当没有directory参数时，将根目录作为参数。

实现了三种参数形式：

- `-a` 显示所有的文件和目录，默认为 `-a`；
- `-d` 显示所有的目录；
- `-f` 显示文件和目录的完整路径。

在命令执行的过程中，也记录了文件和目录的数目，并在末尾输出。

实现方式也不难，采用深度优先搜索dfs即可

- 处理目录的函数

```c
void treedir(char *path, char *name, int depth);
```

- 处理普通文件的函数

```c
void treeReg(char *path, char *name, int depth);
```

其中depth表示目录深度

```c
treedir(path, st.st_name, 0);//从此进入dfs
```

当遍历到一个目录时，进行递归，深度加一

```c
while ((n = readn(fd, &f, sizeof(f))) == sizeof(f)) {
		if (f.f_name[0]) {
			if (f.f_type == FTYPE_REG) {
				treeReg(newpath, f.f_name, depth + 1);
			} else if (f.f_type == FTYPE_DIR) {
				treedir(newpath, f.f_name, depth + 1);
			}
		}
	}
```

### mkdir命令&touch命令

`mkdir dirname`

`touch filename`

在MOS的文件系统中已经有了创建文件的功能，我们只需要进行一定的封装，通过IPC实现用户进程和文件系统服务进程的交互。在`fs/fs.c`中已有函数`file_create`，模仿在`file.c`中的其他用户接口实现`create`用户接口

- 在`user\include\fsreq.h`中添加IPC时用到的结构体

```c
#define FSREQ_CREATE 8
struct Fsreq_create {
	char req_path[MAXPATHLEN];
	int f_type;
};
```

- 在`user\include\lib.h`中声明以下函数

```c
int fsipc_create(const char *, int);
int create(const char *path, int f_type);
```

- 在`fs\serv.h`中声明以下函数

```c
int file_create(char *path, struct File **file);
```

- 实现用户库函数，其中调用`fsipc_create`

```c
int create(const char *path, int f_type) {
	return fsipc_create(path, f_type);
}
```

- 实现`fsipc_create`函数，通过IPC让文件系统完成后续文件的创建

```c
int fsipc_create(const char *path, int f_type) {
	int len = strlen(path);
	if (len == 0 || len > MAXPATHLEN) {
		return -E_BAD_PATH;
	}

	struct Fsreq_create *req = (struct Fsreq_create *)fsipcbuf;
	req->f_type = f_type;
	strcpy(req->req_path, path);

	return fsipc(FSREQ_CREATE, req, 0, 0);
}
```

- 实现`serve_create`函数，并在`switch-case`中添加相应的`case`

```c
void serve_create(u_int envid, struct Fsreq_create *rq) {
	int r;
	struct File *f;
	if ((r = file_create(rq->req_path, &f)) < 0) {
		ipc_send(envid, r, 0, 0);
		return;
	}

	f->f_type = rq->f_type;
	ipc_send(envid, 0, 0, 0);
}

case FSREQ_CREATE:
	serve_create(whom, (struct Fsreq_create *)REQVA);
	break;
```

以上完成之后只需新建`mkdir.c`、`touch.c`，在其中调用创建文件的用户接口，传入文件类型即可

```c
if ((fd = create(argv[i], FTYPE_DIR)) < 0) {
	user_panic("error create directory %s: %d\n", argv[i], fd);
}
```

```c
if ((fd = create(argv[i], FTYPE_REG)) < 0) {
	user_panic("error create file %s: %d\n", argv[i], fd);
}
```

修改重定向输出

```c
case '>':
	...
    if ((r = open(t, O_WRONLY)) < 0) {
		if (create(t, FTYPE_REG) < 0) {
			user_panic("> open failed");
		}
		r = open(t, O_WRONLY);
	} 
    ...
```

### 历史命令功能

- 初始化历史命令记录文件

```c
void his_init() {
	int fd;
	if ((fd = open("/.history", O_RDONLY)) >= 0) {
		close(fd);
		return;
	}

	if (create("/.history", FTYPE_REG) < 0) {
		user_panic("create .history failed");
	}
}
```

- 在每次读取到一行命令后就进行保存

```c
readline(buf, sizeof buf);
saveCmd(buf);
```

- 维护历史命令的条数和每条历史命令在history文件中的偏移

```c
int hislen;
int his_offsetTb[MAXHISNUM];
```

-  由于MOS文件系统的文件打开模式不包括append，对history文件的追加写很不方便，因此先实现`O_APPEND`文件打开模式，在`file.c`的`open`函数中添加以下内容，将文件描述符中的偏移设置为文件的大小

```c
if (mode & O_APPEND) {
		fd->fd_offset = ffd->f_file.f_size;
}
```

```c
void saveCmd(char *cmd) {
	int fd;
	int r;
	if ((fd = open("/.history", O_WRONLY | O_APPEND)) < 0) {
		user_panic("open .history failed");
	}
	if ((r = write(fd, cmd, strlen(cmd))) != strlen(cmd)) {
		user_panic("write error .history: %d\n", r);
	}
	if ((r = write(fd, "\n", 1)) != 1) {
		user_panic("write error .history: %d\n", r);
	}
	his_offsetTb[hislen++] = strlen(cmd) + 1 + (hislen > 0 ? his_offsetTb[hislen - 1] : 0);
	close(fd);
}
```

- 编写`getHis`函数，实现历史命令的随机访问，使用到了`seek`函数和`readn`函数

```c
void getHis(int index, char *cmd) {
	int fd;
	int r;
	if ((fd = open("/.history", O_RDONLY)) < 0) {
		user_panic("open .history failed");
	}
	if (index >= hislen) {
		*cmd = 0;
		return;
	}
	int offset = (index > 0 ? his_offsetTb[index - 1] : 0);
	int len = (index > 0 ? his_offsetTb[index] - his_offsetTb[index - 1] : his_offsetTb[index]);
	if ((r = seek(fd, offset)) < 0) {
		user_panic("seek failed");
	}
	if ((r = readn(fd, cmd, len)) != len) {
		user_panic("read history failed");
	}
	close(fd);
	cmd[len - 1] = 0;
}
```

通过编写以上内容，已经具备实现历史命令的基本条件，接下来需要修改`readline`中处理键盘上下键的地方，我通过维护当前历史命令的下标`hisindex`来实现上下键历史命令的切换，每次`readline`时，初始化`hisindex`为当前历史命令的总条数`hislen`。

形式化步骤如下：

- 维护光标的位置
- 根据当前历史命令下标获取历史命令
- 刷新控制台并写入历史命令

```c
int hisindex = hislen; // 历史命令位置
char histmp[128];
case 1: // up
	MOVEDOWN(1);
	if (hisindex > 0) {
		hisindex--;
		getHis(hisindex, histmp);
		strcpy(buf, histmp);
		flushLine(len, i);
		printf("%s", buf);
		len = strlen(buf);
		i = len;
	}

	break;
case 2: // down
	if (hisindex < hislen) {
		hisindex++;
		getHis(hisindex, histmp);
		strcpy(buf, histmp);
		flushLine(len, i);
		printf("%s", buf);
		len = strlen(buf);
		i = len;
	}
	break;
```

最后实现`history`命令，编写`history.c`文件，打开`.history`文件，将其中的内容格式化输出即可，用到`open`和`read`两个函数

```c
if ((r = open("/.history", O_RDONLY)) < 0) {
	user_panic("open history failed");
}
f = r;

while ((r = read(f, &buf, 1)) == 1) {
	if (newline) {
		printf("%-5d  ", linecnt++);
		newline = 0;
	}
	printf("%c", buf);

	if (buf == '\n') {
		newline = 1;
	}
}
```

### shell环境变量

我的做法是通过系统调用，在内核中维护环境变量，实现对环境变量的增删改查

首先定义保存环境变量相关信息的结构体：

```c
struct shell_var {
	char name[MAXVARNAMELEN];
	char value[MAXVARVALUELEN];
	int shellid;
	int share;//1表示为环境变量
	int rdonly;//1表示为只读
	int valid;//1表示该变量是否有效
}
```

再实现系统调用，以下是四个用户态的系统调用部分

- `syscall_shellid_alloc`用户给每个shell进程分配一个id用于标识环境变量是被哪个环境变量创建的
- `syscall_declare_shell_var`用于声明一个环境变量或者修改环境变量
- `syscall_unset_shell_var`用户删除一个环境变量
- `syscall_get_shell_var`用于获得某个环境变量的值

```c
int syscall_declare_shell_var(int shellid, char *name, char *value, int share, int rdonly);
int syscall_unset_shell_var(int shellid, char *name);
int syscall_get_shell_var(int shellid, char *name, char *value);
int syscall_shellid_alloc();
```

需要注意的时，在实现这四个系统调用时，需要判断内核中保存的环境变量对当前shell进程是否可见，因此我定义了一个`isvisible`函数，用于判断某个变量是否可见。同时为了实现简单，规定对于所有shell进程，其可见的变量不会出现重名的情况，即在实现`declare`的时候，优先在可见的变量中查找有无名字为参数`name`的变量

以下是内核态中的系统调用部分

```c
int sys_shellid_alloc() {
	return ++sid;
}
```

在实现`declare`和`unset`时，需要注意判断可见性和只读性

```c
int sys_declare_shell_var(int shellid, char *name, char *value, int share, int rdonly) {
	int varid = get_var(shellid, name);

	if (varid == -1) {
		varid = varnum++;
	} else if (VAR[varid].rdonly) {
		return -E_VAR_ERR;
	}

	strcpy(VAR[varid].name, name);
	strcpy(VAR[varid].value, value);
	VAR[varid].shellid = shellid;
	VAR[varid].share = share;
	VAR[varid].rdonly = rdonly;
	VAR[varid].valid = 1;
	return 0;
}
```

```c
int sys_unset_shell_var(int shellid, char *name) {
	int varid = get_var(shellid, name);

	if (varid == -1 || VAR[varid].rdonly) {
		return -E_VAR_ERR;
	}

	VAR[varid].valid = 0;
	return 0;
}
```

- 实现`get`的时候，若传入参数`name`为空，则返回所有的可见变量信息

```c
int sys_get_shell_var(int shellid, char *name, char *value) {
	char *tmp = value;
	if (name == 0) {
		for (int i = 0; i < varnum; i++) {
			if (isvisible(shellid, VAR[i]) && VAR[i].valid) {
				strcpy(tmp, "name: ");
				tmp += 6;
				strcpy(tmp, VAR[i].name);
				tmp += strlen(VAR[i].name);
				strcpy(tmp, " value: ");
				tmp += 8;
				strcpy(tmp, VAR[i].value);
				tmp += strlen(VAR[i].value);
				strcpy(tmp, " mood: ");
				tmp += 7;
				if (VAR[i].rdonly) {
					*tmp++ = 'r';
				} else {
					*tmp++ = 'w';
				}
				if (VAR[i].share) {
					*tmp++ = 'x';
				}
				*tmp++ = '\n';
			}
		}
		return 0;
	} else {
		int varid = get_var(shellid, name);

		// printk("varid:%d\n", varid);

		if (varid == -1) {
			return -E_VAR_ERR;
		}
		strcpy(value, VAR[varid].value);
		return 0;
	}
	return -E_VAR_ERR;
}
```

完成上述功能后，需要修改shell部分

- 在`sh.c`中，每次进入主函数都通过系统调用`syscall_shellid_alloc`获得当前shell的id
- 在`parsecmd`中，当遇到`$`时，解析后面的字符串为对应变量的值
- 在`runcmd`中，当命令为`delcare`或`unset`时，在参数列表`argv`末尾追加当前shell的id

#### 实现declare命令

编写`declare.c`文件，根据传入的选项和参数列表进行相应的处理

```c
ARGBEGIN {
	default:
		usage();
	case 'x':
	case 'r':
		flag[(u_char)ARGC()]++;
		break;
	}
ARGEND
```

```c
if (argc == 1) {
		char buf[4096];
		if ((r = syscall_get_shell_var(shellid, 0, buf)) < 0) {
			printf("declare wrong: %d\n", r);
		}
		printf("%s\n", buf);
	} else if (argc == 2) {
		if ((r = syscall_declare_shell_var(shellid, argv[0], "", share, rdonly)) < 0) {
			printf("declare wrong: %d\n", r);
		}
	} else if (argc == 3) {
		val = argv[1];
		if (val[0] == '=') {
			val++;
			if ((r = syscall_declare_shell_var(shellid, argv[0], val, share, rdonly)) <
			    0) {
				printf("declare wrong: %d\n", r);
			}
		} else {
			usage();
		}
	}
```

#### 实现unset命令

编写`unset.c`文件，根据传入的变量名使用系统调用即可

```c
for (i = 1; i < argc - 1; i++) {
			if ((r = syscall_unset_shell_var(shellid, argv[i])) < 0) {
				printf("environment value %s isn't declared or is readonly\n",
				       argv[i]);
			}
		}
```

## 功能测试

### 一行多命令

```c
$ echo 1;echo 12;echo 123
1
[00003805] destroying 00003805
[00003805] free env 00003805
i am killed ... 
[00003004] destroying 00003004
[00003004] free env 00003004
i am killed ... 
12
[00004805] destroying 00004805
[00004805] free env 00004805
i am killed ... 
[00004004] destroying 00004004
[00004004] free env 00004004
i am killed ... 
123
[00005004] destroying 00005004
[00005004] free env 00005004
i am killed ... 
[00002803] destroying 00002803
[00002803] free env 00002803
i am killed ... 

$ echo 123;
123
[00006805] destroying 00006805
[00006805] free env 00006805
i am killed ... 
[00006004] destroying 00006004
[00006004] free env 00006004
i am killed ... 
[00005803] destroying 00005803
[00005803] free env 00005803
i am killed ... 

$ ;echo 123
[00007804] destroying 00007804
[00007804] free env 00007804
i am killed ... 
123
[00008004] destroying 00008004
[00008004] free env 00008004
i am killed ... 
[00007003] destroying 00007003
[00007003] free env 00007003
i am killed ... 
```

### 后台任务

- 编写一个运行时间比较久的测试程序`testbg.c`

```c
#include <lib.h>

int main() {
	int i;
	debugf("a very slow code begin\n");
	for (i = 0; i <= 999999999; i++)
		;
	debugf("a very slow code end\n");
	return 0;
}
```

```c
$ testbg&echo 123
123
[00008803] destroying 00008803
[00008803] free env 00008803
i am killed ... 
[00007804] destroying 00007804
[00007804] free env 00007804
i am killed ... 

$ a very slow code begin
echo 321
321
[0000a003] destroying 0000a003
[0000a003] free env 0000a003
i am killed ... 
[00009804] destroying 00009804
[00009804] free env 00009804
i am killed ... 

$ a very slow code end
[00009005] destroying 00009005
[00009005] free env 00009005
i am killed ... 
[00008006] destroying 00008006
[00008006] free env 00008006
i am killed ... 
```

可以看到`testbg`在后台运行

### 引号支持

```c
$ echo "ls.b | cat.b"
ls.b | cat.b
[0000b805] destroying 0000b805
[0000b805] free env 0000b805
i am killed ... 
[0000b006] destroying 0000b006
[0000b006] free env 0000b006
i am killed ... 
```

### 键入命令时任意位置的修改

测试通过在键盘上依次按下`1,2,3,4,⬅,⬅,a,b,c,Backspace,➡,x,y,z,enter`，显示效果如下：

```c
$ 12ab3xyz4
spawn 12ab3xyz4: -10
[0000c006] destroying 0000c006
[0000c006] free env 0000c006
i am killed ... 
```

### 程序名称`.b`的省略

```c
$ ls
testarg.b cat.b pingpong.b testbss.b newmotd history.b testpiperace.b testpipe.b motd init.b num.b touch.b mkdir.b testfdsharing.b declare.b testbg.b ls.b echo.b sh.b tree.b unset.b halt.b testptelibrary.b .history 
[0000d005] destroying 0000d005
[0000d005] free env 0000d005
i am killed ... 
[0000c806] destroying 0000c806
[0000c806] free env 0000c806
i am killed ... 

$ echo 123
123
[0000e005] destroying 0000e005
[0000e005] free env 0000e005
i am killed ... 
[0000d806] destroying 0000d806
[0000d806] free env 0000d806
i am killed ... 
```

### tree,mkdir,touch命令

```c
$ mkdir dir1
$ touch dir1/file1
$ touch dir1/file2
$ mkdir dir1/dir11
$ touch dir1/dir11/file3
$ touch dir1/dir11/file4
$ mkdir dir1/dir12
$ touch dir1/dir12/file5    
$ touch dir1/dir12/file6
    
$ tree dir1             
dir1
|-- file1
|-- file2
|-- dir11
    |-- file3
    |-- file4
|-- dir12
    |-- file5
    |-- file6

2 directories, 6 files

$ tree -d dir1
dir1
|-- dir11
|-- dir12

2 directories
    
$ tree -f dir1
dir1
|-- dir1/file1
|-- dir1/file2
|-- dir1/dir11
    |-- dir1/dir11/file3
    |-- dir1/dir11/file4
|-- dir1/dir12
    |-- dir1/dir12/file5
    |-- dir1/dir12/file6

2 directories, 6 files
```

- 重定向

```c
$ echo test123 > newfile
[00015804] destroying 00015804
[00015804] free env 00015804
i am killed ... 
[00015003] destroying 00015003
[00015003] free env 00015003
i am killed ... 
    
$ cat newfile
test123
[00017804] destroying 00017804
[00017804] free env 00017804
i am killed ... 
[00017003] destroying 00017003
[00017003] free env 00017003
i am killed ... 
```

### 历史命令功能

输入序列`history,enter,↑,↑,↑,↓,enter`

```c
$ history
1      mkdir dir1
2      mkdir dir2
3      mkdir dir3
4      touch dir1/file1
5      touch dir1/file2
6      mkdir dir1/dir11
7      touch dir1/dir11/file3
8      touch dir1/dir11/file4
9      tree
10     tree dir1
11     mkdir dir1/dir12
12     touch dir1/dir12/file5
13     touch dir1/dir12/file6
14     tree dir1
15     tree -d dir1
16     tree -f dir1
17     tree
18     tree -f
19     tree -f dir1
20     echo test123 > newfile
21     tree
22     cat newfile
23     history
[00018804] destroying 00018804
[00018804] free env 00018804
i am killed ... 
[00018003] destroying 00018003
[00018003] free env 00018003
i am killed ... 

$ cat newfile           
test123
[00019804] destroying 00019804
[00019804] free env 00019804
i am killed ... 
[00019003] destroying 00019003
[00019003] free env 00019003
i am killed ... 
```

### shell环境变量

```c
$ declare -x var1 =1

$ declare -x var2 =2

$ declare var3 =3

$ echo $var3
3

$ declare
name: var1 value: 1 mood: wx
name: var2 value: 2 mood: wx
name: var3 value: 3 mood: w


$ sh
shellid:2

:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::
::                                                         ::
::                     MOS Shell 2023                      ::
::                                                         ::
:::::::::::::::::::::::::::::::::::::::::::::::::::::::::::::

$ declare
name: var1 value: 1 mood: wx
name: var2 value: 2 mood: wx


$ declare -r var4 =4

$ unset var1

$ declare 
name: var2 value: 2 mood: wx
name: var4 value: 4 mood: r


$ declare var2 =5
 
$ declare
name: var2 value: 5 mood: w
name: var4 value: 4 mood: r

$ declare var5 

$ declare
name: var2 value: 5 mood: w
name: var4 value: 4 mood: r
name: var5 value:  mood: w
 

$ unset var4
environment value var4 isn't declared or is readonly

$ declare var4
declare wrong: -14
```

## 困难和解决方案

### 命令运行出现的异步问题

在实现一行多命令时，需要确保命令执行顺序与输入顺序保持一致，解决方案是通过`fork`创建子进程，子进程直接运行左边的命令，

父进程等待子进程运行完毕再运行右边的命令。

### 内核态阻塞的问题

实现后台任务时，由于`read`函数的实现是在内核态进行阻塞，无法进行调度，因此解决方案是将阻塞的位置调整在用户态，使其能进行调度。

### 方向键的监听

一开始并不了解方向键在系统中的存在形式，查找相关资料后，了解到每个方向键也对应着相应的字符，因此只需要判断输入的字符是否为方向键对应的字符即可

### 解析形如`-abc`的参数的方法

起初并不了解这种参数的解析，但发现`ls`命令也有类似的参数，故阅读`ls.c`文件并查找相关资料后，学习到了这种参数的解析方法

```c
ARGBEGIN {
	default:
		usage();
	case 'd':
	case 'F':
	case 'l':
		flag[(u_char)ARGC()]++;
		break;
	}
ARGEND
```

### 文件打开模式`O_APPEND`的实现

MOS的文件系统中并没有实现追加打开方式，实现该模式，需要在打开文件时，将文件描述符里存的偏移设置为文件大小

### 历史命令的保存

仅有一个`.history`文件对存取是不方便的，因此我维护了一个偏移表，进行历史命令的存取。