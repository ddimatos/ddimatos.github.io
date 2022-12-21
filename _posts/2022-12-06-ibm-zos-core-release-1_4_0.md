---
layout: post
title: Ansible z/OS Core 1.4.0
subtitle: Let's discuss the changes
thumbnail-img: /assets/img/open_source_250_px.png
# share-img: /assets/img/code2.png
# tags: [ansible, zos, ibm_zos_core]
---

Ansible Core 1.4.0 has been in the making for some time and is finally being
released into Galaxy and Automation Hub. This release brings a new module,
`zos_mount` that can manage mount operations for a z/OS UNIX System Services
file system as well as many other enhancements and some deprecations which
we will discuss in more detail.

<ins>SSH interaction</ins>

Modules `zos_copy` and `zos_fetch` were enhanced to inherit options configured
for the connection plugin [ansible.builtin.ssh](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html)
which is usually managed with
[ansible.cfg](https://docs.ansible.com/ansible/latest/reference_appendices/config.html).
For example, you may want to override the default time for SSH option
[ControlPersist](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html#parameter-ssh_args)
such that it persists for 120 seconds; now those changes are propagated to
the collection which is also why the module option `sftp_port` has been
deprecated because it can be configured using the Ansible option
[port](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html#parameter-port).


![image](https://user-images.githubusercontent.com/25803172/205228434-eb6166d9-2c8a-4a13-94e5-68374e5ab350.png)

<ins>Significant redesign</ins>

`zos_copy` has been redesigned to allow for data sets to be defined using option
`dest_data_set` which follows our precedence rules that control the order in
which the destination data is written. Since `zos_data_set`
and `zos_copy` are two of the most frequently used modules, we decided to enhance
``zos_copy`` so that with few more lines of code, you can also create data sets
to copy data from within the `zos_copy` module.

<ins>Precedence rules</ins>

The next major change is the introduction of our precedence rules which define
how `zos_copy` will write to a destination. For example, if `dest_data_set`
is defined, it takes precedence over an empty data set because a conscious effort
was made to set the option values. If the destination data set is empty, it will
be used with the expectation the attributes support the copy. Lastly, if no
precedent rule has been exercised and the source is a data set, the module will
create the destination data set using the source data set attributes. If the
source is a UNIX file, the data set will be created with a Fixed Block (FB)
record format and the remaining attributes will be computed. There are many more
enhancements such as aligning `zos_copy` behaviors to the community module, but
those will be discussed in a future blog that will dive deeper into the modules
architecture.

![image](https://user-images.githubusercontent.com/25803172/205228487-eaa4b949-c2ad-4ac4-9fc5-8ef99a4a59b2.png)

<ins>Enhancements</ins>

Module ``zos_job_output`` was enhanced to include the completion code (CC) for
each individual job step which is helpful in identifying which step might be
resulting in a job failure. Additional updates were made that improve the
result when a ddname is specified and some cases that triggered the job's
content to be truncated.

``zos_job_query`` was enhanced to support up to a 7 digit job number ID , for
example, job ID `J1234567` as well as `JOB12345` are both supported. This is
helpful for when there are greater than 99,999 jobs in the job history.

In an effort to improve job submission when using `zos_job_submit`, it was
enhanced to to fail fast over waiting the entire `wait_time_s` a user has set.
Additionally, other logic was added to scan for job submission failures in order
to return a proper response. For example, if the JCL contained a syntax error,
the module will be looking for keywords such as `JCL ERROR` to properly fail and
deliver an appropriate message.

``zos_operator`` was enhanced to allow using the MVS operator `SET` command where
in the past only the abbreviated equivalent `T` could be used. As well,
`zos_ping` was enhanced so it does not use the now **deprecated** `zos_ssh`
connection plugin.

<ins>Deprecations</ins>

Along with the updates and enhancements there are some deprecated functions to
note. If you are using ``zos_copy`` or ``zos_fetch`` option `sftp_port`, it has
been deprecated and will be removed in the next release. Instead, you should use
the `port` option available in
[ansible.builtin.ssh](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html).
``zos_copy`` module option ``model_ds`` has been removed
because the modules redesign made this option no longer necessary. If you were
using the ``zos_copy`` module from `1.4.0-beta.1` option `dest_dataset` it has
been renamed to `dest_data_set`. Lastly, if you were using the `zos_ssh` plugin
it has been removed so you must remove it from any playbooks using Ansible
z/OS core 1.4.0 or later.

![image](https://user-images.githubusercontent.com/25803172/205228545-4c513a79-1120-4770-bc02-4399613bffd5.png)

<ins>Dependencies</ins>

In this release, the dependencies changed in that now IBM
[Open Enterprise SDK for Python](https://www.ibm.com/products/open-enterprise-python-zos)
version [3.9.x](https://www.ibm.com/docs/en/python-zos/3.9) is the minimum
Python level and IBM Z Open Automation Utilities remains the same with version
[1.1.x](https://www.ibm.com/docs/en/zoau/1.1.x).

In closing, there are many more changes that were not discussed, thus I encourage
you to review the [release notes](https://ibm.github.io/z_ansible_collections_doc/ibm_zos_core/docs/source/release_notes.html#version-1-4-0), [changelog](https://github.com/ansible-collections/ibm_zos_core/blob/v1.4.0/CHANGELOG.rst) and updated [documentation](https://ibm.github.io/z_ansible_collections_doc/index.html).

Resources
[IBM Ansible Core Version 1.4.0 on Galaxy](https://galaxy.ansible.com/ibm/ibm_zos_core)
[IBM Ansible Core Collection Repository on GitHub](https://github.com/ansible-collections/ibm_zos_core)
[IBM Ansible Core Collection on Automation Hub](https://console.redhat.com/ansible/automation-hub/repo/published/ibm/ibm_zos_core)
[Red Hat® Ansible Certified Content for IBM Z documentation](https://ibm.github.io/z_ansible_collections_doc/index.html)