---
title: Moving away from Alpine
categories: article
tags: dev docker alpine ubuntu
---

When container movement started getting a lot of traction thanks to docker, there was a real demand for lightweight base image that is optimized for single process, unlike your typical OS. Enter Alpine, a lightweight linux distribution as small as 3MB! It also came with good enough [package repository](https://pkgs.alpinelinux.org/packages) which helped a lot with adoption. Unfortunately we have stopped adopting Alpine when possible, due to reasons I will outline.

## Adopting Alpine
At work, we started adopting Alpine pretty early on for development, CI, and even production. All is well, except when it’s not. Turns out it’s a lot of work to get packages that are not readily available in Alpine repository. You see, Alpine uses [musl](http://www.musl-libc.org/) libc instead of [glibc](https://www.gnu.org/software/libc/) and most popular distros use the latter. So things compiled in Alpine won’t be usable on Ubuntu, for example, and vice versa. It’s great when all packages you need are there in the `main` & `community` alpine repositories. When that is not the case, you have to build it on your own and _hope_ that the dependencies are available or at least easy to build as well (against musl).

## Missing package
One day our CI started failing during docker image build phase. `mysql` package (which is just a compatibility package pointing to mariadb) suddenly went missing. We issued the bug report: [Bug #8030: Missing x86_64 architecture for mysql and mysql-client packages in Alpine v3.3 - Alpine Linux - Alpine Linux Development](https://bugs.alpinelinux.org/issues/8030). To Natanael's credit, the issue was resolved within the day, but this issue got us to start questioning things.

## Losing version
I covered this in a [previous post](https://ibnusani.com/article/php-curl-segfault/), it's basically about the difficulty in pinning package versions in Alpine. I would continue but I think [this guy](https://medium.com/@stschindler/the-problem-with-docker-and-alpines-package-pinning-18346593e891) got through the same situation and has the same thoughts. Recommended read.

## When a package is not available
Developers at work needed to use PHP [V8js](http://php.net/manual/en/book.v8js.php) for our experimental branch so I had to get the extension for our alpine-based images. Someone got through most of the trouble for me as he detailed the findings in this Github [gist](https://gist.github.com/tylerchr/15a74b05944cfb90729db6a51265b6c9). Basically we had to compile [GN](https://gn.googlesource.com/gn/), download v8 source, and then build it against musl. Even with the steps laid out, it wasn't a smooth experience. This method is not sustainable for us, considering future updates and patches.

## Syslog limit
Developers rely heavily on app logs via syslog (mounted `/dev/log`) and Alpine uses busybox [syslog](https://wiki.alpinelinux.org/wiki/Syslog) by default. The problem is, messages are truncated at 1024-character limit, which is very small. The root issue is musl has hardcoded limit of 1024 syslog buffer, which is a generous increase from the initial [256](https://git.musl-libc.org/cgit/musl/commit/src/misc/syslog.c?id=3f65494a4cb2544eb16af3fa64a161bd8142f487)(!) limit but still not enough. This doesn't make sense as a default, and I could not find a way to configure this _easily_.

## Final straw
Ubuntu officially launched minimal ubuntu images for cloud / container use around [July](https://blog.ubuntu.com/2018/07/09/minimal-ubuntu-released) last year.

> These images are less than 50% the size of the standard Ubuntu server image, and boot up to 40% faster.

Not only the base image just 29MB in size, but you also get to use all apt packages! All our previous problems with Alpine made it very easy to switch to ubuntu as our base image and we have been satisfied with the switch so far. This in not to say Alpine is not good. It's just not a fit for us.

### Edit: I need to clarify that this post is not about what's the better alternative. Rather, it's about ditching Alpine. There are definitely few good alternatives to ubuntu-minimal. :)
