# Follow XDG Base Directory Specifications

- Author: [Sébastien Helleu](https://github.com/flashcode)
- Created on: 2021-03-28
- Last updated: 2021-05-01
- Issue: [#1285](https://github.com/weechat/weechat/issues/1285)
- Status: development in progress
- Target WeeChat version: 3.2

## Context

Historically, WeeChat has always defined and used a "WeeChat home" which is
the directory where WeeChat stores all it files (configuration, logs, etc.)
and allows plugins/scripts to store their files as well.

This directory is `$HOME/.weechat` by default and can be forced in 3 ways,
by order or priority (the first one has higher priority):

- at runtime, with two command line options: `-d` / `--dir` or `-t` / `--temp-dir`
- at runtime, with environment variable `WEECHAT_HOME`
- at compilation time, so that WeeChat uses another directory by default.

## Goals

Purpose of this specification is to describe how to "smoothly" follow the
XDG specifications, without breaking changes for the existing WeeChat installations.

The main goal is to separate the different type of data in 4 directories:

- config directory
- data directory
- cache directory
- runtime directory.

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

These variables may be used later, when WeeChat will be able to read configuration
and load scripts from system directories.

## Implementation

### WeeChat home

The WeeChat home (internal variable `weechat_home`) must be split into 4 variables:

Variable              | Default value              | Fallback value
--------------------- | -------------------------- | ------------------------------------------------------------------------
`weechat_config_dir`  | `$XDG_CONFIG_HOME/weechat` | `$HOME/.config/weechat` if `$XDG_CONFIG_HOME` is not defined or empty
`weechat_data_dir`    | `$XDG_DATA_HOME/weechat`   | `$HOME/.local/share/weechat` if `$XDG_DATA_HOME` is not defined or empty
`weechat_cache_dir`   | `$XDG_CACHE_HOME/weechat`  | `$HOME/.cache/weechat` if `$XDG_CACHE_HOME` is not defined or empty
`weechat_runtime_dir` | `$XDG_RUNTIME_DIR/weechat` | same as cache directory if `$XDG_RUNTIME_DIR` is not defined or empty

These directories can be forced by the user in different ways, see the next chapters
for more information.

The following data is stored by WeeChat or the user in each directory:

- config:
  - configuration files: `*.conf`
  - IRC: ECC private keys for SASL mechanism "ecdsa-nist256p-challenge"
  - IRC: SSL certificate files used to automatically identify with a nick
  - relay: SSL certificates and private keys (for serving clients with SSL)
  - certificate authorities
  - scripts configuration (depends on scripts)
- data:
  - WeeChat log file (and crash log): `weechat*.log`
  - WeeChat upgrade files: `*.upgrade`
  - local plugins: `plugins/*`
  - logs files written by logger plugin: `logs/*`
  - xfer files received: `xfer/*`
  - scripts installed: `python/*`, `perl/*`, etc.
  - scripts data (depends on scripts)
- cache:
  - scripts repository contents: `script/plugins.xml.gz`
  - script downloaded (temporary, during installation)
  - scripts cache (depends on scripts)
- runtime:
  - FIFO pipe: `weechat_fifo`
  - relay UNIX sockets
  - scripts runtime (depends on scripts).

For compatibility, all functions using WeeChat home are now using `data`
directory by default, unless the caller explicitly mention another directory.

### Compilation options

The WeeChat home can be forced at compilation time, now the default value is
an empty string instead of `~/.weechat`:

- CMake:
  - `-DWEECHAT_HOME=xxx`
- autotools:
  - environment variable `WEECHAT_HOME`

In addition, the value can now be either one directory (to use it for the 4
directories) or 4 directories separated by colons, to force a separate value
for each directory, in this order: config, data, cache, runtime.

Note: forcing 4 different directories is not recommended, better let WeeChat
automatically find them according to XDG environment variables
(see [Determining WeeChat directories](#determining-weechat-directories)).

### Environment variables

The following environment variables are used by WeeChat:

- `WEECHAT_HOME`: force WeeChat home; it can be either one or 4 directories
  separated by colons (in this order: config, data, cache, runtime)
- `WEECHAT_EXTRA_LIBDIR`: directory with plugins (no changes).

### Command line options

The two command line options are used to force directories:

- `-d` / `--dir`: force WeeChat home; it can be either one or 4 directories
  separated by colons (in this order: config, data, cache, runtime)
- `-t` / `--temp-dir`: force WeeChat to use a temporary home directory, deleted
  upon exit.

Note: forcing 4 different directories with `-d` / `--dir` is not recommended,
it is used internally by the `/upgrade` command to be sure the new binary
executed uses exactly the same directories
(see impacts on [function command_upgrade](#command_upgrade)).

### Determining WeeChat directories

The new behavior is described below, for each step not verified, we continue
with the next step:

1. If a command-line argument `-t` / `--temp-dir` is given,
   create a temporary directory and use it as WeeChat home for the 4 directories.
2. If a command-line argument `-d` / `--dir` is given,
   use it as WeeChat home (one or 4 directories).
3. If the environment variable `WEECHAT_HOME` is defined,
   use it as WeeChat home (one or 4 directories).
4. If the compilation option with WeeChat home is set,
   use it as WeeChat home (one or 4 directories).
5. If `weechat.conf` exists in XDG `config` directory,
   use XDG directories.
6. If `weechat.conf` exists in directory `$HOME/.weechat`,
   use it as WeeChat home (for the 4 directories);
   this is for compatibility with old releases not supporting XDG directories.
7. By default, if no config is found anywhere,
   use XDG directories.

When no home is forced and WeeChat is executed for the first time, the XDG
directories are used by default (last step).

Once the directories have been determined, as usual on startup, the directories
are created if not existing, and default configuration files are created if
they are not found.

### Impacted functions

#### string_eval_expression

The 4 directories are replaced by default in evaluation of all strings:

- `${weechat_config_dir}`: config directory
- `${weechat_data_dir}`: data directory
- `${weechat_cache_dir}`: cache directory
- `${weechat_runtime_dir}`: runtime directory.

Note: if such variable name is also given in the hashtable `pointers` or `extra_vars`,
this variable is used in priority (the WeeChat directory is not used).

#### string_eval_path_home

This function replaces, among others, `%h` by the WeeChat home. This `%h`
becomes deprecated and should not be used any more because its name is ambiguous.

Instead, the 4 new variables added in `string_eval_expression` must be used.

For compatibility, if the `%h` is still used (for example in the value of an option),
the directory can be forced with the hashtable `options`, using a key
`directory` which can have one of these values:

- `config`
- `data` (default if the key is not present)
- `cache`
- `runtime`.

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

Support of other directories is supported in the `directory` argument:

- `${weechat_config_dir}`: config directory
- `${weechat_data_dir}`: data directory
- `${weechat_cache_dir}`: cache directory
- `${weechat_runtime_dir}`: runtime directory.

Note: the syntax looks similar to an evaluated expression, but the path is
**NOT** evaluated.

If no prefix is present, the `data` directory is used.

Examples:

```C
weechat_mkdir_home ("test", 0755);  /* data directory by default */
weechat_mkdir_home ("${weechat_config_dir}/test", 0755);
weechat_mkdir_home ("${weechat_data_dir}/test", 0755);
weechat_mkdir_home ("${weechat_cache_dir}/test", 0755);
weechat_mkdir_home ("${weechat_runtime_dir}/test", 0755);
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

- `weechat_config_dir`: config directory
- `weechat_data_dir`: data directory
- `weechat_cache_dir`: cache directory
- `weechat_runtime_dir`: runtime directory.

They are available in evaluation of expressions, for example:

```
/eval -n ${info:weechat_config_dir}
== [/home/user/.config/weechat]
```

If the legacy home directory is in use, the 4 info return the same directory.

The following scripts are using the `weechat_dir` info and must be changed if
the desired directory is not the `data` one:

Script              | Directory      | Update needed
------------------- | -------------- | -------------
autoconf.py         | config         | **Yes**
axolotl.py          | data           | No
beinc.py            | config         | **Yes**
buddylist.pl        | config         | **Yes**
chanop.py           | data           | No
chanstat.py         | data           | No
colorize\_lines.pl  | config         | **Yes**
confversion.py      | config         | **Yes**
country.py          | cache          | **Yes**
cron.py             | config         | **Yes**
crypt.py            | data           | No
grep.py             | data           | No
growl.py            | data           | No
histman.py          | data           | No
hl2file.py          | data           | No
hotlist2extern.pl   | data           | No
jnotify.pl          | config         | **Yes**
luanma.pl           | config         | **Yes**
otr.py              | data           | No
pop3\_mail.pl       | config         | **Yes**
purgelogs.py        | data           | No
query\_blocker.pl   | config         | **Yes**
queryman.py         | data           | No
queue.py            | data           | No
rslap.pl            | data           | No
rssagg.pl           | config + cache | **Yes**
slack.py            | data           | No
stalker.pl          | data           | No
substitution.rb     | config         | **Yes**
triggerreply.py     | data           | No
update\_notifier.py | cache          | **Yes**
url\_olde.py        | data           | No
urlserver.py        | data           | No
weetext.py          | data           | No
zncplayback.py      | data           | No

Note: all these scripts should be updated anyway, even if using the data
directory, to replace info `weechat_dir` with the new one (with a condition on
the WeeChat version).

#### command_upgrade

The `/upgrade` command uses `--dir` command line argument to force the WeeChat
home.

The 4 directories are now given in the `--dir` option (in this order: config,
data, cache, runtime).

#### completion_list_add_filename_cb

There is a fallback on `weechat_home` which must be changed to `weechat_data_dir`.

#### config_file_read_internal

The directory `weechat_home` must be replaced by `weechat_config_dir`.

#### config_file_write_internal

The directory `weechat_home` must be replaced by `weechat_config_dir`.

#### debug_sigsegv_cb

The directory `weechat_home` must be replaced by `weechat_data_dir`.

#### debug_directories

The function must be updated to display the 4 different directories.

The current output of command `/debug dirs` in WeeChat is:

```
Directories:
  home: /home/user/.weechat
        (default: ~/.weechat)
  lib: /usr/lib/x86_64-linux-gnu/weechat
  lib (extra): -
  share: /usr/share/weechat
  locale: /usr/share/locale
```

With XDG directories, the new output must be:

```
Directories:
  home:
    config: /home/user/.config/weechat
    data: /home/user/.local/share/weechat
    cache: /home/user/.cache/weechat
    runtime: /run/user/1000/weechat
  lib: /usr/lib/x86_64-linux-gnu/weechat
  lib (extra): -
  share: /usr/share/weechat
  locale: /usr/share/locale
```

With a single forced home directory, for example `--dir ~/.weechat-dev`:

```
Directories:
  home:
    config: /home/user/.weechat-dev
    data: /home/user/.weechat-dev
    cache: /home/user/.weechat-dev
    runtime: /home/user/.weechat-dev
  lib: /usr/lib/x86_64-linux-gnu/weechat
  lib (extra): -
  share: /usr/share/weechat
  locale: /usr/share/locale
```

With a temporary home directory, using `--temp-dir`:

```
Directories:
  home:
    config: /tmp/weechat_temp_2RWTXJ (TEMPORARY, deleted on exit)
    data: /tmp/weechat_temp_2RWTXJ (TEMPORARY, deleted on exit)
    cache: /tmp/weechat_temp_2RWTXJ (TEMPORARY, deleted on exit)
    runtime: /tmp/weechat_temp_2RWTXJ (TEMPORARY, deleted on exit)
  lib: /usr/lib/x86_64-linux-gnu/weechat
  lib (extra): -
  share: /usr/share/weechat
  locale: /usr/share/locale
```

#### log_open

The directory `weechat_home` must be replaced by `weechat_data_dir`.

#### log_crash_rename

The directory `weechat_home` must be replaced by `weechat_data_dir`.

#### upgrade_file_new

The directory `weechat_home` must be replaced by `weechat_data_dir`.

#### upgrade_weechat_end

The directory `weechat_home` must be replaced by `weechat_data_dir`.

#### util_search_full_lib_name_ext

The directory `weechat_home` must be replaced by `weechat_data_dir`.

#### plugin_script_auto_load

The info `weechat_dir` must be replaced by `weechat_data_dir`.

#### plugin_script_search_path

The info `weechat_dir` must be replaced by `weechat_data_dir`.

#### plugin_script_action_install

The info `weechat_dir` must be replaced by `weechat_data_dir`.

#### plugin_script_action_autoload

The info `weechat_dir` must be replaced by `weechat_data_dir`.

#### weechat_python_load

The info `weechat_dir` must be replaced by `weechat_data_dir`.

#### script_completion_scripts_files_cb

The info `weechat_dir` must be replaced by `weechat_data_dir`.

#### script_repo_get_filename_loaded

The info `weechat_dir` must be replaced by `weechat_data_dir`.

#### script_repo_update_status

The info `weechat_dir` must be replaced by `weechat_data_dir`.

#### spell_warning_aspell_config

The string `%h` must be replaced by `${weechat_config_dir}` in the call
to function `weechat_string_eval_path_home`.

### Options using WeeChat home

The following options are referencing WeeChat home with `%h`:

- if the default value uses `%h`, it is changed to use the new directory
- when using/evaluating the option value, any remaining `%h` is forced to the
  new directory.

Option                           | Old default value                    | New default value                     | Forced directory
-------------------------------- | ------------------------------------ | ------------------------------------- | ----------------
fifo.file.path                   | `%h/weechat_fifo`                    | `${weechat_runtime_dir}/weechat_fifo` | runtime
irc.server\_default.sasl\_key    | (empty string)                       | (unchanged)                           | config
irc.server.\*.sasl\_key          | (null)                               | (unchanged)                           | config
irc.server\_default.ssl\_cert    | (empty string)                       | (unchanged)                           | config
irc.server.\*.ssl\_cert          | (null)                               | (unchanged)                           | config
logger.file.path                 | `%h/logs/`                           | `${weechat_data_dir}/logs/`           | data
relay.network.ssl\_cert\_key     | `%h/ssl/relay.pem`                   | `${weechat_config_dir}/ssl/relay.pem` | config
relay.port.\*                    | (option not defined)                 | (unchanged)                           | runtime
script.scripts.path              | `%h/script`                          | `${weechat_cache_dir}/script`         | cache
weechat.network.gnutls\_ca\_file | `/etc/ssl/certs/ca-certificates.crt` | (unchanged)                           | config
weechat.plugin.path              | `%h/plugins`                         | `${weechat_data_dir}/plugins`         | data
xfer.file.download\_path         | `%h/xfer`                            | `${weechat_data_dir}/xfer`            | data
xfer.file.upload\_path           | `~`                                  | (unchanged)                           | data

The following scripts are using one of these options and must be changed if
the `%h` is replaced by a hard coded directory or if function `string_eval_path_home`
is called on them:

Script         | Option used      | Update needed
-------------- | ---------------- | -------------
grep.py        | logger.file.path | **Yes**
purgelogs.py   | logger.file.path | **Yes**

## Planning

The changes must be implemented in this order:

1. evaluate option `weechat.network.gnutls_ca_file`
2. evaluate option `weechat.plugin.path`
3. evaluate options `irc.server_default.sasl_key` and `irc.server.*.sasl_key`
4. evaluate options `irc.server_default.ssl_cert` and `irc.server.*.ssl_cert`
5. evaluate option `relay.network.ssl_cert_key`
6. set default WeeChat home to empty string in CMake and autotools
7. add the 4 new variables with directories: set them on startup, display them in `/debug dirs`, do not use them for now
8. use the 4 directories, remove variable `weechat_home`:
   1. allow 4 directories in forced home (compilation option, `-d` / `--dir`, environment variable `WEECHAT_HOME`)
   2. replace the 4 directories in `string_eval_expression`
   3. add 4 info with new directories
   4. update all impacted functions
   5. remove use of info `weechat_dir` in C plugins
   6. force appropriate directory in calls to `string_eval_path_home`
   7. remove use of `%h` in options: change default values, update help
9. update all scripts

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/001285-follow-xdg-base-dir-spec.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/001285-follow-xdg-base-dir-spec.md)
- WeeChat issue on GitHub: [https://github.com/weechat/weechat/issues/1285](https://github.com/weechat/weechat/issues/1285)
- WeeChat task on Savannah (obsolete): [https://savannah.nongnu.org/task/?10934](https://savannah.nongnu.org/task/?10934)
- Latest XDG Base Directory Specification: [https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html](https://specifications.freedesktop.org/basedir-spec/basedir-spec-latest.html)
