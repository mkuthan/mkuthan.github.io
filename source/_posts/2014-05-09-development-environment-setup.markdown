---
layout: post
title: "Development Environment Setup"
date: 2014-05-09
categories: linux bash ruby java node.js
---

This document is a manual how to configure flexible development environment for _Java_, _JavaScript_ and _Ruby_.
Even if the runtimes installation with `apt-get` is trivial task, there is limited control over installed version.
The goal is to configure environment where you can easily change _Java_, _Ruby_ or _node.js_ versions.

The most convenient way to configure and manage runtimes is to use environment manager.
Environment manager is nothing more than shell script, the script intercepts executed commands using shim executables injected into your `PATH`.
There are two flavours of the environment managers: `rvm` and `rbenv` like.
I prefer the second one, it is less obtrusive and follows general unix  principle: "do one thing and do it well".

Let's start and install environment managers (for _Java_, _Ruby_ and _node.js_) into your home directory:

``` console
git clone https://github.com/gcuisinier/jenv.git ~/.jenv
git clone https://github.com/sstephenson/rbenv.git ~/.rbenv
git clone https://github.com/OiNutter/nodenv.git ~/.nodenv
```

For `rbenv` and `nodenv` you can install plugins that provide `rbenv install` and `nodenv install` commands to compile and install runtimes automatically.

``` console
git clone https://github.com/sstephenson/ruby-build.git ~/.rbenv/plugins/ruby-build
git clone https://github.com/OiNutter/node-build.git ~/.nodenv/plugins/node-build
```

Add environment managers to the `PATH` and initialize them to get command auto completion.
Append the following snippet at the end of `.bashrc` file.

``` bash
export PATH="$HOME/.jenv/bin:$PATH"
eval "$(jenv init -)"
 
export PATH="$HOME/.rbenv/bin:$PATH"
eval "$(rbenv init -)"
 
export PATH="$HOME/.nodenv/bin:$PATH"
eval "$(nodenv init -)"
```

Install runtimes using environment managers (Java needs to be installed manually):

``` console
jenv add /path/to/installed/jdk
rbenv install 1.9.3-p448
nodenv install 0.10.12
```

Install build tools (_maven_, _gradle_, _sbt_, etc.), create symbolic links, and configure path in `.profile` file:

``` bash
APPS="$HOME/apps"
export PATH="$APPS/apache-maven/bin:$APPS/gradle/bin:$APPS/sbt/bin:$PATH"
```

Make build tools _jenv_ aware:

``` console
jenv enable-plugin maven
jenv enable-plugin gradle
jenv enable-plugin sbt
```

Finally add shell helper functions for JVM configuration to `.profile` file:

``` bash
function jdebug_set() {
    jenv shell-options "$JENV_OPTIONS -Xdebug -Xrunjdwp:server=y,transport=dt_socket,address=8000,suspend=n"
}

function jdebug_unset() {
    jenv shell-options --unset
}

function gc_set() {
    jenv shell-options "$JENV_OPTIONS -XX:+PrintGCDetails -Xloggc:gc.log"
}

function gc_unset() {
    jenv shell-options --unset
}

function jrebel_set() {
    jenv shell-options "$JENV_OPTIONS -javaagent:$APPS/jrebel/jrebel.jar -noverify"
}

function jrebel_unset() {
    jenv shell-options --unset
}

function jprofiler_set() {
    jenv shell-options "$JENV_OPTIONS -javaagent:$APPS/jprofiler/bin/agent.jar"
}

function jprofiler_unset() {
    jenv shell-options --unset
}
```
