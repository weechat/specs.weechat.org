# Case sensitive identifiers

- Author: [Sébastien Helleu](https://github.com/flashcode)
- License: CC BY-NC-SA 4.0
- Created on: 2022-12-21
- Last updated: 2023-01-15
- Issue: [#1872](https://github.com/weechat/weechat/issues/1872): case sensitive identifiers
- Status: draft
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

- nick completions
- nicklist: groups and nicks
- fset filters
- irc channel name
- irc nicks
- relay: IRC commands relayed
- relay: IRC commands ignored

## Changes

These identifiers are currently case insensitive and must become case sensitive:

- configuration (files, sections, options)
- commands
- commands parameters
- hook command_run
- hook config
- hook info
- hook info_hashtable
- infolist names
- hashtable types
- curl constants
- curl options
- completion items
- completions (except nick completions)
- infolist variable name
- weelist data (used to keep list sorted)
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
- buffer notify (none, highlight, etc.)
- user buffer input (`q` to close)
- function gui_color_get_custom (parameter `color_name`, example: `bold`, `reset`, etc.)
- filter names
- hotlist priorities (low/message/private/highlight)
- key contexts (default/search/cursor/mouse)
- notify tags in line ("notify_none", "notify_message", etc.)
- alias name
- irc server name
- irc raw buffer input (`q` to close)
- API function prefix (parameter `prefix`, example: `error`, `network`, etc.)
- script name
- plugin name (eg: `irc` vs `Irc`)
- relay buffer input (`q` to close, etc.)
- relay compression (off/zlib/zstd)
- trigger options
- trigger types
- trigger return codes
- trigger post actions
- trigger names
- xfer buffer input (`q` to close, etc.)
- xfer types
- xfer protocols (none/dcc)
- functions to set object properties

### String comparison functions

The following string comparison functions must be replaced in WeeChat core to be case sensitive:

Function                     | Replacement
---------------------------- | ---------------------------------------
`string_charcasecmp`         | `string_charcmp`
`string_strcasecmp`          | `string_strcmp` or `strcmp`
`string_strncasecmp`         | `string_strncmp`
`string_strcmp_ignore_chars` | Same function with `case_sensitive` = 1
`string_strcasestr`          | `strstr`
`string_match`               | Same function with `case_sensitive` = 1
`string_match_list`          | Same function with `case_sensitive` = 1

The following string comparison functions must be replaced in WeeChat plugins to be case sensitive:

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

Nicks in channel                 | Input text   | Old completion    | New completion
-------------------------------- | ------------ | ----------------- | ----------------
`{Andrew}` and `{Andrew}_Mobile` | `{a` or `{A` | `{andrew}`        | `{Andrew}`
`NickName` and `NickToto`        | `ni` or `Ni` | `nick`            | `Nick`

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

## Planning

The changes must be implemented in this order:

1. Make case sensitive: configuration files, sections, options
2. Make case sensitive: aliases
3. Convert default aliases to lower case
4. Make case sensitive: commands, commands parameters, hook "command_run", aliases and completions (except nick)
5. Make case sensitive: info, info_hashtable, infolist
6. Make case sensitive: bars, bar items
7. Make case sensitive: plugins
8. Make case sensitive: functions to get/set properties
9. [TO BE COMPLETED]

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-001-case-sensitive-identifiers.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-001-case-sensitive-identifiers.md)
