# Case sensitive identifiers

- Author: [Sébastien Helleu](https://github.com/flashcode)
- License: CC BY-NC-SA 4.0
- Created on: 2022-12-21
- Last updated: 2023-01-27
- Issue: [#1872](https://github.com/weechat/weechat/issues/1872): case sensitive identifiers
- Status: implemented
- Target WeeChat version: 3.9

## Context

Many identifiers are case insensitive in WeeChat and should not be.

## Goals

Purpose of this specification is to make many identifiers case sensitive in
data objects managed by the user, API and internal functions.

Changes proposed in this specification will also fix the following issues:

- [#398](https://github.com/weechat/weechat/issues/398): autocomplete wrong case
- [#32213](https://savannah.nongnu.org/bugs/?32213): completion problems with uppercase letters

## Out of scope

These identifiers must stay case insensitive:

- hook config (check if option is matching hook mask)
- hooks hsignal and signal (check if signal is matching hook signal masks)
- hook line (check if buffer is matching hook mask)
- hook modifier name
- hook print (check if prefix or message displayed is contained in hook message)
- buffer matching condition in filters
- system signal name ("kill", "term", "usr1", etc.)
- nick completions
- nicklist: groups and nicks
- fset filters
- weelist data (used to keep list sorted)
- IRC channel name
- IRC nicks
- IRC CTCP names
- IRC capabilities
- relay: IRC commands relayed
- relay: IRC commands ignored
- filters on trigger monitor buffer

## Changes

These identifiers are currently case insensitive and must become case sensitive:

- configuration (files, sections, options)
- commands
- commands parameters
- hook command_run
- hook info
- hook info_hashtable
- infolist names
- hashtable types
- curl constants
- curl options
- completion items
- completions (except nick completions)
- infolist variable name
- weelist position (beginning/end/sort)
- proxy options
- proxy types
- /secure buffer input (`q` to close)
- color names
- /color buffer input (`q` to close, etc.)
- custom bar item option ("conditions", "content")
- bar options
- bar types
- bar positions
- bar conditions ("active", "inactive", "nicklist")
- bar names
- buffer types ("formatted", "free")
- buffer notify levels (none, highlight, etc.)
- buffer match list
- user buffer input (`q` to close)
- function gui_color_get_custom (parameter `color_name`, example: `bold`, `reset`, etc.)
- filter names
- hotlist priorities ("low", "message", "private", "highlight")
- key contexts ("default", "search", "cursor", "mouse")
- notify tags in line ("notify_none", "notify_message", etc.)
- alias name
- IRC server name
- IRC raw buffer input (`q` to close)
- IRC raw filters
- API function prefix (parameter `prefix`, example: `error`, `network`, etc.)
- script name
- plugin name (eg: `irc` vs `Irc`)
- relay buffer input (`q` to close, etc.)
- relay compression (off/zlib/zstd)
- trigger names
- trigger options
- trigger types
- trigger return codes
- trigger post actions
- xfer buffer input (`q` to close, etc.)
- xfer types
- xfer protocols (none/dcc)
- functions to set object properties

### String comparison functions

When a comparison needs to be case sensitive instead of case insensitive, the call to the function must be replaced with another, in WeeChat core:

Function                     | Replacement
---------------------------- | ---------------------------------------
`string_charcasecmp`         | `string_charcmp`
`string_strcasecmp`          | `string_strcmp` or `strcmp`
`string_strncasecmp`         | `string_strncmp`
`string_strcmp_ignore_chars` | Same function with `case_sensitive` = 1
`string_strcasestr`          | `strstr`
`string_match`               | Same function with `case_sensitive` = 1
`string_match_list`          | Same function with `case_sensitive` = 1

When a comparison needs to be case sensitive instead of case insensitive, the call to the function must be replaced with another, in WeeChat plugins:

Function                      | Replacement
----------------------------- | ---------------------------------------
`weechat_string_charcasecmp`  | `weechat_string_charcmp`
`weechat_strcasecmp`          | `weechat_strcmp` or `strcmp`
`weechat_strncasecmp`         | `weechat_strncmp`
`weechat_strcmp_ignore_chars` | Same function with `case_sensitive` = 1
`weechat_strcasestr`          | `strstr`
`weechat_string_match`        | Same function with `case_sensitive` = 1
`weechat_string_match_list`   | Same function with `case_sensitive` = 1

### Configuration files, sections, options

Configuration files, sections and options are made case sensitive.

Functions to update:

- config_file_search
- config_file_config_find_pos
- config_file_section_find_pos
- config_file_search_section
- config_file_option_find_pos
- config_file_new_option
- config_file_search_option
- config_file_search_section_option
- config_file_string_boolean_is_valid
- config_file_string_to_boolean
- config_file_option_set
- config_file_read_internal

### Commands, commands parameters, command_runs, aliases and completions

Commands, commands parameters, hook "command_run", aliases and completions (except nick) are made case sensitive.

Functions to update:

- hook_command_search
- hook_command_exec
- hook_command_run_exec
- hook_completion_exec
- hook_find_pos
- hook_add_to_infolist_type
- command_help
- input_exec_command
- gui_completion_word_compare_cb
- gui_completion_search_command
- gui_completion_list_add
- gui_completion_complete
- alias_search
- alias_find_pos
- all command callbacks (core and plugins)

That means for example `/test -o` and `/test -O` (upper case parameter) would have different meaning, `-o` and `-O` being two different options for the command `/test`.

Regarding commands, a new alias `/AWAY` could be defined to be away on all servers, like this:

```text
/alias add AWAY /allserv /away
```

So that `/away` and `/AWAY` (the new alias) are two separate commands,
with separate completion: the completion of `/aw` is `/away` (only this one),
and the completion of `/AW` is `/AWAY` (only this one).

Nick completion remains case insensitive, that means if there are nicks "Nick"
and "Nick_away", typing "ni" with Tab will complete to partial completion "Nick"
and typing "_" and Tab again will complete to "Nick_away".

Partial completion is not converted to lower case any more, this fixes the issue
[#398](https://github.com/weechat/weechat/issues/398) and Savannah bug
[#32213](https://savannah.nongnu.org/bugs/?32213).

That means when partial completion is enabled (`/set weechat.completion.partial_completion_other on`), the result is the following:

Nicks in channel                 | Input text   | Old completion | New completion
-------------------------------- | ------------ | -------------- | --------------
`{Andrew}` and `{Andrew}_Mobile` | `{a` or `{A` | `{andrew}`     | `{Andrew}`
`NickName` and `NickToto`        | `ni` or `Ni` | `nick`         | `Nick`

### Info, info_hashtable, infolist

Name of "info", "info_hashtable" and "infolist" are made case sensitive.

Functions to update:

- hook_info_get
- hook_info_get_hashtable
- hook_infolist_get
- infolist_search_var
- infolist_integer
- infolist_string
- infolist_pointer
- infolist_buffer
- infolist_time

### Bars and bar items

Name of bar items and all these bar identifiers are made case sensitive:

- name
- options
- type
- position
- conditions

Functions to update:

- command_bar
- gui_bar_item_custom_search_option
- gui_bar_item_custom_search_with_option_name
- gui_bar_search_option
- gui_bar_search_type
- gui_bar_search_position
- gui_bar_check_conditions
- gui_bar_search
- gui_bar_search_with_option_name
- gui_bar_default_items
- gui_bar_update

### Plugins

Plugin names are made case sensitive.

Functions to update:

- command_plugin_list
- gui_layout_buffer_get_number
- plugin_search
- plugin_check_extension_allowed (option `weechat.plugin.extension`)
- plugin_check_autoload (option `weechat.plugin.extension`)
- xfer_search

### Functions to get/set properties

The functions used to set object properties are updated to be case sensitive for the property to set (parameter `property`):

- config_file_option_get_string
- config_file_option_get_pointer
- hashtable_get_integer
- hashtable_get_string
- hashtable_set_pointer
- hdata_get_string
- hook_set
- proxy_set
- gui_bar_set
- gui_buffer_get_integer
- gui_buffer_get_string
- gui_buffer_get_pointer
- gui_buffer_set
- gui_buffer_set_pointer
- gui_completion_get_string
- gui_nicklist_group_get_integer
- gui_nicklist_group_get_string
- gui_nicklist_group_get_pointer
- gui_nicklist_group_set
- gui_nicklist_nick_get_integer
- gui_nicklist_nick_get_string
- gui_nicklist_nick_get_pointer
- gui_nicklist_nick_set
- gui_window_get_integer
- gui_window_get_pointer

### Curl constants and options

Curl constants and options are made case sensitive.

Functions to update:

- weeurl_search_constant
- weeurl_search_option

### Hashtables

Hashtable types are made case sensitive.

Functions to update:

- hashtable_get_type

### Weelist

Weelist positions are made case sensitive.

Functions to update:

- weelist_insert

### Proxies

Proxy options and types are made case sensitive.

Proxy names are made case sensitive in read of infolist "proxy".

Functions to update:

- proxy_search_option
- proxy_search_type
- plugin_api_infolist_proxy_cb

### Buffers

Buffer types and notify levels are made case sensitive.

API function `buffer_match_list` is made case sensitive.

Buffer names are made case sensitive in read of infolist "buffer".

Functions to update:

- gui_buffer_search_type
- gui_buffer_search_notify
- gui_buffer_match_list
- plugin_api_infolist_buffer_cb

### Input actions in buffers

Input actions (like "q" to close buffer) are made case sensitive.

Functions to update:

- secure_buffer_input_cb
- gui_color_buffer_input_cb
- gui_buffer_user_input_cb
- irc_input_data
- relay_buffer_input_cb
- xfer_buffer_input_cb

### Colors

Color names (like `blue`, `red`, etc.) and color attributes (`bold`, `underline`, etc.) are made case sensitive.

Functions to update:

- gui_color_search
- gui_color_get_custom

### Filters

Filter names are made case sensitive.

Functions to update:

- gui_filter_find_pos

### Hotlist priorities

Hotlist priorities ("low", "message", "private", "highlight") are made case sensitive.

Functions to update:

- gui_hotlist_search_priority

### Key contexts

Key contexts ("default", "search", "cursor", "mouse") are made case sensitive.

Functions to update:

- gui_key_search_context

### Notify tags in line

Notify tags in line ("notify_none", "notify_message", etc.) are made case sensitive.

Functions to update:

- gui_line_set_notify_level

### IRC servers

IRC server names are made case sensitive.

Functions to update:

- irc_command_exec_all_servers
- irc_command_server
- irc_ignore_search
- irc_ignore_check_server
- irc_info_infolist_irc_server_cb
- irc_info_infolist_irc_notify_cb
- irc_server_casesearch (function removed)

### IRC raw filters

Filters on IRC raw buffer (except command name with `m:xxx`) are made case sensitive

Functions to update:

- irc_raw_message_match_filter

### API function "prefix"

The prefix parameter in "prefix" API function is made case sensitive.

Functions to update:

- plugin_api_prefix

### Scripts

Script names are made case sensitive.

Functions to update:

- plugin_script_search
- plugin_script_find_pos

### Triggers

Trigger names, options, types, return codes and post actions are made case sensitive.

Functions to update:

- trigger_command_trigger
- trigger_search_option
- trigger_search_hook_type
- trigger_search_return_code
- trigger_search_post_action
- trigger_search

### Xfer

Xfer types and protocols are made case sensitive.

Functions to update:

- xfer_search_type
- xfer_search_protocol

## Planning

The changes must be implemented in this order:

1. Make case sensitive:
    - configuration files, sections, options
    - aliases
2. Convert default aliases to lower case
3. Make case sensitive:
    - commands, commands parameters, hook "command_run", aliases and completions (except nick)
    - info, info_hashtable, infolist
    - bars, bar items
    - plugins
    - functions to get/set properties
    - Curl constants and options
    - hashtable types
    - weelist position
    - proxy options and types, proxy name in infolist "proxy"
    - buffer types and notify levels, API function buffer_match_list, buffer name in infolist "buffer"
    - input actions in buffers
    - color names and attributes
    - filter names
    - hotlist priorities
    - notify tags in line
    - IRC server names
    - IRC raw filters
    - API function "prefix"
    - script names
    - trigger names, options, types, return codes and post actions
    - xfer types and protocols

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-001-case-sensitive-identifiers.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-001-case-sensitive-identifiers.md)
