---
layout: post
title: "Rotating Rails Production Logs with LogRotate"
date: 2017-10-23
categories: rails
---

## Rotating Rails Production Logs with LogRotate

#### 参考

#### Keep your log files in check as time goes on and make sure they don't fill up your server's disk space

### Overview

One of those things you have to pay attention to on your production Rails servers is the size of the log files that your Rails application produces. I’ll be showing you how to logrotate Rails production logs each day to archive a copy of your logs and then compress them to save space and make them much more manageable.

Your log files can grow fast, especially if you have a lot of traffic. For example, after about 1 month, my log files are 300MB in size. Forget about that for a while and you could run out of disk space on your server and that would be bad.

### Configuring Logrotate For Rails Production Logs

You might be surprised at just how easy to setup logrotate Rails logs is. The reason it is so handy is that a bunch of your operating system software is already using it. We just have to plug in our configuration and we’re set!

The first step is to open up ** /etc/logrotate.conf ** using ** vim ** or ** nano**. Jump to the bottom of the file an add the following block of code. You’ll want to change the first line to match the location where your Rails app is deployed. Mine is under the ** deploy ** user’s home directory. Make sure to point to the log directory with the ** *.log ** bit on the end so that we rotate all the log files.

```
/home/deploy/APPNAME/current/log/*.log {
  daily
  missingok
  rotate 7
  compress
  delaycompress
  notifempty
  copytruncate
}
```

### How It Works

This is fantastically easy. Each bit of the configuration does the following:

- daily – Rotate the log files each day. You can also use weekly or monthly here instead.
- missingok – If the log file doesn’t exist, ignore it
- rotate 7 – Only keep 7 days of logs around
- compress – GZip the log file on rotation
- delaycompress – Rotate the file one day, then compress it the next day so we can be sure that it won’t interfere with the Rails server
- notifempty – Don’t rotate the file if the logs are empty
- copytruncate – Copy the log file and then empties it. This makes sure that the log file Rails is writing to always exists so you won’t get problems because the file does not actually change. If you don’t use this, you would need to restart your Rails application each time.
