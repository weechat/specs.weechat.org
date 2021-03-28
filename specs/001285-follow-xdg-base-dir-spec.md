# Follow XDG Base Directory Specifications

- Author: [Sébastien Helleu](https://github.com/flashcode)
- Created on: 2021-03-28
- Last updated: 2021-03-28
- Issue: [#1285](https://github.com/weechat/weechat/issues/1285)
- Status: draft
- Target WeeChat version: to define

## Context

Historically, WeeChat has always defined and used a "WeeChat home" which is
the directory where WeeChat stores all it files (configuration, logs, etc.)
and allows plugins/scripts to store files there as well.

This directory is `$HOME/.weechat` by default and can be forced by 3 ways,
by order or priority (the first one has higher priority):

- at runtime, with two command line options: `-d`/`--dir` or `-t`/`--temp-dir`
- at runtime, with environment variable `WEECHAT_HOME`
- at compilation time, so that WeeChat uses another directory by default.

## Goals

Purpose of this specification is to describe how to "smoothly" follow the
XDG specifications, without breaking changes for the existing WeeChat installations.

The main goal is to separate the different type of data in 4 directories:
config, data, cache, runtime.

This way the different files are properly saved in appropriate directories
and there's no more `.weechat` directory in the user's home directory.

The latest XDG Base Directory Specification can be found at:
[https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)

## Out of scope

### System-wide configuration files

Another related feature request is to be able to read system-wide WeeChat
configuration files at startup (if no local configuration is found), and write
them as usual in local directory.

This can be implemented later, but we have to keep this future improvement in
mind, to make it easier to happen soon.

### $XDG_CONFIG_DIRS and $XDG_DATA_DIRS

Variables `$XDG_CONFIG_DIRS` and `$XDG_DATA_DIRS` (as defined in the
XDG Base Directory Specification) are not yet used by this first implementation.

## Implementation

### WeeChat home

The WeeChat home, internal variable `weechat_home` must be split into 4 directories:

- config: `$XDG_CONFIG_HOME/weechat`, defaulting to `$HOME/.config/weechat`
  if `$XDG_CONFIG_HOME` is not defined or empty
- data: `$XDG_DATA_HOME/weechat`, defaulting to `$HOME/.local/share/weechat`
  if `$XDG_DATA_HOME` is not defined or empty
- cache: `$XDG_CACHE_HOME/weechat`, defaulting to `$HOME/.cache/weechat`
  if `$XDG_CACHE_HOME` is not defined or empty
- runtime: `$XDG_RUNTIME_DIR/weechat`, defaulting to the cache directory
  if `$XDG_RUNTIME_DIR` is not defined or empty.

WeeChat stores the following data in each directory:

- config:
  - configuration files: `*.conf`
  - scripts configuration (depends on scripts)
- data:
  - WeeChat log file (and crash log): `weechat*.log`
  - WeeChat upgrade files: `*.upgrade`
  - local plugins: `plugins/*`
  - logs files written by logger plugin: `logs/*`
  - Xfer files received: `xfer/*`
  - scripts installed: `python/*`, `perl/*`, etc.
  - scripts data (depends on scripts)
- cache:
  - scripts repository contents: `script/plugins.xml.gz`
  - script downloaded (temporary, during installation)
  - scripts cache (depends on scripts)
- runtime:
  - FIFO pipe: `weechat_fifo`
  - scripts runtime (depends on scripts).

For compatibility, all functions using WeeChat home are now using `data`
directory, unless they explicitly mention another directory.

### Determining WeeChat directories

The new behavior is described below, for each step not verified, we continue
with the following step:

1. If a command-line argument is given ('`-d`, `--dir`, `-t` or `--temp-dir`),
   use it as WeeChat home (for the 4 directories).
1. If the environment variable `WEECHAT_HOME` is defined,
   use it as WeeChat home (for the 4 directories).
1. If the compilation option with WeeChat home is set,
   use it as WeeChat home (for the 4 directories).
1. If `weechat.conf` exists in XDG `config` directory,
   use XDG directories.
1. If `weechat.conf` exists in WeeChat home (`$HOME/.weechat` by default),
   use it as WeeChat home (for the 4 directories).
1. By default, if no config is found anywhere,
   use XDG directories.

Once the directories have been determined, as usual on startup, the directories
are created if not existing, and default configuration files are created if
they are not found.

### Compilation options

The WeeChat home can be forced at compilation time, now the default value is
an empty string instead of `$HOME/.weechat`:

- CMake:
  - `-DWEECHAT_HOME=xxx`
- autotools:
  - environment variable `WEECHAT_HOME`

### Environment variables

The following environment variables are used by WeeChat, there are no changes:

- `WEECHAT_HOME`: force a WeeChat home
- `WEECHAT_ExTRA_LIBDIR`: directory with plugins

### Command line options

The two command line options are kept unchanged, if they are used, the XDG
directories are completely ignored and the 4 new directories variables point
to the same directory:

- `-d` / `--dir`: forces WeeChat home
- `-t` / `--temp-dir`: forces WeeChat to use a temporary home directory, deleted
  upon exit

### Functions using WeeChat home

#### string_eval_path_home

This function replaces, among others, `%h` by the WeeChat home.
Now that we have 4 different directories, the `%h` may be replaced by one of
these directories:

- config directory
- data directory
- cache directory
- runtime directory

The hashtable options is used to specify another directory if needed, with a key
`directory` and one of these values:

- `config`
- `data` (default if the key is not present)
- `cache`
- `runtime`

The following scripts are calling this function and must be changed if the
desired directory is not the `data` one:

Script            | Directory | Update needed
----------------- | --------- | -------------
bufsave.py        | data      | No
latex\_unicode.py | cache     | **Yes**
slack.py          | data      | No

#### mkdir_home

This function creates a directory in the WeeChat home.
By default, the directory is now created in the `data` directory.

Support of other directories is supported in the `directory` argument, using
a prefix between curly brackets:

```C
weechat_mkdir_home ("test", 0755);  /* data directory by default */
weechat_mkdir_home ("{config}/test", 0755);
weechat_mkdir_home ("{data}/test", 0755);
weechat_mkdir_home ("{cache}/test", 0755);
weechat_mkdir_home ("{runtime}/test", 0755);
```

The following scripts are calling this function and must be changed if the
desired directory is not the `data` one:

Script    | Type   | Update needed
--------- | ------ | -------------
luanma.pl | config | **Yes**
otr.py    | data   | No

#### info_get

The info `weechat_dir` is kept for compatibility and now points to the `data`
directory.

It is marked as `deprecated` in the API documentation and must not be used any more.

New info are added:

- `weechat_config_dir`: the config directory
- `weechat_data_dir`: the data directory
- `weechat_cache_dir`: the cache directory
- `weechat_runtime_dir`: the runtime directory

If the legacy home directory is in use, the 4 info return the same directory.

They are available in evaluation of expressions, for example:
`${info:weechat_config_dir}`.

The following scripts are using the `weechat_dir` info and must be changed if the
desired directory is not the `data` one:

Script              | Directory | Update needed
------------------- | --------- | -------------
buddylist.pl        | ?         | ?
colorize\_lines.pl  | ?         | ?
hotlist2extern.pl   | ?         | ?
jnotify.pl          | ?         | ?
luanma.pl           | ?         | ?
pop3\_mail.pl       | ?         | ?
query\_blocker.pl   | ?         | ?
rslap.pl            | ?         | ?
rssagg.pl           | ?         | ?
stalker.pl          | ?         | ?
autoconf.py         | ?         | ?
axolotl.py          | ?         | ?
beinc.py            | ?         | ?
chanop.py           | ?         | ?
chanstat.py         | ?         | ?
confversion.py      | ?         | ?
country.py          | ?         | ?
cron.py             | ?         | ?
crypt.py            | ?         | ?
grep.py             | ?         | ?
growl.py            | ?         | ?
histman.py          | ?         | ?
hl2file.py          | ?         | ?
otr.py              | ?         | ?
purgelogs.py        | ?         | ?
queryman.py         | ?         | ?
queue.py            | ?         | ?
slack.py            | ?         | ?
triggerreply.py     | ?         | ?
update\_notifier.py | ?         | ?
url\_olde.py        | ?         | ?
urlserver.py        | ?         | ?
weeget.py           | ?         | ?
weetext.py          | ?         | ?
zncplayback.py      | ?         | ?
substitution.rb     | ?         | ?

### Options using WeeChat home

The following options are referencing WeeChat home with `%h`, they are updated
to use the new directory:

Option                       | Default value      | Directory
---------------------------- | ------------------ | ---------
fifo.file.path               | `%h/weechat_fifo`  | runtime
logger.file.path             | `%h/logs/`         | data
relay.network.ssl\_cert\_key | `%h/ssl/relay.pem` | ?
script.scripts.path          | `%h/script`        | cache
weechat.plugin.path          | `%h/plugins`       | data
xfer.file.download\_path     | `%h/xfer`          | data
