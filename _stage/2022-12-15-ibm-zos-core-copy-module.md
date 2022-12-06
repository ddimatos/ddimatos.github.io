---
layout: post
title: Ansible z/OS copy module architecture
subtitle: Deep dive into the modules architecture
thumbnail-img: /assets/img/magnifying_glass_250_px.jpg
# share-img: /assets/img/code2.png
# tags: [ansible, zos, ibm_zos_core]
---

It's time we dive into the `zos_copy` module redesign that became available with
Ansible Core 1.4.0. At this time, the module is just over 2 years old and one of
our most popular modules used to maintain customers production systems. You are
probably wondering why we would decide to redesign the entire module, well there
is no particular reason but several good ones. As we collected feedback on how
our customers were using the module, exploring the Ansible core community code
and seeing how our initial design had served its purpose; now it was time to
redesign it with maintainability in mind.

<ins>Plugins and modules</ins>

The `zos_copy` module actually is comprised of two parts, one is an action plugin
and the other is the Ansible module. Not all modules need to have a
corresponding plugin, but they do need to be named the same such that Ansible
can find them when the playbook is executed. The purpose of the action
plugin is to handle all the controller based operations. The controller is where
Ansible is running, this could be your personal laptop, production server or
Ansible Tower to name a few. The action plugin is going to perform controller
operations such as ensuring a local file is present and then either SCP or SFTP it to
the managed z/OS node. The plugin tracks the controllers locale, collect some stats and then transmits
this to the module during execution. A good example of this is when `zos_copy` is copying
a file on the controller to the managed z/OS node. It will check the systems locale,
and put that information into a payload and share it with the module so that
the file is properly encoded for the managed node.

<ins>SSH interaction</ins>

The modules underlying transport used is SFTP and this was originally managed by
spawning a process and initializing SFTP with Pythons subprocess. Soon enough we
witnessed that the isolated spawned process had no way of obtaining Ansible's
configurations and in some instances you might be prompted to enter your SSH
password if you were not using SSH passwordless authentication. This was resolved
by leveraging the Ansible SSH connection instance and passing that along to our
module. In doing so, now our module can access any options you configure in Ansible, such as
[port](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html#parameter-port)
which is also why we deprecated the option `sftp_port`. The added flexibility
does allow a user to override our supported transport SFTP, so we perform some
analysis and if needed, we toggle the transport mode to SFTP and log this operation
which can be seen when you run a playbook with verbosity.

<img width="906" alt="image" src="https://user-images.githubusercontent.com/25803172/205431471-bc733161-66a6-4d88-9931-13bc0b425e5c.png">

<ins>Updated interface</ins>

The next update is the expanded interface to allow for the module to create
data sets to copy data into from within the same module. You might be asking
why we added data set creation in the module when you can use the `zos_data_set`
module to create data sets. There are a couple of reasons; the first is to simplify
playbooks, after all, if you think about commands `tar` and `gzip` are separate
utilities yet their combined function can be found in `tar` alone. The second
reason is performance, its one less task and interaction that needs to take place.
Lastly, is that it enables our precedence rules which we will talk more about later.

Lets have a look at how this simplifies playbooks. In this snippet we are copying
a UNIX file into a sequential data set using the enhanced interface.

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

Precedence rules is a concept we introduced which defines the order data will
be written. The redesign introduced more more than one way to define
the destination data set, thus we decided on a logical ordering which we will
explain in more detail.

With the introduction of option `zos_data_set`, you can create a data set with
specific options where the data set **name** will come from the option `dest`.
Since the name comes from the `dest` option, it's possible that value is pointing
to an existing data set which could be empty making it difficult for the
module to know if the user intended to use the empty data set or they wanted to set
the new data set name.

This introduces a bit of a complex issue, we have `dest` pointing at
and existing data set, the data set is empty and also the attributes for
`zos_data_set` have been set, so what do we do? Do we use the empty data set? Do
we check that the empty data set has adequate space before using it or do we
create a new data set using the provided attributes? This is why we have
precedence rules, they help us answer these questions, so lets look at what that
flow is going to look like.

When the option `zos_data_set` is configured, we consider this a conscious
action, if you have set the attributes we interpret that as the action you want
to occur. After all, you went to the trouble to configure a number of attributes,
this is why this action takes precedence over all others.

Continuing with a similar example, lets say that you did not specify `zos_data_set`
and that `dest` is pointing at an empty data set. In this case we will review the size
of the data in `src` to ensure that the empty `dest` can contain the data, if there
is not enough space then we move to our next precedent rule. Assuming we determine
that the empty data set does not have enough space, we need a way to create
a new data set and the logical choice is that we use the `src` data set as the model.
We will read all the `src` attributes and use them to create a new `dest`
data set and copy the contents into it.

If the `src` is a Unix file, there will not going to be any data set
attributes to read. Well, we do have some stats we can leverage, just no data set
attributes. We will use the `src` Unix file to compute the size of the data and
if its not binary, we will read the file to determine the longest record
length (LRECL) and then create a physical sequential (ps) with a
Fixed Block (FB) record format.

For the remaining destination data set types such as VSAM or Unix files, no
new rules apply, those interactions remain the same.

<img width="827" alt="image" src="https://user-images.githubusercontent.com/25803172/205479351-e70201b9-b7d3-41a1-81c5-da1a5ea7df9e.png">

It's worth noting that the option `force` was enhanced so that when the `dest`
is not empty, the `dest` will be deleted and recreated using the `src` as the
model. This is done this way because there is no point on assuming the `dest`
capacity is enough for the data, hence it makes sense to use the `src` as the
model.


<ins>Community alignment</ins>

Over time, modules can diverge in behavior in comparison to the
community modules, in this case the community module is `ansible.builtin.copy`.
We always try to model the community interface and behavior, in some cases we
can do so but because z/OS is quite different, sometimes we have to define our
own modules behavior.

To align with the community module, `zos_copy` was enhanced to create the parent
directory when it does not exist in the path. For example, if the `dest` path
was `/top-level/files/file.txt` and `files` in the path did not exist, in the
prior versions the module would exit, now the module will create `files` and
even `top-level` directories if needed.

When permission bits are set with option `mode` for `zos_copy`, now it applies
the mode to both directories and files. A directory copy is accomplished
by **not** having a trailing forward slash (`/`) in the `src`, for
example `src` = `/new/files`. This results in copying the **directory** `files`
to the managed node. When `src` = `/new/files/`  (no trailing `/`) will copy the
**contents** that are in directory files to the managed node.

When the `src` is a directory being copied and mode is set, there could exist
other files in the `dest`. To prevent the mode from altering file permissions
in the destination, the module will track the `src` content and ensure mode is
applied only to the `src` files being copied.

For example, this task will copy the **directory** `some-dir` to
`/u/omvsadm` that will result in `/u/omvsadm/some-dir` with directory
permissions **644**, owner **omvsadm** and group **admin**.

```
- name: Copy directory 'some-dir' and set permissions
  zos_copy:
    src: /path/to/a/some-dir
    dest: /u/omvsadm
    mode: 0644
    group: admin
    owner: omvsadm
```

For example, this task will copy the directory **contents** in `some-dir` to
`/u/omvsadm` where the contents will have permissions **644**,
owner **omvsadm** and group **admin**. Notice the trailing forward slash `/`
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

In the beginning we noted how the original design had served us well and now
leveraging what we learned, we decided to redesign the module to allow us to
maintain the module with ease, accuracy and continue to extended it. We always
strive to have organized code, logically isolate function and design it so it
it can be easily maintained and extended.

The `zos_copy` module has a number of copy handlers who's function is to handle
copying data from one type of source to another. For example, we might have a
`pds_to_pdse` or a `uss_to_seq` copy handler, under the new design the copy
handlers have only one purpose, to perform only the copy operation.

In the prior design, the copy handlers were given the responsibility to decide if
the `dest` is the right type, adequate in size, exits and even the responsibility to
create the destination. To maintain this type of logic in each copy handler, it
became complex and redundant. Now the copy handlers only task is to copy data.

Recall the precedence rules, these rules help in creating a contract for the entire
module such that no copy handler will concern itself on whether the destination
is the right type, size, etc. Now the decision is made early in the code on
whether the destination is adequate and if not, the code will create the
destination before invoking any handlers alleviating the handlers responsibility.

<ins>Wrapping it up</ins>

As you can see, we are constantly reinventing, improving and changing both
the modules capabilities as well as architecture. We are continuously
expanding our pipelines with linters, scanners, profilers and are always
looking to improve and expand our collection.

In the next blog, we will talk about migration to GitHub project to manage all
our issues and how we are working towards 100% transparency. 
