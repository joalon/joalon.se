---
layout: post
title: "Linux Developers Conference Brazil 2019"
author: "Joakim Lönnegren"
categories: blog
tags: [Linux Developers Conference Brazil, Conference, 2019]
image:
  feature: linuxdev-br-2019/feature.jpg
  teaser: linuxdev-br-2019/teaser.jpg
---
Just returned from a great weekend in São Paulo at the [Linux Developers Conference Brazil](https://linuxdev-br.net/ "Linux Developers Conference Brazil"). Attended great talks by really sharp people and made some new friends. Many thanks to the awesome GNU/Linux community in Brazil!

# Friday, workshops at University of São Paulo
I arrived to the São Paulo University campus at 9 in the morning and got my badge. Then followed the workshops, here's the titles and a short recap:

- Development Environment Setup for Linux Kernel Contribution by FLUSP
Clone the kernel, install development tools, build a custom driver and send a formated patch by email. Instructions for the hardest part (setting up email) can be found at the flusp website [Flusp sendmail instructions](https://flusp.ime.usp.br/git/2019/02/15/sending-patches-by-email-with-git/).

- SystemTap: crossing the line between the userspace and the kernel by Jõao Avelino Bellomo Filho
Build SystemTap probes to show events inside and outside the kernel. I found an older set of the slides used here: [SystemTap slides](https://www.slideshare.net/tchelinux/introduo-ao-systemtap-joo-avelino-bellomo-filho-tchelinux-caxias-2018)

In the evening there was a Happy hour at Ampere bar sponsored by Pantacore. Loud music and friendly people!

# Saturday, talks at Radisson Paulista Plaza
First day at Radisson Paulista Plaza on the Paulista avenue. Great place but a little crowded during the breaks. The opening of the conference was in portuguese. Sorry, my portuguese wasn't up to par so I didn't make many notes. Here is the names of the talks:

- Introducing ICTL: Instituto de Conservação de Tecnologias Livres. By Antonio Terceiro
- The Contemporary value of OPEN. By Klaus Heinrich Kiwi 
- Linux Graphics Revolution. By Gustavo Padovan
- Top 5 Reasons for Accelerating Your Cloud Native DevOps with cutting edge Open Source Solutions. By Alexandre Sousa

After the opening I attended the following talks, this time in english:

- Performance: Not just about speed any more. By John "Maddog" Hall  
The "Maddog" gave his take on performance - "To get a result in time". Great talk which inspired me so much I might have to take up assembly again.

- BPF is eating the world, don't you see? By Arnaldo Carvalho de Melo  
The eBPF (extended Berkeley Packet Filter) is gaining ground, both in debugging and packet filtering. 

- How to write your own KVM client from scratch. By Murilo Opsfelder Araújo  
A beginners guide to creating your own virtual machines in kvm via the C client interface. An explanation of the following code [lwn.net article](https://lwn.net/Articles/658512/ "lwn.net/Articles/658512")

- CPU vulnerabilities and KVM security. By Eduardo Habkost  
Some recaps on bugs like spectre and meltdown and how they affect KVM virtual machines.

- Leveraging OP-TEE as a generic HSM via PKCS#11 for secure OTA. By Ricardo Salveti  
A security talk on using the Open Portable Trusted Execution Environment [OP-TEE](https://www.op-tee.org/) and a hardware module for ensuring application authenticity. 

- Malicious Linux Binaries: A Landscape. By Lucas Galante and Marcus Botacin  
Great summary of linux malware. The big takeway was: Linux malware exists and should be taken seriously.

After the event there was a happy hour at Pizzaria Urca sponsored by Google. Despite the mild weather there was a warm atmosphere!

# Sunday, talks at Radisson Paulista Plaza continued
Great talks on sunday! I attended:

- Performance on virtual machines: passing through PCI devices. By Leonardo Garcia  
Some design considerations and recommendations for how to pass through PCI devices to virtual clients, as well as some inofficial benchmarks. Use "host-passthrough" in libvirt when you can.

- Libpulp: we patched user space live patching. By Jõao Moreira  
A talk from the OpenSUSE Labs where they have developed live patching of running libraries in user space. Real interesting in my work as a sysadmin.

- V4L2: A Status Update. By Hans Verkuil  
An update from the Video for Linux subsystem maintainer.

- Object-Oriented Techniques in C: A Case Study on Git and Linux. By Renato Geh and Matheus Tavares  
Two interesting case studies in the git and kernel code bases of using C language features to implement constructs that look object-oriented. For example inheritance, public/private members and polymorphism.

- Graphics: An overview of DRM/KMS kernel API, main concepts and some caveats. By Helen Koike (In portuguese)  
Some showcases on rendering and the different issues you can run into. Cool demos!

- How bias impacts what we develop. By Alda Rocha  
A recap on recent developments where sex and race shouldn't have been an issue but actually became one. Really strong ending!

Hoping to post links to the talks when they are accessible online.

# Conclusion
It was my first time visiting São Paulo city despite living in the same state for half a year. Beautiful city and a very pleasant community, I will definitely be returning soon!

Big thanks to Gustavo Padovan and the linuxdev-br committee for arranging an awesome conference.
