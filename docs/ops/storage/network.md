# 网络存储系统

!!! warning "本文初稿编写中"

本文介绍在服务器上常见的网络存储方案，包括 NFS 与 iSCSI。
<!-- TODO: well, 我们没有使用过 ceph；如果有人有经验想写的话，可能需要放在高级内容里面 -->

## NFS

NFS 是非常常用的文件系统级别的网络存储协议，允许服务器暴露文件系统给多个客户端。
目前 NFS 有两个主要的版本：NFSv3 和 NFSv4。简单来讲：

- NFSv3（1995 年）是无状态协议，支持 TCP 与 UDP（2049，111 以及其他动态端口）
- NFSv4（2000 年）是有状态协议，只支持 TCP（2049 端口），性能更好

NFS 默认没有基于密码等的鉴权机制，仅依靠客户端 IP 地址来鉴权。并且默认没有加密，因此主要用于内部网络共享存储。
以下不涉及有关高级鉴权方法（如 Kerberos）与 TLS 加密的内容。

NFS 在 Linux 上的服务端和客户端实现均有内核态与用户态的选择，如果没有特殊考虑，建议使用内核态实现。

### 服务端配置

服务端需要安装 `nfs-kernel-server` 包。除此之外，NFSv3 支持还需要安装 `rpcbind` 包（对应 NFSv3 协议的 111 端口），该包在目前的 Debian 中以 `rpcbind.socket` 的形式对外提供服务。

<!-- TODO: a link to systemd socket -->

NFS 的导出配置位于 `/etc/exports` 文件中。例如以下的配置：

```exports
/srv/abcde	localhost(ro,no_root_squash,async,insecure,no_subtree_check)
```

将 `/srv/abcde` 目录「导出」给了本机（localhost），并且设置了诸如只读等选项。
在修改配置后，运行 `systemctl reload nfs-server.service`（等价于 `exportfs -r`）重新加载配置。

!!! lab "尝试配置并挂载"

    NFS 在各种主流桌面/服务器操作系统上都有不错的支持。请尝试在你的系统（虚拟机或 Linux 主机）上配置 NFS 服务端，
    并在主机上挂载。如果需要在 Linux 上挂载，可先阅读下面的客户端配置。
    
    需要注意的是，Windows 需要启用 NFS 对应的可选特性，并且不支持 NFSv4。

接下来主要介绍 `exports` 文件的常见配置。以上例子中对应目录被导出到了 `localhost`，但是同样也接受其他格式，例如：

```exports
/var/lib/example 10.0.0.0/25(rw,sync,no_subtree_check,no_root_squash)
/share	*(ro,no_root_squash,async,insecure,no_subtree_check)
/media 192.168.93.2(rw,no_subtree_check) 192.168.93.3(rw,no_subtree_check)
```

分别代表目录导出给了一个 CIDR 网段、所有 IP，以及两个特定的 IP。

!!! warning "使用 `exportfs -a` 检查配置问题"

    `exports` 文件如果编辑不当，可能会导致客户端无法正确挂载，或者带来非预期的安全风险。
    这是在 [RHEL 9 手册](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html-single/configuring_and_using_network_file_services/index#the-etc-exports-configuration-file_exporting-nfs-shares)中举的一个例子：

    ```exports
    /home bob.example.com(rw)
    /home bob.example.com (rw)
    ```

    第一行是正确的配置，而**第二行是错误的配置，实际行为是：bob.example.com 获得只读权限，而其他所有人获得读写权限。**
    这显然是非预期的。我们在本地模仿类似的问题：

    ```exports
    /srv/abcde	localhost (rw,no_root_squash,async,insecure,no_subtree_check)
    ```

    之后执行 `exportfs -a`，可以看到输出了一些警告，提示我们目前的配置存在潜在的问题（一些配置没有显式给出）：

    ```console
    $ sudo exportfs -a
    exportfs: No options for /srv/abcde localhost: suggest localhost(sync) to avoid warning
    exportfs: /etc/exports [2]: Neither 'subtree_check' or 'no_subtree_check' specified for export "localhost:/srv/abcde".
    Assuming default behaviour ('no_subtree_check').
    NOTE: this default has changed since nfs-utils version 1.0.x

    exportfs: No host name given with /srv/abcde (rw,no_root_squash,async,insecure,no_subtree_check), suggest *(rw,no_root_squash,async,insecure,no_subtree_check) to avoid warning
    ```

    可以发现 `localhost` 没有得到正确的配置，而 `*` 对应的配置却是应该给 `localhost` 的。
    在修复（去掉空格）后，再次执行 `exportfs -a`，就不会有警告了：

    ```console
    $ sudo exportfs -a
    $
    ```

在参数部分有一些常见的选项：

- `rw`/`ro`：读写/只读
- `sync`/`async`：同步/异步，后者允许服务器在完成写入操作前就返回，在提升性能的同时可能会导致数据在崩溃/断电时丢失
- `no_subtree_check`：「取消子树检查」——大部分情况下加上不会对安全有太大影响，但是能够提升性能
- `no_root_squash`：默认情况下，客户端的 root 会被映射到特定用户（`nobody`），设置 `no_root_squash` 会让客户端的 root 拥有和服务器 root 一样的文件权限；该选项不影响非 root 用户
- `all_squash`：将所有客户端的用户映射到一个特定的用户（`nobody`）
- `secure`/`insecure`：默认情况下，NFS 服务端只接受客户端使用 1024 以下的端口访问，设置 `insecure` 会放宽这个限制

!!! tip "1024 端口，与 root"

    在传统上，类 Unix 操作系统中只有 root 用户才能使用 1024 以下的端口。因此在很久以前，程序可以假设来自 1024 以下的端口的请求/服务是 root 用户发出/授权的。
    在 NFS 的情况下，这意味着只有 root 才能访问 NFS 服务，没有 root 权限的用户不能够假冒自己为 NFS 客户端，从而以假的用户权限读写数据。
    这在很久以前的内部网络中的服务器场景是有意义的。

    目前 Linux 仍然保留了这个机制，但是程序不应该仅仅依赖于这个机制来保证安全性。

??? note "关于「子树检查」的解释"

    NFS 服务器与客户端之间使用「文件句柄」（file handle）来标识文件。
    如果 NFS 共享（导出）的目录不是文件系统的根目录，那么服务器就需要判断客户端请求的文件句柄是否真的在共享的目录下。
    这一项检查需要从文件的 inode 信息不停向上查找来确认文件确实在共享的目录中。

    （不过要伪造这种「文件句柄」也不是一件容易的事情）

    部分可能有帮助的信息：

    - [Making Filesystems Exportable -- The Linux Kernel  documentation](https://docs.kernel.org/filesystems/nfs/exporting.html?highlight=export_op_nosubtreechk)
    - [Network File System (NFS) | Ubuntu](https://ubuntu.com/server/docs/service-nfs)
    - [Linux NFS faq](https://nfs.sourceforge.net/#faq_c7)

### 客户端配置

## iSCSI