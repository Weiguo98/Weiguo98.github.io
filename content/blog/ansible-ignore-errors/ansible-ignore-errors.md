---
title: "Ansible complex set up error handling"
date: "2024-08-14"
tags: ["ansible"]
---

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
