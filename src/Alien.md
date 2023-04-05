# Alien

A simple operating system implemented in rust. The purpose is to explore how to use modules to build a complete os,
so the system is composed of a series of independent modules. At present, the system already supports user-mode programs and some simple functions.


## Modules
`pci` ：pci driver to detect devices on the bus

`rtc` ：rtc driver to get time

`page-table `： page table to manage virtual memory

`simple-bitmap`： a simple bitmap to manage frames

`gmanager`: a simple allocator to manage `process`/ `fd` and so on

`rvfs `: a vfs framework like linux, it can support multiple file systems

`fat32-vfs`: a disk file system, it can support fat32

`jammdb `: a key-value database, it can support `dbfs`

`dbfs2 `:  a disk file system, it is based on `jammdb`

`trace_lib `: stack trace library

`preprint ` : a simple print library

`rslab `: slab allocator like linux

`syscall-table `: A tool to automatically collect syscalls

`dbop`: Make database functions available to users as system calls

`plic`: riscv plic driver

`uart`: uart driver, it supports interrupt

Other modules are not listed here, you can find them in the cargo.toml file.

## filesystem syscall support
Now the os supports the following system calls:

```rust
pub fn sys_openat(dirfd: isize, path: usize, flag: usize, mode: usize) -> isize
pub fn sys_close(fd: usize) -> isize
pub fn sys_read(fd: usize, buf: *mut u8, len: usize) -> isize
pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize
pub fn sys_getcwd(buf: *mut u8, len: usize) -> isize 
pub fn sys_chdir(path: *const u8) -> isize
pub fn sys_mkdir(path: *const u8) -> isize
pub fn sys_list(path: *const u8) -> isize
pub fn sys_lseek(fd: usize, offset: isize, whence: usize) -> isize 
pub fn sys_fstat(fd: usize, stat: *mut u8) -> isize
pub fn sys_linkat(
    old_fd: isize,
    old_name: *const u8,
    new_fd: isize,
    new_name: *const u8,
    flag: usize,
) -> isize
pub fn sys_unlinkat(fd: isize, path: *const u8, flag: usize) -> isize
pub fn sys_symlinkat(old_name: *const u8, new_fd: isize, new_name: *const u8) -> isize
pub fn sys_readlinkat(fd: isize, path: *const u8, buf: *mut u8, size: usize) -> isize
pub fn sys_fstateat(dir_fd: isize, path: *const u8, stat: *mut u8, flag: usize) -> isize 
pub fn sys_fstatfs(fd: isize, buf: *mut u8) -> isize
pub fn sys_statfs(path: *const u8, statfs: *const u8) -> isize 
pub fn sys_renameat(
    old_dirfd: isize,
    old_path: *const u8,
    new_dirfd: isize,
    new_path: *const u8,
) -> isize
pub fn sys_mkdirat(dirfd: isize, path: *const u8, flag: usize) -> isize
pub fn sys_setxattr(
    path: *const u8,
    name: *const u8,
    value: *const u8,
    size: usize,
    flag: usize,
) -> isize
pub fn sys_getxattr(path: *const u8, name: *const u8, value: *const u8, size: usize) -> isize
pub fn sys_fgetxattr(fd: usize, name: *const u8, value: *const u8, size: usize) -> isize
pub fn sys_listxattr(path: *const u8, list: *const u8, size: usize) -> isize
pub fn sys_flistxattr(fd: usize, list: *const u8, size: usize) -> isize
pub fn sys_removexattr(path: *const u8, name: *const u8) -> isize
pub fn sys_fremovexattr(fd: usize, name: *const u8) -> isize
```

There are some applications in the `app` directory to test the system calls.


## TODO

- [ ] Thread/Mutil-core
- [x] full vfs
- [x] fat32
- [x] dbfs
- [ ] Mutex
- [x] sleep task queue
- [x] uart task queue
- [ ] block driver task queue
- [x] a simple shell
- [x] memory management
- [x] process management

## Github
[Alien OS](https://github.com/Godones/Alien)