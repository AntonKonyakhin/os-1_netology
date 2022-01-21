### 1. Задача
Какой системный вызов делает команда `cd`? В прошлом ДЗ мы выяснили, что `cd` не является самостоятельной программой, это `shell builtin`, поэтому запустить `strace` непосредственно на `cd` не получится. Тем не менее, вы можете запустить `strace` на `/bin/bash -c 'cd /tmp'`. В этом случае вы увидите полный список системных вызовов, которые делает сам `bash` при старте. Вам нужно найти тот единственный, который относится именно к `cd`. Обратите внимание, что `strace` выдаёт результат своей работы в поток `stderr`, а не в `stdout`.  
Ответ:  
```
chdir("/tmp")
```
### 2. Задача  
Используя `strace` выясните, где находится база данных `file` на основании которой она делает свои догадки.  
Ответ:
```
openat(AT_FDCWD, "/usr/share/misc/magic.mgc", O_RDONLY) = 3
```

### 3. Задача  
Предположим, приложение пишет лог в текстовый файл. Этот файл оказался удален (deleted в `lsof`), однако возможности сигналом сказать приложению переоткрыть файлы или просто перезапустить приложение – нет. Так как приложение продолжает писать в удаленный файл, место на диске постепенно заканчивается. Основываясь на знаниях о перенаправлении потоков предложите способ обнуления открытого удаленного файла (чтобы освободить место на файловой системе).

Ответ:
1. мы знаем приложение и поэтому можем узнать его PID
```shell
anton@Yupiter:~/big$ lsof -nP | grep file.img
less    211             anton    3r      REG   8,48 1073741824 40750 /home/anton/big/file.img (deleted)
```
из строки выше видно PID 211 программы less и дескриптор файла 3  
Нужно теперь зайти в директорию:

```shell
anton@Yupiter:~/big$ cd /proc/211/fd
anton@Yupiter:/proc/211/fd$ ls -la
total 0
dr-x------ 2 anton anton  0 Jan 20 22:23 .
dr-xr-xr-x 9 anton anton  0 Jan 20 22:23 ..
lrwx------ 1 anton anton 64 Jan 20 22:23 0 -> /dev/pts/0
l-wx------ 1 anton anton 64 Jan 20 22:23 1 -> /dev/null
lrwx------ 1 anton anton 64 Jan 20 22:23 2 -> /dev/pts/0
lr-x------ 1 anton anton 64 Jan 20 22:23 3 -> '/home/anton/big/file.img (deleted)'
```

Просто перенаправляем пустоту в этот файл

```shell
anton@Yupiter:/proc/211/fd$ echo > 3
anton@Yupiter:/proc/211/fd$ lsof -nP | grep file.img
less    211             anton    3r      REG   8,48        1 40750 /home/anton/big/file.img (deleted)
```
Видно, что размер файла уменьшился


### 4. Задача  
Занимают ли зомби-процессы какие-то ресурсы в ОС (CPU, RAM, IO)?  
Ответ:
нет, не занимают, т.к. они завершились, но не удалились и занимают только строку в списке процессов  

### 5. Задача  
В iovisor BCC есть утилита `opensnoop` :  
```shell
root@vagrant:~# dpkg -L bpfcc-tools | grep sbin/opensnoop
/usr/sbin/opensnoop-bpfcc

```

На какие файлы вы увидели вызовы группы `open` за первую секунду работы утилиты? Воспользуйтесь пакетом `bpfcc-tools` для Ubuntu 20.04.  
Ответ:  
сначала выведем strace со значением tt и opensnoop-bpfcc со значение -d 1 в файл:
```shell
root@ubuntu-vm:/home/anton# strace -o opensnoop2.txt -tt opensnoop-bpfcc -d 1

```

Затем можно посмотереть сколько раз выполнился системный вызов openat
```shell
rootroot@ubuntu-vm:/home/anton# cat opensnoop2.txt | grep openat

получаем очень много строк вида
23:32:39.947158 openat(AT_FDCWD, "./arch/x86/include/asm/msr-index.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.948149 openat(AT_FDCWD, "./arch/x86/include/asm/unwind_hints.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.948498 openat(AT_FDCWD, "./arch/x86/include/asm/orc_types.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.950138 openat(AT_FDCWD, "./arch/x86/include/asm/spinlock_types.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.950468 openat(AT_FDCWD, "include/asm-generic/qspinlock_types.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.950841 openat(AT_FDCWD, "include/asm-generic/qrwlock_types.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.951605 openat(AT_FDCWD, "./arch/x86/include/asm/proto.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.951933 openat(AT_FDCWD, "./arch/x86/include/asm/ldt.h", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
23:32:39.951987 openat(AT_FDCWD, "arch/x86/include/generated/uapi/asm/ldt.h", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
23:32:39.952029 openat(AT_FDCWD, "arch/x86/include/generated/asm/ldt.h", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
23:32:39.952070 openat(AT_FDCWD, "./arch/x86/include/uapi/asm/ldt.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.954229 openat(AT_FDCWD, "include/uapi/linux/limits.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.954672 openat(AT_FDCWD, "include/linux/sched.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.954957 openat(AT_FDCWD, "include/uapi/linux/sched.h", O_RDONLY|O_CLOEXEC) = 3
23:32:39.955451 openat(AT_FDCWD, "./arch/x86/include/asm/current.h", O_RDONLY|O_CLOEXEC) = 3
...

```

### 6. Задача  
Какой системный вызов использует `uname -a`? Приведите цитату из `man` по этому системному вызову, где описывается  альтернативное местоположение в `/proc`, где можно узнать версию ядра и релиз ОС.

Ответ:
- выполним команду
```shell
anton@Yupiter:/proc/211/fd$ strace uname  -a
```
видим, что используется системный вызов `uname`
```
uname({sysname="Linux", nodename="ubuntu-vm", ...}) = 0
fstat(1, {st_mode=S_IFCHR|0600, st_rdev=makedev(0x88, 0), ...}) = 0
uname({sysname="Linux", nodename="ubuntu-vm", ...}) = 0
uname({sysname="Linux", nodename="ubuntu-vm", ...}) = 0

```

двлее нам нужно доставить man секцию 2  
```shell
anton@Yupiter:/proc/211/fd$ sudo apt install manpages-dev
[sudo] password for anton:
Reading package lists... Done
Building dependency tree
Reading state information... Done
manpages-dev is already the newest version (5.05-1).
manpages-dev set to manually installed.
0 upgraded, 0 newly installed, 0 to remove and 99 not upgraded.
```  
уже установлено
смотрим в эту секцию

```
man 2 uname
```
далее ищем 'proc'  
```
/proc
```  
находим такую строчку:  
```
...skipping...
       Part of the utsname information is also accessible via /proc/sys/kernel/{ostype, hostname, osrelease, version, domainname}.
```  
Далее проверяем:
```shell
anton@Yupiter:/proc/211/fd$ cat /proc/sys/kernel/ostype
Linux
anton@Yupiter:/proc/211/fd$ cat /proc/sys/kernel/osrelease
5.10.60.1-microsoft-standard-WSL2
anton@Yupiter:/proc/211/fd$ cat /proc/sys/kernel/version
#1 SMP Wed Aug 25 23:20:18 UTC 2021
```
или так, не WSL:
```shell
root@ubuntu-vm:~# cat /proc/sys/kernel/osrelease 
5.11.0-43-generic
root@ubuntu-vm:~# cat /proc/sys/kernel/ostype 
Linux
root@ubuntu-vm:~# cat /proc/sys/kernel/version 
#47~20.04.2-Ubuntu SMP Mon Dec 13 11:06:56 UTC 2021

```

### 7. Задача  
Чем отличается последовательность команд через `;` и через `&&` в bash?  
Например:
```bash
root@netology1:~# test -d /tmp/some_dir; echo Hi
Hi
root@netology1:~# test -d /tmp/some_dir && echo Hi
root@netology1:~#
```  
Ответ:  
- команда после `;` выполнится в любом случае, независимо от кода завершения первой команды  
- команда после `&&` выполнится в том случае, если код завершения предыдущей команды равен 0, т.е. команды выполнилась успешно

Есть ли смысл использовать в bash `&&`, если применить `set -e`?  
 - `&&` выполняется команда, если предыдущая вернула код 0
 - `set -e` завершает оболочку, если команды вернула отличный от 0 код выполнения  
смысла нет, т.к. при первой встрече с кодом возврата не 0, выполнение прерывается незамедлительно

### 8. Задача  
Из каких опций состоит режим bash set `-euxo pipefail` и почему его хорошо было бы использовать в сценариях?  
-v распечатывает строки ввода оболочки по мере их чтения.  
-e указывает оболочке выйти, если команда дает ненулевой статус выполнения  
-u обрабатывает неустановленные или неопределенные переменные, как ошибки во время раскрытия параметра.  
-o устанавливает опцию (например, pipefail)
-pipefail - указывает, что привыполнении команд в конвеере нужно передать ненулевой код в правую последнюю команду или нулевой, если все команды выполнились успешно. Параметр -e смотрит на последнюю команду в конвеере.  

Хорошо использовать `set -euxo pipefail` для отладки  

### 9. Задача  
Используя `-o stat` для `ps`, определите, какой наиболее часто встречающийся статус у процессов в системе. В `man ps` ознакомьтесь (`/PROCESS STATE CODES`) что значат дополнительные к основной заглавной буквы статуса процессов. Его можно не учитывать при расчете (считать S, Ss или Ssl равнозначными).  

```
root@ubuntu-vm:/home/anton# ps -o stat
STAT
S
S
S
R+

```

больше всего процессов со значение S - спящий процесс, ожэидает  

что значат дополнительные к основной заглавной буквы статуса процессов:
- < : процесс в приоритетном режиме;
- N : процесс в режиме низкого приоритета;
- L : real-time процесс, имеются страницы, заблокированные в памяти;
- s : лидер сессии
- l : является многопоточным
- '+' : в группе процессов foreground  
- 
