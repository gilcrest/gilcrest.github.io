---
title: "My Path to node.js and Oracle Glory - Volume 3"
date: 2015-03-16T14:02:00-04:00
description: node.js setup on ubuntu on chromebook
menu:
  sidebar:
    name: Volume 3
    identifier: volume-3
    parent: node-oracle-glory
    weight: 70
---


Next step - node.js setup on Ubuntu.  This was remarkably easy!  I went to [https://nodejs.org/][url-1] and viewed their Downloads section, where I then went to "Installing from package managers".  That link takes you to this [URL][url-2] on GitHub.  If you read the first line of "Debian and Ubuntu based Linux distributions", you'll note you should go to [nodesource's blog post][url-3] for the most up to date instructions.  I did what I was told - read what they have to say on the page - helpful stuff!

The instructions to install are as follows:

Open a new crosh window or use ctrl+alt+t and type:

```bash
shell
```

To get into Ubuntu, type:

```bash
sudo startxfce4
```

Inside Ubuntu, click on the Terminal Emulator icon

You now need curl (a tool for making http calls, etc. - [http://en.wikipedia.org/wiki/CURL][curl]), so get it - type:

```bash
sudo apt-get install curl
```

The instructions from nodesource's blog post say to then enter the following:

```bash
# Note the new setup script name for Node.js v0.12
curl -sL https://deb.nodesource.com/setup_0.12 | sudo bash -
```

```bash
# Then install with:
sudo apt-get install -y nodejs
```

Type node to start node..

That's it!  node.js is installed in Ubuntu inside my chromebook!

There are a *TON* of node.js intros, etc. that I've been pouring over for a while now - I'm not going to bother to explain the basics of node as it's extremely well documented.  I will now start to write some small pl/sql and js programs using the node-oracledb driver and will continue to blog as things progress in that area...  That's it for today!

[url-1]:   https://nodejs.org/
[url-2]:   https://github.com/joyent/node/wiki/Installing-Node.js-via-package-manager
[url-3]:   https://nodesource.com/blog/nodejs-v012-iojs-and-the-nodesource-linux-repositories
[curl]:    http://en.wikipedia.org/wiki/CURL
