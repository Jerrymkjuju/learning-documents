## 缩略语

cli command line interface

tty reletypewriter 电传打字机



## 常用命令和备忘

#### 进程

ps -l

S ：进程的状态（O代表正在运行；S代表在休眠；R代表可运行，正等待运行；Z代表僵化，进程已结束但父进程已不存在；T代表停止）。



#### top

通常，如果系统的负载值超过了2，就说明系统比较繁忙了。

#### 子shell

使用（）包住的命令会在子shell中执行

echo$BASH_SUBSHELL 返回 0 则说明没有在子shell中执行

> 命令分组： 进程列表 (command1;command2) ; {command;command;} 不会创建出子shell
>
> **注意 “{” 后面需要有空格**，“}” 前可以不需要空格 但是最后一条命令需要结束后也需要加“;”  
>
> 在shell脚本中，经常使用子shell进行多进程处理。但是采用子shell的成本不菲，会明显拖慢处理速度。在交互式的CLI shell会话中，子shell同样存在问题。它并非真正的多进程处理，因为终端控制着子shell的I/O。