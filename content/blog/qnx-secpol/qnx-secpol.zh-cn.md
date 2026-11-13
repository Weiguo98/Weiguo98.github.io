---
title: "QNX 中的安全策略"
date: "2025-08-25"
tags: ["QNX"]
---

## 什么是安全策略？

官方文档：<https://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.security.system/topic/manual/security_policies.html>

简单来说，它是一个系统级别的文件，规定了每个进程被允许执行哪些操作。

## 如何使用？

首先需要一个 secpol.txt，可以通过 `secpolgenerate` 生成，也可以直接从现有产品中获取。

1. 首先，用 secpolcompile 编译出 secpol.bin。<https://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.neutrino.utilities/topic/s/secpolcompile.html>

2. 然后，把它放到 /proc/boot 下。在某些环境中，你可能没有权限修改 /proc/boot 目录，这时可以创建一个链接：

    `ln -sP /root/secpol.bin /proc/boot/secpol.bin`

3. 最后，运行 `secpolpush`。安全策略会被推送到内核，并立即生效。

> [!IMPORTANT]
> 这意味着从现在开始，启动任何进程都需要为它指定一个安全策略类型，否则会使用默认的 `init_t` 类型。

## 可观测性

使用 `secpolmonitor` 监控安全事件：

运行下面的命令，可以显示能力（ability）的使用情况和路径空间的变化（[命令详细文档](https://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.neutrino.utilities/topic/s/secpolmonitor.html)）：

`secpolmonitor -aps`

输出示例：

```txt
info: usr/sbin/sshd (pid:3325972) type init_t uses ability CHROOT as root
info: proc/boot/example_app (pid:1) type default uses ability KEYDATA as root
info: usr/sbin/sshd (pid:3325972) type init_t uses ability SETTYPEID(140) (ssh_privsep_t) as root
info: usr/sbin/sshd (pid:3325972) type ssh_privsep_t uses ability SETGID(6) as root
info: usr/sbin/sshd (pid:3325972) type ssh_privsep_t uses ability SETUID(15) as root
```

基本语法说明：

```txt {title="secpol.txt"}
# type 表示定义一个新类型
type ssh_privsep_t;

# 允许某个 secpol 类型拥有不同的能力。这里表示 ssh_privsep_t 可以 setuid 和 setgid。
allow ssh_privsep_t self:ability {
   setuid:0x0-0xFFFFFFFF
   setgid:0x1-0xFFFFFFFF
};

# 能力可以拆开写，方便管理，也可以写在一起。
allow init_t self:ability {
    network/privport
    chroot
}

# 表示 init_t 以非 root 用户运行时也拥有 iofunc/exec 能力。
allow init_t self:ability{
     nonroot
     iofunc/exec
};
```

更详细的说明：<https://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.security.system/topic/manual/secpol_language.html>

## 问题排查

1. secpolcompile 编译失败

    有时 secpolcompile 会编译失败，例如：

    ```txt
    error: type 'sshd_t' cannot be granted ability 'settypeid:ssh_privsep_t' as it would gain abilities:
        setuid: range
        setgid: range
    ```

    这个错误表示 sshd_t 和 ssh_privsep_t 的能力有冲突。仔细检查两个类型中的能力，确保它们的 setuid 和 setgid 范围一致。

    > [!WARNING]
    > 修改能力时要小心。

2. Operation not permitted（操作不被允许）

    ```txt
    on: Operation not permitted (/usr/bin/example_app).
    ```

    这表示 `on` 无法启动该进程，因为对应的安全类型缺少启动进程所需的某些能力。可以先查看 secpolmonitor.log。

### secpolmonitor 日志

下面是 secpolmonitor.log 的常见格式。可以看到有一条错误：init_t 以非 root 运行时缺少 iofunc/exec 能力。只需在 secpol.txt 中给 init_t 加上 iofunc/exec 能力，重新编译并推送策略，就可以解决。

```txt {title="secpolmonitor.log"}
info: usr/bin/example_app (pid:xxxx) type init_t uses ability MAP_FIXED(337416937472 - 337416992071) as non-root
info: usr/bin/example_app (pid:xxxx) type init_t uses ability MAP_FIXED(337417058792 - 337417060415) as non-root
info: usr/bin/example_app (pid:xxxx) type init_t uses ability MAP_FIXED(337417056256 - 337417060351) as non-root
error: usr/bin/example_app (pid:xxxx) type init_t lacks ability iofunc/exec as non-root
info: usr/bin/example_app (pid:xxxx) type init_t uses ability PROT_EXEC(146050510848 - 146050768735) as non-root
```

### pidin 信息

也可以通过下面的命令查看进程信息：
`pidin -F "%a  %N %_"`。从输出中可以清楚地看到进程是否以正确的安全类型启动。
