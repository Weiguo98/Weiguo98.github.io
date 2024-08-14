---
title: "Ansible complex set up error handling"
date: 2024-08-14T16:26:00+01:00
---

// 中文版在下面（Chinese version down below）

Handling errors during environment configuration with Ansible is crucial. While ignore_errors and fails_when keywords exist, they only apply to specific tasks, not to roles or entire playbooks.

### Problem Description

`ignore_errors=true` is ineffective in Ansible roles, only working within sessions. This requires repeatedly adding this option in test and subsequent tasks to ensure the next test runs even if the previous one fails. How can we avoid overusing ignore_errors and skip tests if pre-tasks fail?

### Solution

 Refactor test cases using block, rescue, always, and `clear_host_errors`.

Example:

```ansible
- block:
    - name: Pre tasks
      command: /bin/false
    - name: Test tasks
      command: /bin/false
      register: result
  rescue:
    - name: Record error
      ansible.builtin.debug:
        var: result
  always:
    - name: Post tasks
      command: echo "post tasks"
    - name: Clear host errors
      meta: clear_host_errors
```

### Detailed Explanation

1. Using ignore_errors:

    In Ansible, if a task fails, Ansible will stop executing tasks on that host by default. However, you can use `ignore_errors` to ignore errors and continue execution. For example:

    ```ansible
    - name: Attempt to Execute a Command
    command: /some/nonexistent/command
    ignore_errors: true # This task will fail, but the playbook will continue.
    ```

1. Using block, rescue, and always
    By breaking down each role's tasks into multiple parts and using the block, rescue, and always keywords, more granular error handling can be achieved. As seen in the example above, if the pre task fails, the test task will not be executed because they are in the same block. The failure of an Ansible task will trigger the execution of the rescue block, where we can log errors. In the example, debug is used, but you can also write to a verdict/any file so that we can judge the overall test results in the end. Regardless of the execution status of the tasks in the block, the post task will always be executed. In the post task, you can add tasks such as collecting logs. Even if it fails, the tasks in always will still be executed. Finally, by using clear_host_error, the error is removed, so Ansible will not stop executing tasks on that host, and the next test block will still be executed.

This structure can build multiple tests in a playbook without affecting each other, which can reduce the number of VMs that need to be started at the same time to a certain extent.

#### References

- [Ansible Error Handling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html)

---

在使用Ansible进行环境配置时，处理错误是一个关键问题，尽管有`ignore_errors`和`fails_when`两个keyword支持，但他们只能应用在具体的task，不能应用在role/整个playbook上。本文将介绍如何在复杂设置中如何更好的处理错误，特别是对于应用ansible在测试的情况下。

### 问题描述
ignore_errors=true在Ansible角色中不起作用，只在会话中有效。需要在测试任务和后续任务中多次添加此选项，以确保如果测试或者仅仅是不能够获取日志，下一个测试仍然可以运行。那如何更好的处理这种情况避免滥用`ignore_errors` 以及如果某个测试的预处理任务失败了，该测试可以被跳过，而不是被执行？

解决方案是，可以重构测试用例，使用`block`、`rescue`、`always`和`clear_host_errors`。
以下是一个可以工作的示例：

```ansible
- block:
    - name: Pre tasks
      command: /bin/false
    - name: Test tasks
      command: /bin/false
      register: result
  rescue:
    - name: Record error
      ansible.builtin.debug:
        var: result
  always:
    - name: Post tasks
      command: echo "post tasks"
    - name: Clear host errors
      meta: clear_host_errors

```

### 详细说明

1.使用ignore_errors
在Ansible中，如果一个任务失败，Ansible默认会停止在该主机上执行任务。但是，你可以使用ignore_errors来忽略错误并继续执行。例如：

```ansible
- name: Attempt to Execute a Command
  command: /some/nonexistent/command
  ignore_errors: true # This task will fail, but the playbook will continue.
```

2.使用block、rescue和always
通过将每个角色的任务分解为多个部分，并使用block、rescue和always关键字，可以实现更细粒度的错误处理。 从上面的例子可以看到如果pre task失败了，test task不会被执行，因为他们在一个block里面。ansible task的失败会trigger ansible执行rescue block， 在这里我们可以记录错误，例子中只是使用了debug，也可以写入到verdict/任何文件中。无论block中的task执行状态如何，都会执行post task，在post task中可以加入collect logs等任务，即使失败，在always中的任务仍然会被执行。最后通过clear_host_error 将error删除，这样ansible不会停止在该主机上执行任务，下一个test block仍然会被执行。

这种结构可以构建多个test在一个playbook中，并且互不影响，能够从一定程度上减少同一时间需要启动的vm的数量。

#### 参考资料

- [ansible error handling](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html)
