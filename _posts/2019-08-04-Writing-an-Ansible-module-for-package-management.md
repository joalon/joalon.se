---
layout: post
title: "Writing an Ansible module for package management"
author: "Joakim Lönnegren"
categories: blog
tags: [Ansible, Python]
image:
  feature: 
  teaser: 
---
I really like ansible for configuration management. Lately I've been running some arch linux systems at home and wanted to use ansible to install packages from the AUR (Arch User Repository) but couldn't find a module equivalent to the yum or apt ones for red hat or ubuntu/debian systems.

Here's the basic boiler plate code for a module:
```python

```

Try it out! Put it in a file `test_module.py` in the directory ~/.ansible/plugins/modules/ and run it with `ansible localhost -m test_module -a 'var1=World'`.

From here it's just standard python. I wrote the 'Ansible AUR' - ansur - module. [Check it out](https://github.com/joalon/ansur)

Some gotchas: I got the exception "DeprecationWarning: the imp module is deprecated in favour of importlib" when running `module.run_command(..., cwd=package_path)`. Spent some time on the (Ansible Issues)[https://github.com/ansible/ansible/issues/55213] page before I realised that I passed a posix path from pathlib as cwd (current working directory) instead of a string as the library expected.

