---
title: "Visual Studio Code Server"
tags: 
  - iOS
  - VSCode
  - code-server
---

I'm going to keep this brief as you can find [any number of articles](https://lmgtfy.com/?q=use+ipad+pro+for+coding&s=g&t=w) with similar content and great suggestions for coding on an iPad.  I'm impressed with [code-server](https://github.com/cdr/code-server) overall and think it's a great concept and want to share it with everyone.

I've recently started researching options to use my iPad Pro for coding and have come across a project named [code-server](https://github.com/cdr/code-server) which allows you to run Visual Studio Code on a remote server and access it via a web browser.  This works surprisingly well on an iPad with an external keyboard and a mouse.

Currently most of the solutions for coding with an iPad seem to revolve around using a remote computer which is unfortunate but understandable since you can't run an interpretor on an iPad.  I'm currently writing this post on an iPad Pro using [code-server](https://github.com/cdr/code-server) backed by a Digital Ocean droplet with 1 vCPU and 1 GB of memory for $5 a month.  While this is well below code-server's recommended specifications it is working quiet well for writing the smaller scripts that I use to automate infrastucture.  I also use the same droplet for other random needs which require a full shell.

You can find more information on [setting up code-server here](https://github.com/cdr/code-server/blob/master/README.md).  Feel free to [reach out](https://twitter.com/chrisneely_tech) if you have any questions.

Also, [this blog post](https://medium.com/@ow/its-finally-possible-to-code-web-apps-on-an-ipad-pro-90ad9c1fb59a) is how I found out about code-server and has useful information specifically around using code-server on an iPad.