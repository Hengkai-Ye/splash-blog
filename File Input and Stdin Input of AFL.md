## Background
When fuzzing a program, AFL provides two ways to feed input to the target program. One is file input, the other is stdin input. For example, `./input/` is the input path, `./input/dry_run` is the input file. The content of dry run could be `hello world`. When executing `afl-fuzz` without `@@`,  `afl-fuzz` will copy `hello world` to stdin as the input. When executing `afl-fuzz` with `@@`, `afl-fuzz` will  copy `dry_run` to `./output//.cur_input` first and then use `.cur_input` as the file input to target program. In this way, even the target program modify or delete the input file, `dry_run` will not be changed. However, more details about handling file input and stdin input are unknown to me.
## Motivation Example
Several days ago, when I was testing `sqlite3` with `afl-fuzz`, I noticed some differences between execution started from `afl-fuzz` and normal execution.

- Normal execution

`input/input.db` is the file input. When running `./sqlite3 input/input.db`, `sqlite` will keep waiting for next command from stdin. 
```c
./sqlite3 input/input.db 
SQLite version 3.38.5 2022-05-06 15:25:27
Enter ".help" for usage hints.
sqlite> 
```

- Execution started from `afl-fuzz`

The running process of `./sqlite3` will be terminated quickly instead of waiting for further input from stdin.
## Analysis of afl-fuzz.c
In function `init_forkserver`
```c
    if (out_file) {

      dup2(dev_null_fd, 0);

    } else {

      dup2(out_fd, 0);
      close(out_fd);

    }
```
`out_file` is the input file to fuzz. Therefore, when we use `@@`,  `afl-fuzz.c` will call `dup2(dev_null_fd, 0)`, which will redirect `/dev/null`(EOF) to stdin. And then, we guess, `sqlite3` will exit  if it  receives something from `/dev/null`(stdin).

- Validation in GDB
```c
gdb ./sqlite3
b init_forkserver
r -i ./input/ -o ./output/ ./sqlite3 @@

Breakpoint 3, init_forkserver (argv=0x7fffffffe0c0) at afl-fuzz.c:2084
2084    EXP_ST void init_forkserver(char** argv) {
(gdb) p out_file
$6 = (u8 *) 0x555555678058 "./output//.cur_input"
(gdb) 

```
When we don't use `@@`,  `out_file` will be 0 (validated in gdb also). `afl-fuzz` will redirct `out_fd` to stdin. Let's check what `out_fd` is! 
```c
gdb ./sqlite3
b init_forkserver
r -i ./input/ -o ./output/ ./sqlite3

Breakpoint 1, init_forkserver (argv=0x7fffffffe0c0) at afl-fuzz.c:2084
2084    EXP_ST void init_forkserver(char** argv) {
(gdb) p out_file
$1 = (u8 *) 0x0
(gdb) 
```
```c
(gdb) p out_fd
$2 = 7
(gdb) info proc
process 1225282
(gdb) 
    
ls -l /proc/1225282/fd/7
lrwx------ 1 hfy5130 domain users 64 Jun  1 21:03 /proc/1225282/fd/7 -> /home/AFL/output/.cur_input
```
We can see that `afl-fuzz` will use the content in `.cur_input` as stdin!



