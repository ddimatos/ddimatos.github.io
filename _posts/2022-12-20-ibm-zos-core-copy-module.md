---
layout: post
title: A deep dive into the z/OS® core copy module
subtitle: Deep dive into the modules architecture
thumbnail-img: /assets/img/magnifying_glass_250_px.jpg
# share-img: /assets/img/code2.png
# tags: [ansible, zos, ibm_zos_core]
---

It's time we dive into the `zos_copy` module redesign that became available with
**Ansible®** Core 1.4.0. At this time, the module is just over 2 years old and one
of our most popular modules used to maintain customers production systems. You
are probably wondering why we would decide to redesign the entire module, well
there is no particular reason but several good ones. As we collected feedback
on how our customers were using the module, exploring the Ansible core community
code and seeing how our initial design had served its purpose; now it was time
to redesign it with maintainability in mind.

<ins>Plugins and modules</ins>

The `zos_copy` module is actually comprised of two parts: one is an action plugin
and the other is the Ansible module. Not all modules require a corresponding
plugin, but they must be named the same so that Ansible can find them when the
playbook is executed. The purpose of the action plugin is to handle all the
controller-based operations. The controller is where Ansible is running, this
could be your personal laptop, production server or Ansible Tower, to name a few.
The action plugin is going to perform controller operations such as ensuring that
a local file is present and then either use **SCP** or **SFTP** to send data to the
managed z/OS® node. The plugin tracks the controller's locale, collects some
stats and then transmits this to the module during execution. A good example of
this is when `zos_copy` is copying a file on the controller to the managed z/OS
node. It will check the system's locale, and put that information into a payload
and share it with the module so that the file is properly encoded for the
managed node.

<ins>SSH interaction</ins>

The module's underlying transport protocol is **SFTP**. This was originally managed
by spawning a process and initializing **SFTP** with Python's `subprocess`. Soon enough,
we witnessed that the isolated process had no way of obtaining Ansible's
configurations and in some instances you might be prompted to enter your SSH
password, if you were not using **SSH** passwordless authentication. This was
resolved by leveraging the Ansible **SSH** connection instance and passing that
along to our module. In doing so, our module can access any options you configure
in Ansible, such as the
[port](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html#parameter-port)
which is also why we deprecated the option **sftp_port**. The added flexibility does
allow a user to override our supported **SFTP** protocol, so we perform some analysis
and if needed, we toggle the transport mode to **SFTP** and log this operation, which
can be seen when you run a playbook with verbosity.

<img width="906" alt="image" src="https://user-images.githubusercontent.com/25803172/205431471-bc733161-66a6-4d88-9931-13bc0b425e5c.png">

<ins>Updated interface</ins>

The next update is the expanded interface to allow for the module to create
data sets to copy data into from within the same module. You might be asking why
we added data set creation in the module when you can use the `zos_data_set` module
to create data sets. There are a couple of reasons: the first is to simplify
playbooks, after all, if you think about commands **tar** and **gzip**, they are separate
utilities yet their combined function can be found in tar alone. The second
reason is performance, it's one less task and interaction that needs to take
place. Lastly, it enables our precedence rules, which we will talk more about
later.

Let's have a look at how this simplifies playbooks. In this snippet we are
copying a UNIX file into a sequential data set using the enhanced interface.

```
- name: Copy a USS file to a fully customized sequential data set
  zos_copy:
    src: /path/to/uss/src
    dest: SOME.SEQ.DEST
    remote_src: true
    volume: '222222'
    dest_data_set:
      type: SEQ
      space_primary: 10
      space_secondary: 3
      space_type: K
      record_format: VB
      record_length: 150
```

Now lets have a look at how to do this using both `zos_copy` and `zos_data_set`.

```
- name: Copy a USS file to a fully customized sequential data set
  zos_copy:
    src: /path/to/uss/src
    dest: SOME.SEQ.DEST
    remote_src: true
```

```
- name: Create a sequential data set if it does not exist
  zos_data_set:
    name: SOME.SEQ.DEST
      type: SEQ
      space_primary: 10
      space_secondary: 3
      space_type: K
      record_format: VB
      record_length: 150
      volumes: "222222"
```

Not only is it condensed, if you are registering variables to capture STDOUT,
you only need to register one variable.

<ins>Precedence rules</ins>

Precedence rules is a concept we introduced this release which defines the order
data will be written. The redesign introduced more than one way to define the
destination data set, thus we decided on a logical ordering to handle this
definition which we will explain in more detail.

With the introduction of option **dest_data_set**, deprecating the previous option
**destination_dataset**, you can create a data set with specific attributes where the
data set **name** is defined by the **dest** option. Since the **dest** defines the data set
name, it is possible that the destination is pointing to an existing data set
which could be empty, making it difficult for the module to know if the user
intended to use the empty data set or if they wanted to allocate a new data set.
This scenario introduces complexity for the module. If **dest** is configured with
an existing empty data set and the attributes for **dest_data_set** have been set;
what do we do? Do we use the empty data set? Do we check that the empty data set
has adequate space before using it? Do we create a new data set using the
provided attributes? This is why we have created precedence rules, they help us
answer these questions, so let's look at what that flow is going to look like.

When the option **dest_data_set** is configured, we consider this a conscious action.
If you have set the attributes, we interpret that as the action you want to occur.
After all, you went to the trouble to configure a number of attributes, this is
why this action takes precedence over all others. With **dest_data_set** you can set
things such as the **record** **format** and **block** **size** of a data set, but
also the **key** **length** and **key** **offset** for VSAMs and **storage**,
**data and management class** for SMS-managed data sets.

Continuing with a similar example, let's pretend that you did not specify
**dest_data_set** and that **dest** is pointing at an empty data set. In this case,
we will try to use it as is, assuming that you created it to suite your
requirements. On the other hand, when the destination does not exit, we need a
way to create a new data set and the logical choice is that we use the src data
set as a model. We will read all the src attributes and use them to create a new
**dest** data set to copy the contents into it.

If the **src** is a UNIX file, there will not be any data set attributes to read but
we do have some file stats we can leverage. We will use the **src** UNIX file to
compute the size of the data and if its not binary, we will read the file to
determine the longest record length (LRECL) and then create a physical
sequential (PS) with a Fixed Block (FB) record format.

For the remaining destination data set types such as VSAM or UNIX files, no new
rules apply, those interactions remain the same.

<img width="827" alt="image" src="https://user-images.githubusercontent.com/25803172/205479351-e70201b9-b7d3-41a1-81c5-da1a5ea7df9e.png">

It's worth noting that the option **force** was enhanced so that when the **dest**
is not empty, the dest will be deleted and recreated using the **src** as the model.
The logic behind this decision is that there is no point on assuming the **dest**
space is enough for the data, thus the **src** is used as the model to create the
destination.

<ins>Community alignment</ins>

Over time, modules can diverge in behavior in comparison to the community modules.
In this case, the community module is `ansible.builtin.copy`. We try to model our
module interfaces and behaviors after the community modules to ensure an easier
migration to our modules. In some cases we do diverge, this is expected when
supporting z/OS leaving us to occasionally define our module's behavior.

For example, to align with the community module, initially our module did not
create parent directories when they did not exist, after careful review of the
community module, we enhanced `zos_copy` to create parent directories when they
don't exist in the destination path. For instance, if the **dest** path
is **/top-level/files/file.txt** and directory **/files** did not exist, the module
would exit in previous versions, now the module will create **/files** and
even **/top-level** if needed so long as permissions will allow.

Now when permission bits are set with option **mode**, permissions are applied to
both directories and files. In an effort to align our module with the community,
we distinguish between copying a directory and copying directory content. A
directory copy is accomplished by **omitting** the trailing slash (**/**) in the **src**,
for example **src: /new/dir**. This results in copying the directory **/dir** to the
managed node. On the other hand, when **src: /new/dir/** where **src** has a trailing
slash (**/**), the module will copy only the **contents** that are in the
directory **/dir** to the destination (dest).

When the **src** is a directory (trailing slash omitted) and the mode is set,
there could exist other files in the **dest**; to prevent the module from
altering those existing file permissions in the destination, the module will
review the src content and ensure the mode is only applied only to the **src** files
being copied.

For example, this task will copy the directory **/some-dir** to **/u/omvsadm**.
This will result in **/u/omvsadm/some-dir** with directory permissions **644**,
owner **omvsadm** and group **admin**.

```
- name: Copy directory 'some-dir' and set permissions
  zos_copy:
    src: /path/to/a/some-dir
    dest: /u/omvsadm
    mode: 0644
    group: admin
    owner: omvsadm
```

In the next example, this task will copy the directory contents in **some-dir**
to **/u/omvsadm** where the contents will have permissions **644**,
owner **omvsadm** and group **admin**. Notice the trailing forward slash **/**
that differentiates this task from the prior.

```
- name: Copy directory contents and set permissions
  zos_copy:
    src: /path/to/a/some-dir/
    dest: /u/omvsadm
    mode: 0644
    group: admin
    owner: omvsadm
```

<ins>Maintainability and design</ins>

In the beginning, we noted how the original design had served us well, and
leveraging what we learned, we decided to redesign the module to allow us to
maintain the module with ease, accuracy and continue extending it. Our goal is
always to develop code such that it is organized, reusable, isolated and ensure
modules can be easily maintained and extended.

The `zos_copy` module has a number of handlers whose function is to handle
copying data from one type of source to another. For example, we might have
a **pds_to_pdse** or a **uss_to_seq** handler and under the new design, these
handlers have been enhanced to only perform copy operations. In the prior design,
the handlers were given the responsibility to decide if the **dest** was the right
type, adequate in size, existed and even the responsibility to create the
destination. To maintain this type of logic the handlers, it became complex and
redundant. With the **precedent rules** now in place, the handlers can expect the
destination will be evaluated and ready to copy data into by the time they are
invoked. Now the handlers only task is to copy data.

The **precedence rules** help in creating a contract for the entire module such
that no handler needs to decide if he destination is the right type, size, etc.
The decision is made early in the code on whether the destination is adequate
and if not, the code will create the destination before invoking any handlers,
alleviating the handlers responsibility.



<ins>Wrapping it up</ins>

As you can see, we are constantly reinventing, improving and changing both the
modules capabilities as well as architecture. We are continuously expanding our
pipelines with linters, scanners, profilers and are always looking to improve
and expand our collection.


In the next blog, we will talk about our migration to GitHub project to manage
all our issues and how we are working towards 100% transparency.

Resources
[IBM Ansible Core Version 1.4.0 on Galaxy](https://galaxy.ansible.com/ibm/ibm_zos_core)
[IBM Ansible Core Collection Repository on GitHub](https://github.com/ansible-collections/ibm_zos_core)
[IBM Ansible Core Collection on Automation Hub](https://console.redhat.com/ansible/automation-hub/repo/published/ibm/ibm_zos_core)
[Red Hat® Ansible Certified Content for IBM Z documentation](https://ibm.github.io/z_ansible_collections_doc/index.html)