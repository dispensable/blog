---
layout: post
title: Avoid mix use nginx ssi directive and lua-resty-module
tags:
- nginx
---

I have a simple Nginx server with some SSI directives to generate common parts of the web pages. I use the lua-resty-module for NGINX to handle precheck auth logic, and it has been working fine for a long time. However, one day, the Nginx host sent me an alert about a disk full error. After SSHing to the host and running ncdu, I found that the disk was filled with coredump files.

There must be a bug in the NGINX config. I opened the error log of Nginx but couldn't find any errors that made sense because the worker process just SIGFAULTed and didn't have chance to write a error log.

The Nginx debug [page](https://docs.nginx.com/nginx/admin-guide/monitoring/debugging/) shows how to use debug tools to analyze the coredump file. However, my Nginx package was installed from the package manager, and it stripped the debug info, so GDB mostly showed `??????`.

Luckily, building a debug Nginx is not difficult. After enabling debugging with `--with-debug --with-cc-opt='-O0 -g -DDDEBUG=1'` and rebuild/install/restarting the Nginx. It still kept coredumping after serving some requests. I used GDB to debug the coredump file, and the stack traceback showed a failure at the SSI directive. Although this was an old configuration, I still deleted this configuration and restarted Nginx to see if it would still crash. The crashes stopped, indicating that something was happening when the SSI directive was running.

With the stack trace from the coredump file, I could identify the request URI. However, when I tried this URI, it returned a 200 success. Even with the SSI version of the configuration, it still return 200 with out segfault. There must have been some input that triggered this crash. After going through the log file with this URI and consulting with others, they pointed out that the cookie part of the logs (extracted with Lua logic) seemed strange. It used semicolons to separate the cookie parts, but my code couldn't construct a cookie data like this. It must have been a spider that was implemented roughly enough to construct a wrong cookie and triggered this bug.

With the help of my friends, I finally reproduced the bug. Requesting the URI with the wrong cookie format would immediately kill the NGINX worker.

I tried to understand why this was happening. I adjusted the logic of the Lua code to check the cookie, but it didn't help with the bug. Disabling SSI include in this position (while keeping others) also didn't help with the bug. Rearranging the Lua logic after the SSI (where SSI is implemented as an NGINX subrequest that will go through the Lua logic whenever the SSI is executed) stopped the crash. Putting the logic in another position caused the NGINX worker to fail. It was really confusing.

Fortunately, this bug is reproducible. I enabled a more verbose debug level to see what was happening in the Nginx logic. I finally confirmed that it was the Lua and the SSI directive causing the issue. If I made the subrequest by SSI bypass the Lua logic, it would be fixed. Any other logic adjustment would cause the NGINX worker to fail.

The only option left for me was to read the lua-resty-module source code, and I found this [issue](https://github.com/openresty/lua-resty-redis/issues/23). You just CANNOT mix the NGINX SSI directive and Lua logic. It may work (like the normal request) or not work (like the accidentally constructed request header). That's why my fix may work or may not work confusingly.

RTFM!
