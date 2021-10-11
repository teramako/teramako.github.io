---
title: Ansible keywords
date: 2021-10-11 22:50:00
tags:
  - ansible
---

[Playbook Keywords][playbook_keywords] がちょっと分かり難いので表にまとめてみた。

* Table of Contents
{:toc}

# 共通して使えるもの

|-------------------+-----+-----+------+------|
|                   |Play |Task |Block |Role  |
|-------------------|:---:|:---:|:----:|:----:|
|any_errors_fatal   |○   |○   |○    |○    |
|become             |○   |○   |○    |○    |
|become_flags       |○   |○   |○    |○    |
|become_method      |○   |○   |○    |○    |
|become_user        |○   |○   |○    |○    |
|check_mode         |○   |○   |○    |○    |
|connection         |○   |○   |○    |○    |
|debugger           |○   |○   |○    |○    |
|diff               |○   |○   |○    |○    |
|environment        |○   |○   |○    |○    |
|ignore_errors      |○   |○   |○    |○    |
|ignore_unreachable |○   |○   |○    |○    |
|module_defaults    |○   |○   |○    |○    |
|name               |○   |○   |○    |○    |
|no_log             |○   |○   |○    |○    |
|port               |○   |○   |○    |○    |
|remote_user        |○   |○   |○    |○    |
|run_once           |○   |○   |○    |○    |
|tags               |○   |○   |○    |○    |
|throttle           |○   |○   |○    |○    |
|vars               |○   |○   |○    |○    |
|connections        |○   |○   |○    |○    |
|timeout            |○   |○   |○    |○    |
|-------------------+-----+-----+------+------|

# Play以外で使えるもの

|-------------------+-----+-----+------+------|
|                   |Play |Task |Block |Role  |
|-------------------+:---:+:---:+:----:+:----:|
|delegate_facts     |     |○   |○    |○    |
|delegate_to        |     |○   |○    |○    |
|when               |     |○   |○    |○    |
|-------------------+-----+-----+------+------|

# Playでのみ使えるもの

|-------------------+-----+-----+------+------|
|                   |Play |Task |Block |Role  |
|-------------------+:---:+:---:+:----:+:----:|
|fact_path          |○   |     |      |      |
|force_handlers     |○   |     |      |      |
|gather_facts       |○   |     |      |      |
|gather_subset      |○   |     |      |      |
|gather_timeout     |○   |     |      |      |
|handlers           |○   |     |      |      |
|hosts              |○   |     |      |      |
|max_fail_percentage|○   |     |      |      |
|order              |○   |     |      |      |
|post_tasks         |○   |     |      |      |
|pre_tasks          |○   |     |      |      |
|roles              |○   |     |      |      |
|serial             |○   |     |      |      |
|strategy           |○   |     |      |      |
|tasks              |○   |     |      |      |
|vars_files         |○   |     |      |      |
|vars_prompt        |○   |     |      |      |
|-------------------+-----+-----+------+------|

# Taskでのみ使えるもの

|-------------------+-----+-----+------+------|
|                   |Play |Task |Block |Role  |
|-------------------|:---:|:---:|:----:|:----:|
|action             |     |○   |      |      |
|args               |     |○   |      |      |
|async              |     |○   |      |      |
|changed_when       |     |○   |      |      |
|delay              |     |○   |      |      |
|failed_when        |     |○   |      |      |
|local_actioln      |     |○   |      |      |
|loop               |     |○   |      |      |
|loop_control       |     |○   |      |      |
|notify             |     |○   |      |      |
|poll               |     |○   |      |      |
|register           |     |○   |      |      |
|retries            |     |○   |      |      |
|until              |     |○   |      |      |
|with\_&lt;lookup&gt;|     |○   |      |      |
|-------------------+-----+-----+------+------|

# Blockでのみ使えるもの

|-------------------+-----+-----+------+------|
|                   |Play |Task |Block |Role  |
|-------------------+:---:+:---:+:----:+:----:|
|always             |     |     |○    |      |
|rescue             |     |     |○    |      |
|-------------------+-----+-----+------+------|



[playbook_keywords]: https://docs.ansible.com/ansible/latest/reference_appendices/playbooks_keywords.html "Playbook Keywords - Ansible Documentation"

