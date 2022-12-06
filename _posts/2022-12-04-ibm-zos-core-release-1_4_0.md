---
layout: post
title: Ansible z/OS Core 1.4.0
subtitle: Let's discuss the changes
thumbnail-img: /assets/img/open_source_250_px.png
# share-img: /assets/img/code2.png
# tags: [ansible, zos, ibm_zos_core]
---

Ansible z/OS Core been developing the collections source code in the open and more
recently has moved the entire issue tracking to GitHub for everyone to follow
along.

Lets begin by setting the scope of what it is like and what it means to have an
open source project. Although not a new concept but rather a bit unfamiliar one
for software that runs on z/OS. As the senior technical lead for the IBM z/OS
core collection I hold quite a few responsibilities which started with defining
the project from the ground up. What does that entail?

<ins>Consistency</ins>

To start, you want consistency, although vague, it really applies to everything
your team does. You'll want consistency in how the team interacts with the
repository meaning; which type of workflow will your team adopt. Will you use
a branch strategy or a tag strategy? What coding standards do you want the team
to follow? Will you use snake_case or camelCase, or something else. What branch
naming conventions will you use? What does the development flow look like and how
does code merge into a main branch? How will you group your issues and organize
them into milestones? How will you size items and how many points will you allow
for a sprint? How long will your sprint be? The list goes on but we will focus
on the main points that will help you navigate our repository and interact with
it and the maintainers.

<ins>What is issue tracking software</ins>

Issue tracking software is essentially a program that teams use to manage their
work items. When it comes down to it, an issue is a unit of work that needs to be
completed, that work needs to be tracked over time as its being developed. Issue
tracking allows for that work to have attributes such as the sizing, the description
and when will it be done. In our case, we are using GitHub's project not only
manage our issues but to plan out a quarters worth of work, categorize, organize
and track all the other work items that don't make it into plan. 
