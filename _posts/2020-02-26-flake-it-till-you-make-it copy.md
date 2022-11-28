---
layout: post
title: Ansible z/OS Core 1.4.0
subtitle: Let's discuss the changes
# cover-img:
thumbnail-img: /assets/img/open_source_250_px.png
# share-img: /assets/img/code2.png
# tags: [ansible, zos, ibm_zos_core]
---

Ansible Core 1.4.0 has been in the making for some time and its finally being
released into Galaxy and Automation Hub. There are too many changes to
discuss in a single blog so I will follow up with a more technical review in
later blogs.

The most significant changes appear in how the collection manages connectivity
over SSH, a re-architected `zos_copy` module, a new `zos_mount` module and a
number deprecated functions and parameters. While that may already seem like quite a bit, 

that
can manage mount operations for a UNIX System Services (USS) file system data set.
While these are 