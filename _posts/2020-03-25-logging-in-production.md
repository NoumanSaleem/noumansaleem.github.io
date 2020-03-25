---
layout: default
title: Logging in production
category: nodejs
tags:
- nodejs
- javascript
- logging
- 12factor
- stdout
- production
---

Logging is an important responsibility of production applications. Logs provide insight into usage, performance, and aid in debugging issues. When implementing logging, there are a handful of considerations to think through. Some obvious decisions include choosing a log format (e.g JSON or plain text), and determining when and what the application should log.

Another decision, which may seem trivial at first, is deciding how logs are transported outside your application server. If you aren't forwarding your logs to a central location, then I strongly suggest you consider doing so. Leaving application logs on the server makes log viewing inconvenient, or worse, inaccessible should your server crash or become unresponsive.

If you're using a logging library, it probably supports forwarding logs to external systems. For example, [Winston](https://github.com/winstonjs), a popular choice for Node.js, ships with several [built-in transports](https://github.com/winstonjs/winston/blob/master/docs/transports.md#built-in-to-winston) allows saving to disk, sending over HTTP, and writing out to console. While these transports differ in where they transmit logs, both writing to disk and sending over HTTP requires the application process to manage the log lifecycle. The [twelve-factor methodology](https://12factor.net/) recommends avoiding application-managed logs, [stating](https://12factor.net/logs):

> A twelve-factor app never concerns itself with routing or storage of its output stream. It should not attempt to write to or manage logfiles. Instead, each running process writes its event stream, unbuffered, to stdout.

Having previously relied on my application process for log routing, I agree with 12factor having experienced the issues first hand. In this post, we'll evaluate some drawbacks of application-managed logging, and provide guidance for developing a more robust logging architecture.

## Drawbacks of application-managed logging
Application-managed logging, as the name implies, requires the application process to take ownership of log delivery. If you're writing to disk, the application manages buffering messages, writing to file, and rotating logs. For forwarding over HTTP, the application handles batching messages and making network requests.

Imagine a scenario where a new version of the program is being deployed. If you're managing logs in the application, you'll likely want to delay terminating the process until the log queue has finished processing. In Node, you can accomplish this by listening for the `SIGTERM` signal. Once intercepted, your function can wait for the logger to empty before executing `process.exit(0)`. However, what happens if the log queue is too big? Or, worse, if the app is trying to send logs over the network, and the backend is unresponsive?  Now, what if instead of code deployment, the applicatiaon was terminating due to an exception? In that case, waiting for the log queue to empty can delay recovery and potentially leave your application running in a bad state.

Writing to disk or transmitting over the network is not the problem. However, as we've seen, relying on the application process to manage these tasks is not ideal. Let's see how we can build a more resilient system by relying on external processes for log forwarding.

## Logging using STDOUT
Standard output (`stdout`) is simply a stream where application output is written. Many languages ship with a built-in function for writing to stdout, like `print` in Python, or `puts` in Ruby. For Javascript, writing output is done through the `console` global. You may hesitate to use `console.log`, since many linters such as `eslint` warn against it. While scattered use of `console.log` is typically a bad practice, writing or using a logging library will confine use to a single location.

Once our application is writing to stdout, we'll need to redirect it somewhere or we'll lose it. When you execute a command from the terminal, the output is sent to your terminal session by default. To redirect that output to a file, we can use the `>` operator.

```
$ echo 'Hello world!' > output.txt
$ cat output.txt
Hello world!'
```

While we can redirect our application output to disk using this technique, we'd eventually see the logfile grow too large and deplete the disk. What we want, is to prevent our files from growing too large by employing log rotation. In Linux, another way to redirect process output is to use a pipe `|` operator. The pipe operator connects the output of a command to the input of the command that comes after.

The `apache2-utils` package--available on most linux distributions--bundles a binary called `rotatelogs`. [rotatelogs](https://httpd.apache.org/docs/2.4/programs/rotatelogs.html) accepts input from stdin and can be configured to rotate log files based on filesize or a fixed interval. In the snippet below, we're echoing `Hi!` every second, and piping the output to `rotatelogs`. Our `rotatelogs` command is passed the output filename `log` and the number `2` for the time interval.

```bash
$ while true; do echo Hi!; sleep 1; done | rotatelogs log 2
```
If you let this run for a few seconds, you should end up several `log-*` files in the current directory.

Now that our application output is being redirected to disk, we can focus on sending those logs to our central log system. If you're using Elasticsearch to store your logs, then you'll have no issue adopting `filebeat` for log forwarding. `filebeat` is an application written by Elastic.co which can watch a directory of log files and forward them to various backends including Elasticsearch, Logstash, and Redis. Besides `filebeat`, you can leverage the `rsyslog` daemon to perform this task. `rsyslog` is capable of transmitting messages over udp or tcp, and provides built-in outputs for many backends, including `elasticsearch`, `postgres`, and mongodb`.

By reliving our application of log management responsibilities, we've created a more resilient system. Application logs are now safe from app restarts and even system reboots. When the log daemon starts up, it'll read from a state file and continue forwarding messages from where it left off. Also, issues with our downstream logging system won't negatively affect our application performance. Lastly, we can always recover logs by connecting to the server, or mounting its drive on another box and pulling the logs from disk.
