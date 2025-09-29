# Week 01 Checkpoint

Class: CS631

Subject: # Advanced Programming in the UNIX Environment

Date: 2025-09-13

Teacher: Jan Schaumann

1. Briefly describe your experience level using Unix-like OS.

	- I have been using Unix-like OS's about 7 years, mostly on mainstream Linux derivatives such as Ubuntu-LTS, Kubuntu, Arch Linux, and Fedora 36. As of right now, besides Linux virtual machines, my main Unix-based OS is simply MacOS which is what I mostly use for development and schoolwork.


2. Briefly describe your programming experience (languages you know, skill level, group experience, project size in KLOC, etc)

	- The main programming languages that I know well and have used in both an academic and professional setting have been C# (Advanced), C/C++ (Intermediate), Python (Advanced), Java (Advanced), and JavaScript (Intermediate). I have a large amount of group experience in both from my internship work and from most of my Computer Science classes. My academic group experience is mostly when we need to code a group project, and I tend to have the best results for my group projects when I am given the chance to pick my own people for the group. I have worked on several (30+) small and large projects, my largest projects was about ~9-10 thousand lines of code consisting of mostly Python and C++ that took about half a year to do, my smallest has been a couple hundred.

3. Get your NetBSD VirtualBox or UTM virtual machine set up. Make sure to fetch and extract the CS631 APUE code examples as well as the NetBSD source code. Comment on anything that you found noteworthy.

	- The most note worthy thing I found was after completing the SSH setup for NetBSD on Mac, I can easily copy files to and from the virtual machine via my shell using the SCP command, which is what I see as the most straight-forward and fastest way to quickly download files that I would need to submit (i.e homework assignments, project, etc). Following the tutorial (both the video and documentation) allowed me to automate a lot of the workflow of using the UTM virtual machine through simple scripts. The script provided in the documentation allowed me to start the UTM VM from my shell and the automatic port forwarding is convenient, and the script showcased in the video to automatically shutdown the VM made automating shutdown of the VM trivial.

4. Make sure to run all the code examples from the Week 01 videos. If you have any questions, note them here:

	- Will we be using a makefile primarily for this class? Running "cc" works just fine for the code examples, my main concern is that if we do have an assignment or project that would require us to code in multiple files, or homework, it might be easier to use a makefile instead.

5. Please note any particular (sub)topics or aspects of this week's materials that you would like me to revisit in class:

	- Solaris, BSD, and Linux? From my understanding, compiler wise, Solaris from Oracle has its own Sun C compiler, while Linux and NetBSD uses gcc.