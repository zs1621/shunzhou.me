---
title: 案例分析-为什么不能ping
date: 2024-03-14 09:35:11
tags: [案例分析,dns,入门]
---

### 问题重现场景

```
# vim /etc/hosts  将下面两行贴入 /etc/hosts
127.0.0.1　 test.unknow.host
127.0.0.1   test.localhost
```

- 问什么ping不同test.unknow.host

```
ping -c 1 test.unknow.host    notok
ping -c 1 test.localhost      ok
```

### 排查

#### 初步排查 - 对比strace 记录

```
strace ping -c 1 test.localhost
```

> connect(5, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = 0
> sendto(5, "\2\0\0\0\r\0\0\0\6\0\0\0hosts\0", 18, MSG_NOSIGNAL, NULL, 0) = 18
> poll([{fd=5, events=POLLIN|POLLERR|POLLHUP}], 1, 5000) = 1 ([{fd=5, revents=POLLIN|POLLHUP}])
> recvmsg(5, {msg_name=NULL, msg_namelen=0, msg_iov=[{iov_base="hosts\0", iov_len=6}, {iov_base="\310O\3\0\0\0\0\0", iov_len=8}], msg_iovlen=2, msg_control=[{cmsg_len=20, cmsg_level=SOL_SOCKET, cmsg_type=SCM_RIGHTS, cmsg_data=[6]}], msg_controllen=20, msg_flags=MSG_CMSG_CLOEXEC}, MSG_CMSG_CLOEXEC) = 14
> mmap(NULL, 217032, PROT_READ, MAP_SHARED, 6, 0) = 0x7f3fe5371000
> close(6)                                = 0
> close(5)                                = 0
> **socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 5**
> **connect(5, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = 0**
> **sendto(5, "\2\0\0\0\16\0\0\0\17\0\0\0test.localhost\0", 27, MSG_NOSIGNAL, NULL, 0) = 27**
> **poll([{fd=5, events=POLLIN|POLLERR|POLLHUP}], 1, 5000) = 1 ([{fd=5, revents=POLLIN|POLLHUP}])**
> **read(5, "\2\0\0\0\1\0\0\0\1\0\0\0\4\0\0\0\17\0\0\0\0\0\0\0", 24) = 24**
> **read(5, "\177\0\0\1\2test.localhost\0", 20) = 20**
> close(5)                                = 0
> openat(AT_FDCWD, "/usr/lib64/charset.alias", O_RDONLY|O_NOFOLLOW) = -1 ENOENT (No such file or directory)
> socket(AF_INET, SOCK_DGRAM, IPPROTO_IP) = 5
> connect(5, {sa_family=AF_INET, sin_port=htons(1025), sin_addr=inet_addr("127.0.0.1")}, 16) = 0
> getsockname(5, {sa_family=AF_INET, sin_port=htons(41236), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
> close(5)                                = 0
> setsockopt(3, SOL_IP, IP_RECVERR, [1], 4) = 0
> setsockopt(3, SOL_IP, IP_RECVTTL, [1], 4) = 0
> setsockopt(3, SOL_IP, IP_RETOPTS, [1], 4) = 0
> setsockopt(3, SOL_SOCKET, SO_SNDBUF, [324], 4) = 0
> setsockopt(3, SOL_SOCKET, SO_RCVBUF, [65536], 4) = 0
> getsockopt(3, SOL_SOCKET, SO_RCVBUF, [131072], [4]) = 0
> fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
> write(1, "PING test.localhost (127.0.0.1) "..., 54PING test.localhost (127.0.0.1) 56(84) bytes of data.
> ) = 54

```
strace ping -c 1 test.unknow.host
```

> socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 5
> connect(5, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = 0
> sendto(5, "\2\0\0\0\r\0\0\0\6\0\0\0hosts\0", 18, MSG_NOSIGNAL, NULL, 0) = 18
> poll([{fd=5, events=POLLIN|POLLERR|POLLHUP}], 1, 5000) = 1 ([{fd=5, revents=POLLIN|POLLHUP}])
> recvmsg(5, {msg_name=NULL, msg_namelen=0, msg_iov=[{iov_base="hosts\0", iov_len=6}, {iov_base="\310O\3\0\0\0\0\0", iov_len=8}], msg_iovlen=2, msg_control=[{cmsg_len=20, cmsg_level=SOL_SOCKET, cmsg_type=SCM_RIGHTS, cmsg_data=[6]}], msg_controllen=20, msg_flags=MSG_CMSG_CLOEXEC}, MSG_CMSG_CLOEXEC) = 14
> mmap(NULL, 217032, PROT_READ, MAP_SHARED, 6, 0) = 0x7f0ace9ce000
> close(6)                                = 0
> close(5)                                = 0
> **socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 5**
> **connect(5, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = 0**
> **sendto(5, "\2\0\0\0\16\0\0\0\21\0\0\0test.unknow.host\0", 29, MSG_NOSIGNAL, NULL, 0) = 29**
> **poll([{fd=5, events=POLLIN|POLLERR|POLLHUP}], 1, 5000) = 1 ([{fd=5, revents=POLLIN}])**
> **read(5, "\2\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0", 24) = 24**
> close(5)                                = 0
> openat(AT_FDCWD, "/usr/share/locale/locale.alias", O_RDONLY|O_CLOEXEC) = 5
> fstat(5, {st_mode=S_IFREG|0644, st_size=2998, ...}) = 0
> read(5, "# Locale name alias data base.\n#"..., 4096) = 2998
> read(5, "", 4096)                       = 0
> close(5)                                = 0
> openat(AT_FDCWD, "/usr/share/locale/en_US.UTF-8/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
> openat(AT_FDCWD, "/usr/share/locale/en_US.utf8/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
> openat(AT_FDCWD, "/usr/share/locale/en_US/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
> openat(AT_FDCWD, "/usr/share/locale/en.UTF-8/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
> openat(AT_FDCWD, "/usr/share/locale/en.utf8/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
> openat(AT_FDCWD, "/usr/share/locale/en/LC_MESSAGES/libc.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
> write(2, "ping: test.unknow.host: Name or "..., 50ping: test.unknow.host: Name or service not known
> ) = 50
> exit_group(2)                           = ?
> +++ exited with 2 +++

从两段strace结果可以看到 会先去nscd去获取结果

> nscd:   name service cache daemon
>
> Nscd is a daemon that provides a **cache** for the most common name service requests. The default configuration file, */etc/nscd.conf*, determines the behavior of the cache daemon
>
> 简单翻译下就是nscd是个缓存。它会缓存  *passwd* database(/etc/passwd)  或者 hosts database(*/etc/hosts* and */etc/resolv.conf* for the *hosts* database)  来提高性能

- 初步分析

  > test.localhost 可以从 nscd中读到数据  test.unknow.host 中都不到数据

#### 那么排除nscd 缓存，再看下呢

> 因为要确认到底有没有去读 /etc/hosts 

```
systemctl stop nscd 
```

- strace ping -c 1 test.unknow.host

  > **openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 5**
  > fstat(5, {st_mode=S_IFREG|0644, st_size=266, ...}) = 0
  > lseek(5, 0, SEEK_SET)                   = 0
  > read(5, "127.0.0.1\tlocalhost\tlocalhost.lo"..., 4096) = 266
  > read(5, "", 4096)                       = 0
  > close(5)   
  >
  > ...
  >
  > socket(AF_INET, SOCK_DGRAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_IP) = 5
  > setsockopt(5, SOL_IP, IP_RECVERR, [1], 4) = 0
  > **connect(5, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("100.100.2.136")}, 16) = 0**

可以看到确实读了/etc/hosts。 而且因为没找到需要的test.unknow.host 。 又去请求了dns

- strace ping -c 1 test.localhost

  > **openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 5**
  > fstat(5, {st_mode=S_IFREG|0644, st_size=266, ...}) = 0
  > lseek(5, 0, SEEK_SET)                   = 0
  > read(5, "127.0.0.1\tlocalhost\tlocalhost.lo"..., 4096) = 266
  > read(5, "", 4096)                       = 0
  > close(5)                                = 0
  > ...
  > **socket(AF_INET, SOCK_DGRAM, IPPROTO_IP) = 5**
  > **connect(5, {sa_family=AF_INET, sin_port=htons(1025), sin_addr=inet_addr("127.0.0.1")}, 16) = 0**
  > getsockname(5, {sa_family=AF_INET, sin_port=htons(49162), sin_addr=inet_addr("127.0.0.1")}, [16]) = 0
  > close(5)                                = 0
  > ...
  > fstat(1, {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0), ...}) = 0
  > write(1, "PING test.localhost (127.0.0.1) "..., 54PING test.localhost (127.0.0.1) 56(84) bytes of data.

对比看下 test.localhost是从/etc/hosts读到了数据



#### 到这里大概知道问题范围了 

> 现在需要问的问题是:  为什么从/etc/hosts 读不到 test.unknow.host

- [x] 确认字符匹配

  >  确实test.unknow.host字符找到了  

- [x] 从代码的角度看 解析有bug ?  了解了下 ping 会先调用  getaddrinfo 去获取 hostname的ip地址

  

  ```c
  #include <stdio.h>
  #include <sys/types.h>
  #include <sys/socket.h>
  #include <netdb.h>
  #include <arpa/inet.h>
  int main(void) {
    struct addrinfo* addr;
    int result = getaddrinfo("test.localhost", NULL, NULL, &addr);
    if (result != 0) {
      printf("Error from getaddrinfo: %s\n", gai_strerror(result));
      return 1;
    }
    struct sockaddr_in* internet_addr = (struct sockaddr_in*) addr->ai_addr;
    printf("test.localhost is at: %s\n", inet_ntoa(internet_addr->sin_addr));
    return 0;
  }
  ```

  尝试使用vscode去打断点后发现太复杂了。找不到具体的读取/etc/hosts的地方

#### 遇事不决GPT

- when to search /etc/hosts  in   `glibc-2.31/sysdeps/posix/getaddrinfo.c`   of glibc2.31

  ```
  However, the search into /etc/hosts isn't directly specified in getaddrinfo.c. The resolution is abstracted in this file and the actual parsing of /etc/hosts is done in the files/ subdirectory of the nss (Name Service Switch) part of glibc.
  ```

- what file in  nss directory 

  ```
  This is located via this path: nss/nss_files/files-hosts.c
  ```

- which function ?

  ```
  The primary function responsible for parsing the /etc/hosts file in glibc's nss_files/files-hosts.c is the gethostbyname3_r function.
  ```

- continue
  - 问：

  ```
   this is content of /etc/hosts:
   --
    127.0.0.1　 test.unknow.host

    why gethostbyname3_r   can not reslove test.unknow.host?
  ```
  
  - 答：

  ```
  If gethostbyname3_r function can't resolve test.unknown.host, there might be several reasons behind it. Here are a few things you might want to check:
  1. The entries in /etc/nsswitch.conf
  2. The /etc/hosts file format 
  3. The system's DNS Caching
  4. Environment Variables
  5. Code Error
  6. Permissions
  ```

- continue

  - 问

    ```
    content of /etc/hosts
    ```127.0.0.1　 test.unknow.host```  
    
    ---
    can u find the specific error 
    ```

  - 答:

    ```
    The potential issues here could be:
    
    Non-ASCII Space: Between the IP address and the hostname, it is critical to use ASCII spaces
    ```

- how  to  find Non-ASCII Space

  ```
  LC_ALL=C grep -n '[^[:print:][:space:]]' /etc/hosts
  ```

  ![image-20240314090953401](https://shunzhou.me/img/typro/image-20240314090953401.png)



### 总结

- 学习了 dns解析的路径，了解到还有nscd缓存服务
- 搭建了debug  glibc的 环境 。
