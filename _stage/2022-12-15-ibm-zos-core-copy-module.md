---
layout: post
title: Ansible z/OS Core 1.4.0
subtitle: Let's discuss the changes
thumbnail-img: /assets/img/open_source_250_px.png
# share-img: /assets/img/code2.png
# tags: [ansible, zos, ibm_zos_core]
---

It's time we dive into the `zos_copy` module redesign that became available with
Ansible Core 1.4.0. At this time, the module is just over 2 years old and one of
our most popular modules used to maintain customers production systems. You are
probably wondering why we would decide to redesign the entire module, well there
is no single reason but several strong reasons. As we collected feedback on how
our customers were using the module, learning the Ansible core code base and
seeing how our initial design had served its purpose and now it was time to
rewrite it with serviceability in mind.


The `zos_copy` module actually is comprised of two parts, one is an action plugin
and the other part is the Ansible module. Not all modules need to have a
corresponding plugin, but they do need to be named the same such that as Ansible
core can find them during an Ansible playbook execution. The purpose of the action
plugin is to handle all the controller based operations. The controller is where
Ansible is running, this could be your personal laptop, production server or
Ansible Tower to name a few. The action plugin is going to manage perform controller
operations such as ensure a local file is present to either SCP or SFTP to the managed
z/OS node. It might look at the locale, collect some stats and send them over to
the module during execution. A good example of its usage is copying a file on the
controller to the managed node, it will check the systems locale, and put that information
into a payload and share it with the module.

<ins>SSH interaction</ins>

The modules underlying transport is SFTP and this was originally managed by
spawning a process but soon we saw that the isolated spawned process had no way
of obtaining Ansible's configurations and in some instances you might be
prompted to enter your SSH users password if you were not using SSH passwordless
authentication. We resolved this by leveraging the Ansible community ssh
connection instance and passing that along to our module. In doing so, now our module can
see any options you configure in Ansible, such as
[port](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html#parameter-port)
which is also why we deprecated the option `sftp_port`. While this sounded like a great idea, we also
realized Ansible could be configured to use SCP as its transport protocol which would
disable our module, so now we perform some analysis and if needed we toggle the
transport modes and log this which can be seen when you run a playbook with verbosity.

<img width="906" alt="image" src="https://user-images.githubusercontent.com/25803172/205431471-bc733161-66a6-4d88-9931-13bc0b425e5c.png">

<ins>SSH interaction</ins>

The next update was the expanded interface to allow for the module to create
data sets to copy data into all from within the same module. You might be asking
why we added the data set creation when you can use the zos_data_set module to perform
the operation. There are a couple of reasons; the first is to simplify playbooks, you could
even say its the same reason why commands `tar` and `gzip` exist yet you can
`tar` alone to compress. The second reason is performance, its one lest task and
interaction that needs to take place. Lastly, is that it aligns with our data set
precedence rules which we will talk more about later.

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

<ins>Precedence rules</ins>

Precedence rules is a concept we introduced which defines the order destination
data will be written to. Since the redesign, there is more than one way to define
the destination data set and thus we decided on a logical ordering which we will
explain in more detail.

With the introduction of `zos_data_set` you can create a data set with very specific
options where the **name** will come from the option `dest`. It is
possible the user who configures the module wants to have `zos_copy` create the
data set and accidentally sets `dest` to a data set which exists and is empty where
they had intended really to use `dest` to instruct the module to create that
data set. So here we have a bit of a complex issue, we have `dest` pointing at
and existing data set, the data set is empty and also the attributes for
`zos_data_set` have been set, so what do we do? Do we use the empty data set, do
we check that the empty data set has adequate space before using it or do we
create a new data set using the provided attributes? This is why we have
precedence rules so lets have a look at what that flow is going to look like.

Any time option `zos_data_set` is configured, we consider this a conscious
action, there is no room for error, if you have set the attributes we interpret
that as the action you want to occur. After all, you went to the trouble to
populate a number of attributes, this is why this action takes precedence over
all others.

Continuing with a similar example, lets say that you did not specify `zos_data_set`
and `dest` is an empty data set. Well in this case we are going to size the
data in `src` to make sure that the empty `dest` can contain the data, if there
is not enough space then we move to our next precedent rule. Since we now know
that the `dest` does not have enough space, we need a way to create a `dest` and
the logical choice is that we use the `src` as the model. We read all the `src`
attributes and use them to create a new `dest` data set and copy the contents
into it.

What if the `src` is a UNIX file, you don't have any data set attributes to
read. Well, we actually do have some attributes just not data set attributes. We
use the `src` UNIX file to obtain the data size and if its not BINARY we read the
`src` data to determine the longest record length (LRECL) and take some liberties
and create a physical sequential (ps) with a Fixed Block (FB) record format.

For the remaining destinations data set types such as VSAM or UNIX files, no
new rules apply, those interactions remain the same.

<img width="827" alt="image" src="https://user-images.githubusercontent.com/25803172/205479351-e70201b9-b7d3-41a1-81c5-da1a5ea7df9e.png">

Its worth note that the option `force` was enhanced that when the `dest`
is not empty, the `dest` will be deleted and recreated using the `src` as the
model. This is because there is no point on assuming the `dest` capacity
was enough for the `src`, hence it makes sense to use the `src` as the model. 


<ins>Community alignment</ins>

Over time, module implementations can diverge in behavior in comparison to
community modules, in this case `ansible.builtin.copy`. We always strive to retain
the original interface and behavior, in some case we can and other
cases our module behaves differently because our platform is different.

To align with the community module, `zos_copy` was enhanced to create a parent
directory when it might not exist in the path. For example, if the `dest` path
was `/top-level/files/file.txt` and `files` in the path did not exist the module would
exit without completing the copy. Now the module will create `files` and even
`top-level` if it did not exist.

When permission bits are set (mode), `zos_copy` now supports setting mode for
both directories and files. A directory copy is configured by not having a
trailing forward slash (`/`) in the `src`, for example `/new/files` would copy
the directory `files` to the managed node. Where , `/new/files/` would copy the
contents to the managed node. If the `src` is a directory being copied and mode
is configured, there could exist files in the `dest` and to prevent mode from
altering all the exiting `dest` files, the module will inspect all the files
in the `src` and their permissions to ensure only those files mode are updated
and not any existing `dest` files.

For example, this task will copy the entire directory `some-dir` to
`/u/omvsadm/some-dir` where the directory will be given permissions 644 and
set to group admin and owner omvsadm.

```
- name: Copy directory to managed node
  zos_copy:
    src: /path/to/a/some-dir
    dest: /u/omvsadm
    mode: 0644
    group: admin
    owner: omvsadm
```

For example, this task will copy the directory contents in `some-dir` to
`/u/omvsadm` where the contents will be given permissions 644 and
set to group admin and owner omvsadm. Notice the trailing forward slash `/` that
differentiates this task from the prior.

```
- name: Copy directory conto managed node
  zos_copy:
    src: /path/to/a/some-dir/
    dest: /u/omvsadm
    mode: 0644
    group: admin
    owner: omvsadm
```

<ins>Maintainability</ins>

In the beginning of the blog we noted how the original design had served its
purpose and taking all we learned we decided to redesign the module such that
we can maintain the modules complexity with ease, accuracy and success. Code
should always be well organized, logically isolating, easy to maintain and easy
to enhance.

The `zos_copy` module has a number of copy handlers who's function is to handle
copying data from one type of source to another. For example, we might have a
`pds_to_pdse` or a `uss_to_ps` copy handler, under the new design the copy
handlers have only one purpose, to perform only the copy operation. In the prior
design, the copy handlers were being given the responsibility to decide if the
dest is the right type, adequate in size, exits and even the responsibility to
create the destination. To main this type of logic in each copy handler, it
became complex and now the copy handlers only task is to copy. Recall the
precedence rules, these rules helped in creating a common contract for the entire
module such that no copy handler ever would concern itself on if the destination
is the right type, size etc. Now the decision is made early on in the code on
whether the destination is adequate and if not, the same code silo will create
the destination thus alleviating the copy handlers responsibility.

As you can see, we are constantly reinventing, improving and changing both
the modules outward facing capabilities but internal architecture. We are
continuously expanding our pipelines with linters, scanners, profilers and always
looking to improve and expand our collection.
