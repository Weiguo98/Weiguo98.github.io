---
title: "Security Policy in QNX"
date: "2025-08-25"
tags: ["QNX"]
---

## What is security policy?

Official documents: <https://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.security.system/topic/manual/security_policies.html>

It basically is a file that in a system level set processes which operations it is permitted to do.

## How is it used?

First, a secpol.txt can be get it from `secpolgenerate` or simply from any product itself.

1. First, compile an executable called secpol.bin using secpolcompile. <https://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.neutrino.utilities/topic/s/secpolcompile.html>

2. Then, put it in /proc/boot. In some environment, you may not have access to modify the /proc/boot folder. Instead, we can create a link to it.

    `ln -sP /root/secpol.bin /proc/boot/secpol.bin`

3. Finally, run `secpolpush`. The security policy is pushed to the kernel, and it works immediately.

This means, now, if you would like to start any process, you need to assign a security policy to it, otherwise it will use `init_t`  as the default security policy type.

## Observability

Use `secpolmonitor` to monitor security events:

Run it to show uses of abilities and path space changes ([detailed command documentation](https://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.neutrino.utilities/topic/s/secpolmonitor.html)):

`secpolmonitor -aps`

Example output:

```txt
info: usr/sbin/sshd (pid:3325972) type init_t uses ability CHROOT as root
info: proc/boot/example_app (pid:1) type default uses ability KEYDATA as root
info: usr/sbin/sshd (pid:3325972) type init_t uses ability SETTYPEID(140) (ssh_privsep_t) as root
info: usr/sbin/sshd (pid:3325972) type ssh_privsep_t uses ability SETGID(6) as root
info: usr/sbin/sshd (pid:3325972) type ssh_privsep_t uses ability SETUID(15) as root
```

Basic syntax explanation:

```txt
# Type means new type definition
type ssh_privsep_t;

# Allow specific secpol type to have different abilities. This part means ssh_privsep_t can setuid and setgid.
allow ssh_privsep_t self:ability {
   setuid:0x0-0xFFFFFFFF
   setgid:0x1-0xFFFFFFFF
};

# Abilities can be separated to make them easier to manage, or you can keep them together.
allow init_t self:ability {
    network/privport
    chroot
}

# This means init_t can have ability iofunc/exec as non-root user.
allow init_t self:ability{
     nonroot
     iofunc/exec
};
```

More detailed explain: <https://www.qnx.com/developers/docs/7.1/index.html#com.qnx.doc.security.system/topic/manual/secpol_language.html>

## Troubleshoot

1. Secpolcompile failure

    Sometimes, we will run into secpolcompile failure, for example:

    ```txt
    error: type 'sshd_t' cannot be granted ability 'settypeid:ssh_privsep_t' as it would gain abilities:
        setuid: range
        setgid: range
    ```

    In this case, the error means that there are conflicts in abilities between sshd_t and ssh_privsep_t. Double check the abilities in both contexts, and make sure they have the same range of setuid and setgid. Be careful when you try to modify the abilities.

2. Operation is not permitted

    ```txt
    on: Operation not permitted (/usr/bin/example_app).
    ```

    It means on cannot start the process because the security type lacks some needed abilities to spawn the process. You can first check the secpolmonitor.log

### Secpolmonitor log

This is the common format for secpolmonitor.log. You can see there is an error saying that init_t lacks the iofunc/exec ability when running as non-root. Simply put iofunc/exec ability into init_t in the secpol.txt file, then recompile and push the policy. It should work.

```txt
info: usr/bin/example_app (pid:xxxx) type init_t uses ability MAP_FIXED(337416937472 - 337416992071) as non-root
info: usr/bin/example_app (pid:xxxx) type init_t uses ability MAP_FIXED(337417058792 - 337417060415) as non-root
info: usr/bin/example_app (pid:xxxx) type init_t uses ability MAP_FIXED(337417056256 - 337417060351) as non-root
error: usr/bin/example_app (pid:xxxx) type init_t lacks ability iofunc/exec as non-root
info: usr/bin/example_app (pid:xxxx) type init_t uses ability PROT_EXEC(146050510848 - 146050768735) as non-root
```

### Pidin info

We can also check the process info via:
`pidin -F "%a  %N %_"`. In the output, we can clearly see if the process is started with correct security type.
