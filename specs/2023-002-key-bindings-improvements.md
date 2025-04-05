# Key bindings improvements

<!-- SPDX-FileCopyrightText: 2023 Sébastien Helleu <flashcode@flashtux.org> -->
<!-- SPDX-License-Identifier: CC-BY-NC-SA-4.0 -->

- Author: [Sébastien Helleu](https://github.com/flashcode)
- License: CC BY-NC-SA 4.0
- Created on: 2023-02-02
- Last updated: 2023-04-23
- Issues:
  - [#1875](https://github.com/weechat/weechat/issues/1875): force Control keys to lower case
  - [#1238](https://github.com/weechat/weechat/issues/1238): add aliases for key bindings
  - [task #12427](https://savannah.nongnu.org/task/?12427): need a way to list key/mouse events bound to a command
  - [task #11783](https://savannah.nongnu.org/task/?11783): add regexp search to /key list
  - [task #10317](https://savannah.nongnu.org/task/?10317): split key sequences in /key output
- Status: implemented
- Target WeeChat version: 4.0.0

## Context

WeeChat uses raw key codes to bind keys with `/key bind`, the user has to use
key `alt+k` to find such code and any typo in the key may lead to unusable
WeeChat.

There should be a visual way to edit key bindings.

## Goals

Purpose of this specification is to make easier the binding of keys:

- use human readable key bindings (eg: `meta-left` instead of `meta2-1;3D`)
- use comma to separate keys in combos (eg: `meta-w,meta-up` instead of `meta-wmeta-meta2-A`)
- convert control keys to lower keys (eg: `ctrl-a` instead of `ctrl-A`)
- make keys normal options, so they are shown and can be updated with `/set` and `/fset` commands
- make command `/key` without arguments open the fset buffer with all keys

## Out of scope

Keys specific to buffers (like on `/script` buffer) are not covered by this specification.

They may be improved in future by allowing easily the user to add, change or delete keys on specific buffers, and save these keys in configuration file.

## Changes

### New key format

New rules:

- the key combos must be separated by a comma
- the modifiers are `meta-`, `ctrl-` and `shift-` (see below)
- the control keys are in lower case (eg `ctrl-a`, not `ctrl-A`)
- the comma is `comma`
- the space is `space`
- keys that are not simple chars have aliases (see below), but the raw code is still valid and accepted by WeeChat, with higher priority (not recommended, but can be useful sometimes).

Note: the keys are still case sensitive, so `meta-a` is different from `meta-A` (which is `alt` + `shift` + `a`).

Modifiers are used in this order:

1. `meta-`: usually `alt` key; zero, one or multiple are allowed
2. `ctrl-`: zero or one allowed
3. `shift-`: zero or one allowed, and it works only with the new aliases (see below).

The mouse modifiers are used in this order:

1. `alt-`: `alt` key; zero or one allowed
2. `ctrl-`: zero or one allowed

The legacy mouse keys were using `ctrl-` before `alt-`, so there's a change here.\
For compatibility reasons, WeeChat will fix the key on-the-fly when it is created, for example:

```text
/key bindctxt mouse @chat:ctrl-alt-wheelup /test

New key binding (context "mouse"): @chat:alt-ctrl-wheelup => /test
```

Legacy modifier `meta2-` is removed (it is converted to `meta-[` in raw key codes).

Keys that are not simple chars are converted to aliases, the complete list is:

Key alias   | Description
----------- | ------------------
`f0`        | Function key `F0`
`f1`        | Function key `F1`
`f2`        | Function key `F2`
`f3`        | Function key `F3`
`f4`        | Function key `F4`
`f5`        | Function key `F5`
`f6`        | Function key `F6`
`f7`        | Function key `F7`
`f8`        | Function key `F8`
`f9`        | Function key `F9`
`f10`       | Function key `F10`
`f11`       | Function key `F11`
`f12`       | Function key `F12`
`f13`       | Function key `F13`
`f14`       | Function key `F14`
`f15`       | Function key `F15`
`f16`       | Function key `F16`
`f17`       | Function key `F17`
`f18`       | Function key `F18`
`f19`       | Function key `F19`
`f20`       | Function key `F20`
`home`      | Home key
`insert`    | Insert key
`delete`    | Delete key
`end`       | End key
`backspace` | Backspace key
`pgup`      | Page Up
`pgdn`      | Page Down
`up`        | Up arrow
`down`      | Down arrow
`right`     | Right arrow
`left`      | Left arrow
`tab`       | Tab
`return`    | Return
`comma`     | A comma (`,`)
`space`     | A space ("` `")

### Keyboard debug mode

A keyboard debug mode is added with command `/debug key` which displays on each key pressed:

- raw key code
- key name
- key name using alias
- command bound if key is found
- indication whether text must be inserted into input

During this debug mode, no signal nor command is actually executed.

The key `q` is used to quit this mode.

Example of debug messages for some keys:

```text
# key: ctrl + r
debug: "^r" -> ctrl-r -> ctrl-r -> "/input search_text_here"

# key: arrow up
debug: "^[[A" -> meta-[A -> up -> "/input history_previous"

# key: alt + arrow up
debug: "^[[1;3A" -> meta-[1;3A -> meta-up -> "/buffer -1"

# key: alt + w then alt + arrow left
debug: "^[w^[[1;3D" -> meta-w,meta-[1;3D -> meta-w,meta-left -> "/window left"

# key: alt + e
debug: "^[e" -> meta-e -> meta-e  [no key binding]

# key: a
debug: "a" -> a -> a  [no key binding]

# click with mouse "button1" (30, 15)
debug: "^[[M"  [mouse]
debug: " ?0"  [mouse]
debug: "#?0"  [mouse]
```

### Human readable keys

A new function is added to convert a raw key code to a human readable key.

Here are some examples of raw keys and their human readable version that must be used to bind keys:

Raw key          | Key name      | Comment
---------------- | ------------- | -------------
"`,`" (comma)    | `comma`       |
"` `" (space)    | `space`       |
`ctrl-j`         | `return`      |
`ctrl-m`         | `return`      |
`ctrl-i`         | `tab`         |
`meta-[Z`        | `shift-tab`   |
`ctrl-h`         | `backspace`   |
`ctrl-?`         | `backspace`   |
`meta-[H`        | `home`        |
`meta-[1~`       | `home`        |
`meta-[7~`       | `home`        | urxvt
`meta-meta-[1~`  | `meta-home`   |
`meta-meta-[7~`  | `meta-home`   | urxvt
`meta-[1;3H`     | `meta-home`   |
`meta-[F`        | `end`         |
`meta-[4~`       | `end`         |
`meta-[8~`       | `end`         | urxvt
`meta-meta-[4~`  | `meta-end`    |
`meta-meta-[8~`  | `meta-end`    | urxvt
`meta-[1;3F`     | `meta-end`    |
`meta-[2~`       | `insert`      |
`meta-[3~`       | `delete`      |
`meta-[A`        | `up`          |
`meta-[B`        | `down`        |
`meta-[C`        | `right`       |
`meta-[D`        | `left`        |
`meta-Oa`        | `ctrl-up`     | urxvt
`meta-[1;5A`     | `ctrl-up`     |
`meta-Ob`        | `ctrl-down`   | urxvt
`meta-[1;5B`     | `ctrl-down`   |
`meta-Od`        | `ctrl-left`   | urxvt
`meta-[1;5D`     | `ctrl-left`   |
`meta-Oc`        | `ctrl-right`  | urxvt
`meta-[1;5C`     | `ctrl-right`  |
`meta-[1;3A`     | `meta-up`     |
`meta-meta-[A`   | `meta-up`     | urxvt
`meta-[1;3B`     | `meta-down`   |
`meta-meta-[B`   | `meta-down`   | urxvt
`meta-[1;3C`     | `meta-right`  |
`meta-meta-[C`   | `meta-right`  | urxvt
`meta-[1;3D`     | `meta-left`   |
`meta-meta-[D`   | `meta-left`   | urxvt
`meta-[5~`       | `pgup`        |
`meta-[6~`       | `pgdn`        |
`meta-meta-[5~`  | `meta-pgup`   |
`meta-[5;3~`     | `meta-pgup`   |
`meta-meta-[6~`  | `meta-pgdn`   |
`meta-[6;3~`     | `meta-pgdn`   |
`meta-OP`        | `f1`          |
`meta-[11~`      | `f1`          | urxvt
`meta-[[A`       | `f1`          | Linux console
`meta-OQ`        | `f2`          |
`meta-[12~`      | `f2`          | urxvt
`meta-[[B`       | `f2`          | Linux console
`meta-OR`        | `f3`          |
`meta-[13~`      | `f3`          | urxvt
`meta-[[C`       | `f3`          | Linux console
`meta-OS`        | `f4`          |
`meta-[14~`      | `f4`          | urxvt
`meta-[[D`       | `f4`          | Linux console
`meta-[15~`      | `f5`          |
`meta-[[E`       | `f5`          | Linux console
`meta-[17~`      | `f6`          |
`meta-[18~`      | `f7`          |
`meta-[19~`      | `f8`          |
`meta-[20~`      | `f9`          |
`meta-[21~`      | `f10`         |
`meta-[23~`      | `f11`         |
`meta-[24~`      | `f12`         |
`meta-[11^`      | `ctrl-f1`     |
`meta-[1;5P`     | `ctrl-f1`     |
`meta-[12^`      | `ctrl-f2`     |
`meta-[1;5Q`     | `ctrl-f2`     |
`meta-[23^`      | `ctrl-f11`    |
`meta-[23;5~`    | `ctrl-f11`    |
`meta-[24^`      | `ctrl-f12`    |
`meta-[24;5~`    | `ctrl-f12`    |
`meta-meta-OP`   | `meta-f1`     |
`meta-meta-[11~` | `meta-f1`     |
`meta-[1;3P`     | `meta-f1`     |
`meta-meta-OQ`   | `meta-f2`     |
`meta-meta-[12~` | `meta-f2`     |
`meta-[1;3Q`     | `meta-f2`     |
`meta-[23;3~`    | `meta-f11`    |
`meta-meta-[23~` | `meta-f11`    |
`meta-[24;3~`    | `meta-f12`    |
`meta-meta-[24~` | `meta-f12`    |

### Manually convert a legacy key

The option `legacy` is added in command `/key` to convert a legacy key to the new format using the alias.

For example:

```text
/key legacy meta-meta-[A

"meta-meta-[A" => "meta-up"
```

### New default keys

The following tables show the new default keys for each context; this default
key is created by default in configuration and is returned by `alt+k`.

A new key `meta-K` (alt + upper K) is added and bound to `/input grab_raw_key_command`: it displays a raw code and its associated command.\
The raw code is still supported by WeeChat and can be used as well even if it's not recommended (it should be used only as a workaround in case of problem with the new name).

Default keys in context "default", WeeChat core:

Old default key(s)                                 | New default key     | Command
-------------------------------------------------- | ------------------- | ----------------------------------------
`ctrl-M`<br>`ctrl-J`                               | `return`            | `/input return`
`meta-ctrl-M`                                      | `meta-return`       | `/input insert \n`
`ctrl-I`                                           | `tab`               | `/input complete_next`
`meta2-Z`                                          | `shift-tab`         | `/input complete_previous`
`ctrl-R`                                           | `ctrl-r`            | `/input search_text_here`
`ctrl-H`<br>`ctrl-?`                               | `backspace`         | `/input delete_previous_char`
`ctrl-_`                                           | `ctrl-_`            | `/input undo`
`meta-_`                                           | `meta-_`            | `/input redo`
`meta2-3~`                                         | `delete`            | `/input delete_next_char`
`ctrl-D`                                           | `ctrl-d`            | `/input delete_next_char`
`ctrl-W`                                           | `ctrl-w`            | `/input delete_previous_word_whitespace`
`meta-ctrl-?`                                      | `meta-backspace`    | `/input delete_previous_word`
`ctrl-X`                                           | `ctrl-x`            | `/buffer switch`
`meta-x`                                           | `meta-x`            | `/buffer zoom`
`meta-d`                                           | `meta-d`            | `/input delete_next_word`
`ctrl-K`                                           | `ctrl-k`            | `/input delete_end_of_line`
`meta-r`                                           | `meta-r`            | `/input delete_line`
`ctrl-T`                                           | `ctrl-t`            | `/input transpose_chars`
`ctrl-U`                                           | `ctrl-u`            | `/input delete_beginning_of_line`
`ctrl-Y`                                           | `ctrl-y`            | `/input clipboard_paste`
`meta2-1~`<br>`meta2-H`<br>`meta2-7~`<br>`meta-OH` | `home`              | `/input move_beginning_of_line`
`ctrl-A`                                           | `ctrl-a`            | `/input move_beginning_of_line`
`meta2-4~`<br>`meta2-F`<br>`meta2-8~`<br>`meta-OF` | `end`               | `/input move_end_of_line`
`ctrl-E`                                           | `ctrl-e`            | `/input move_end_of_line`
`meta2-D`                                          | `left`              | `/input move_previous_char`
`ctrl-B`                                           | `ctrl-b`            | `/input move_previous_char`
`meta2-C`                                          | `right`             | `/input move_next_char`
`ctrl-F`                                           | `ctrl-f`            | `/input move_next_char`
`meta-b`                                           | `meta-b`            | `/input move_previous_word`
`meta-Od`<br>`meta-OD`<br>`meta2-1;5D`             | `ctrl-left`         | `/input move_previous_word`
`meta-f`                                           | `meta-f`            | `/input move_next_word`
`meta-Oc`<br>`meta-OC`<br>`meta2-1;5C`             | `ctrl-right`        | `/input move_next_word`
`meta2-A`                                          | `up`                | `/input history_previous`
`meta2-B`                                          | `down`              | `/input history_next`
`meta-Oa`<br>`meta-OA`<br>`meta2-1;5A`             | `ctrl-up`           | `/input history_global_previous`
`meta-Ob`<br>`meta-OB`<br>`meta2-1;5B`             | `ctrl-down`         | `/input history_global_next`
`meta-a`                                           | `meta-a`            | `/buffer jump smart`
`meta-jmeta-f`                                     | `meta-j,meta-f`     | `/buffer -`
`meta-jmeta-l`                                     | `meta-j,meta-l`     | `/buffer +`
`meta-jmeta-r`                                     | `meta-j,meta-r`     | `/server raw`
`meta-jmeta-s`                                     | `meta-j,meta-s`     | `/server jump`
`meta-hmeta-c`                                     | `meta-h,meta-c`     | `/hotlist clear`
`meta-hmeta-m`                                     | `meta-h,meta-m`     | `/hotlist remove`
`meta-hmeta-r`                                     | `meta-h,meta-r`     | `/hotlist restore`
`meta-hmeta-R`                                     | `meta-h,meta-R`     | `/hotlist restore -all`
`meta-k`                                           | `meta-k`            | `/input grab_key_command`
`meta-s`                                           | `meta-s`            | `/mute spell toggle`
`meta-u`                                           | `meta-u`            | `/window scroll_unread`
`ctrl-Sctrl-U`                                     | `ctrl-s,ctrl-u`     | `/allbuf /buffer set unread`
`ctrl-Cb`                                          | `ctrl-c,b`          | `/input insert \x02`
`ctrl-Cc`                                          | `ctrl-c,c`          | `/input insert \x03`
`ctrl-Ci`                                          | `ctrl-c,i`          | `/input insert \x1D`
`ctrl-Co`                                          | `ctrl-c,o`          | `/input insert \x0F`
`ctrl-Cv`                                          | `ctrl-c,v`          | `/input insert \x16`
`ctrl-C_`                                          | `ctrl-c,_`          | `/input insert \x1F`
`meta-meta2-C`<br>`meta2-1;3C`                     | `meta-right`        | `/buffer +1`
`meta-meta2-B`<br>`meta2-1;3B`                     | `meta-down`         | `/buffer +1`
`meta2-17~`                                        | `f6`                | `/buffer +1`
`ctrl-N`                                           | `ctrl-n`            | `/buffer +1`
`meta-meta2-D`<br>`meta2-1;3D`                     | `meta-left`         | `/buffer -1`
`meta-meta2-A`<br>`meta2-1;3A`                     | `meta-up`           | `/buffer -1`
`meta2-15~`<br>`meta2-[E`                          | `f5`                | `/buffer -1`
`ctrl-P`                                           | `ctrl-p`            | `/buffer -1`
`meta2-5~`<br>`meta2-I`                            | `pgup`              | `/window page_up`
`meta2-6~`<br>`meta2-G`                            | `pgdn`              | `/window page_down`
`meta-meta2-5~`<br>`meta2-5;3~`                    | `meta-pgup`         | `/window scroll_up`
`meta-meta2-6~`<br>`meta2-6;3~`                    | `meta-pgdn`         | `/window scroll_down`
`meta-meta2-1~`<br>`meta-meta2-7~`<br>`meta2-1;3H` | `meta-home`         | `/window scroll_top`
`meta-meta2-4~`<br>`meta-meta2-8~`<br>`meta2-1;3F` | `meta-end`          | `/window scroll_bottom`
`meta-n`                                           | `meta-n`            | `/window scroll_next_highlight`
`meta-p`                                           | `meta-p`            | `/window scroll_previous_highlight`
`meta-N`                                           | `meta-N`            | `/bar toggle nicklist`
`meta2-20~`                                        | `f9`                | `/bar scroll title * -30%`
`meta2-21~`                                        | `f10`               | `/bar scroll title * +30%`
`meta2-23~`                                        | `f11`               | `/bar scroll nicklist * -100%`
`meta2-24~`                                        | `f12`               | `/bar scroll nicklist * +100%`
`meta2-23^`<br>`meta2-23;5~`                       | `ctrl-f11`          | `/bar scroll nicklist * -100%`
`meta2-24^`<br>`meta2-24;5~`                       | `ctrl-f12`          | `/bar scroll nicklist * +100%`
`meta2-23;3~`<br>`meta-meta2-23~`                  | `meta-f11`          | `/bar scroll nicklist * b`
`meta2-24;3~`<br>`meta-meta2-24~`                  | `meta-f12`          | `/bar scroll nicklist * e`
`ctrl-L`                                           | `ctrl-l`            | `/window refresh`
`meta2-18~`                                        | `f7`                | `/window -1`
`meta2-19~`                                        | `f8`                | `/window +1`
`meta-wmeta-meta2-A`<br>`meta-wmeta2-1;3A`         | `meta-w,meta-up`    | `/window up`
`meta-wmeta-meta2-B`<br>`meta-wmeta2-1;3B`         | `meta-w,meta-down`  | `/window down`
`meta-wmeta-meta2-C`<br>`meta-wmeta2-1;3C`         | `meta-w,meta-right` | `/window right`
`meta-wmeta-meta2-D`<br>`meta-wmeta2-1;3D`         | `meta-w,meta-left`  | `/window left`
`meta-wmeta-b`                                     | `meta-w,meta-b`     | `/window balance`
`meta-wmeta-s`                                     | `meta-w,meta-s`     | `/window swap`
`meta-z`                                           | `meta-z`            | `/window zoom`
`meta-=`                                           | `meta-=`            | `/filter toggle`
`meta--`                                           | `meta--`            | `/filter toggle @`
`meta-0`                                           | `meta-0`            | `/buffer *10`
`meta-1`                                           | `meta-1`            | `/buffer *1`
`meta-2`                                           | `meta-2`            | `/buffer *2`
`meta-3`                                           | `meta-3`            | `/buffer *3`
`meta-4`                                           | `meta-4`            | `/buffer *4`
`meta-5`                                           | `meta-5`            | `/buffer *5`
`meta-6`                                           | `meta-6`            | `/buffer *6`
`meta-7`                                           | `meta-7`            | `/buffer *7`
`meta-8`                                           | `meta-8`            | `/buffer *8`
`meta-9`                                           | `meta-9`            | `/buffer *9`
`meta-<`                                           | `meta-<`            | `/buffer jump prev_visited`
`meta->`                                           | `meta->`            | `/buffer jump next_visited`
`meta-/`                                           | `meta-/`            | `/buffer jump last_displayed`
`meta-l`                                           | `meta-l`            | `/window bare`
`meta-m`                                           | `meta-m`            | `/mute mouse toggle`
`meta2-200~`                                       | Key removed         | N/A
`meta2-201~`                                       | Key removed         | N/A
`meta-j01`                                         | `meta-j,0,1`        | `/buffer *1`
`meta-j02`                                         | `meta-j,0,2`        | `/buffer *2`
(…)                                                | (…)                 | (…)
`meta-j98`                                         | `meta-j,9,8`        | `/buffer *98`
`meta-j99`                                         | `meta-j,9,9`        | `/buffer *99`

Default keys in context "default", plugin buflist:

Old default key(s)                                 | New default key | Command
-------------------------------------------------- | --------------- | ----------------------------------------
`meta-B`                                           | `meta-B`        | `/buflist toggle`
`meta-OP`<br>`meta2-11~`                           | `f1`            | `/bar scroll buflist * -100%`
`meta-OQ`<br>`meta2-12~`                           | `f2`            | `/bar scroll buflist * +100%`
`meta2-11^`<br>`meta2-1;5P`                        | `ctrl-f1`       | `/bar scroll buflist * -100%`
`meta2-12^`<br>`meta2-1;5Q`                        | `ctrl-f2`       | `/bar scroll buflist * +100%`
`meta-meta-OP`<br>`meta-meta2-11~`<br>`meta2-1;3P` | `meta-f1`       | `/bar scroll buflist * b`
`meta-meta-OQ`<br>`meta-meta2-12~`<br>`meta2-1;3Q` | `meta-f2`       | `/bar scroll buflist * e`

Default keys in context "search":

Old default key(s) | New default key | Command
------------------ | --------------- | ----------------------------
`ctrl-M`           | `ctrl-m`        | `/input search_stop_here`
`ctrl-J`           | `ctrl-j`        | `/input search_stop_here`
`ctrl-Q`           | `ctrl-q`        | `/input search_stop`
`meta-c`           | `meta-c`        | `/input search_switch_case`
`ctrl-R`           | `ctrl-r`        | `/input search_switch_regex`
`ctrl-I`           | `ctrl-i`        | `/input search_switch_where`
`meta2-A`          | `up`            | `/input search_previous`
`meta2-B`          | `down`          | `/input search_next`

Default keys in context "cursor":

Old default key(s)             | New default key            | Command
------------------------------ | -------------------------- | -------------------------------------------------------
`ctrl-M`                       | `ctrl-m`                   | `/cursor stop`
`ctrl-J`                       | `ctrl-j`                   | `/cursor stop`
`meta2-A`                      | `up`                       | `/cursor move up`
`meta2-B`                      | `down`                     | `/cursor move down`
`meta2-D`                      | `left`                     | `/cursor move left`
`meta2-C`                      | `right`                    | `/cursor move right`
`meta-meta2-A`<br>`meta2-1;3A` | `meta-up`                  | `/cursor move area_up`
`meta-meta2-B`<br>`meta2-1;3B` | `meta-down`                | `/cursor move area_down`
`meta-meta2-D`<br>`meta2-1;3D` | `meta-left`                | `/cursor move area_left`
`meta-meta2-C`<br>`meta2-1;3C` | `meta-right`               | `/cursor move area_right`
`@chat:m`                      | `@chat:m`                  | `hsignal:chat_quote_message;/cursor stop`
`@chat:q`                      | `@chat:q`                  | `hsignal:chat_quote_prefix_message;/cursor stop`
`@chat:Q`                      | `@chat:Q`                  | `hsignal:chat_quote_time_prefix_message;/cursor stop`
`@item(buffer_nicklist):b`     | `@item(buffer_nicklist):b` | `/window ${_window_number};/ban ${nick}`
`@item(buffer_nicklist):k`     | `@item(buffer_nicklist):k` | `/window ${_window_number};/kick ${nick}`
`@item(buffer_nicklist):K`     | `@item(buffer_nicklist):K` | `/window ${_window_number};/kickban ${nick}`
`@item(buffer_nicklist):q`     | `@item(buffer_nicklist):q` | `/window ${_window_number};/query ${nick};/cursor stop`
`@item(buffer_nicklist):w`     | `@item(buffer_nicklist):w` | `/window ${_window_number};/whois ${nick}`

There are no changes on default keys in context "mouse".

### Backspace key

The backspace key is a bit special (for historical reasons related to terminals), and both raw keys `ctrl-?` and `ctrl-h` points to the new name `backspace`.

If you want to bind `ctrl` + `backspace`, you have to bind either `ctrl-h` or `ctrl-?` depending on the code returned by your terminal.

Example of command `showkey -a`:

```text
$ showkey -a

Press any keys - Ctrl-D will terminate this program

^? 	127 0177 0x7f        <-- pressed: backspace
^H 	  8 0010 0x08        <-- pressed: ctrl + backspace
```

Some terminals may have reverse keys (ie: `ctrl-?` for `ctrl` + `backspace`).

If `ctrl-h` is `ctrl` + `backspace` (as shown with `showkey` above), you could do this to delete previous word (same as `meta-backspace`):

```text
/key bind ctrl-h /input delete_previous_word
```

As raw codes have higher priority than names, this key will match before the default `backspace` key when you press `ctrl` + `backspace`.

### Keys as options

Keys are stored as standard options, that means they can be added, updated and deleted with `/set` and `/fset` commands.

The recommended way to create new keys is still `/key` as it controls the validity of key name.

When the command `/key` is called without arguments, the fset buffer with all keys is displayed (filter `weechat.key*`).

The new options are:

Options                | Description
---------------------- | ------------------------------------------------
`weechat.key.*`        | Keys for context "default"
`weechat.key_search.*` | Keys for context "search" (`ctrl+r`)
`weechat.key_cursor.*` | Keys for context "cursor" (`/cursor`)
`weechat.key_mouse.*`  | Keys for context "mouse" (mouse buttons / wheel)

Example of output with `/key` or `/fset weechat.key*`:

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│1/285 | Filter: weechat.key* | Sort: ~name | Key(input): alt+space=toggle bool│
│weechat.key.backspace: key "backspace" in context "default" [default: "/input │
│delete_previous_char""]                                                       │
│                                                                              │
│------------------------------------------------------------------------------│
│  weechat.key.backspace      string  "/input delete_previous_char"            │
│  weechat.key.ctrl-_         string  "/input undo"                            │
│  weechat.key.ctrl-a         string  "/input move_beginning_of_line"          │
│  weechat.key.ctrl-b         string  "/input move_previous_char"              │
│  weechat.key.ctrl-c,_       string  "/input insert \x1F"                     │
│  weechat.key.ctrl-c,b       string  "/input insert \x02"                     │
│  weechat.key.ctrl-c,c       string  "/input insert \x03"                     │
│  weechat.key.ctrl-c,i       string  "/input insert \x1D"                     │
│  weechat.key.ctrl-c,o       string  "/input insert \x0F"                     │
│  weechat.key.ctrl-c,v       string  "/input insert \x16"                     │
│  weechat.key.ctrl-d         string  "/input delete_next_char"                │
│  weechat.key.ctrl-down      string  "/input history_global_next"             │
│  weechat.key.ctrl-e         string  "/input move_end_of_line"                │
│  weechat.key.ctrl-f         string  "/input move_next_char"                  │
│  weechat.key.ctrl-f1        string  "//bar scroll buflist * -100%"           │
│  weechat.key.ctrl-f11       string  "/bar scroll nicklist * -100%"           │
│  weechat.key.ctrl-f12       string  "/bar scroll nicklist * +100%"           │
│  weechat.key.ctrl-f2        string  "/bar scroll buflist * +100%"            │
│  weechat.key.ctrl-k         string  "/input delete_end_of_line"              │
│  weechat.key.ctrl-l         string  "/window refresh"                        │
│  weechat.key.ctrl-left      string  "/input move_previous_word"              │
│  weechat.key.ctrl-n         string  "/buffer +1"                             │
│  weechat.key.ctrl-p         string  "/buffer -1"                             │
│  weechat.key.ctrl-r         string  "/input search_text_here"                │
│  weechat.key.ctrl-right     string  "/input move_next_word"                  │
│  weechat.key.ctrl-s,ctrl-u  string  "/allbuf /buffer set unread"             │
│  weechat.key.ctrl-t         string  "/input transpose_chars"                 │
│  weechat.key.ctrl-u         string  "/input delete_beginning_of_line"        │
│  weechat.key.ctrl-up        string  "/input history_global_previous"         │
│  weechat.key.ctrl-w         string  "/input delete_previous_word_whitespace" │
│  weechat.key.ctrl-x         string  "/buffer switch"                         │
│  weechat.key.ctrl-y         string  "/input clipboard_paste"                 │
│  weechat.key.delete         string  "/input delete_next_char"                │
│  weechat.key.down           string  "/input history_next"                    │
│  weechat.key.end            string  "/input move_end_of_line"                │
│  weechat.key.f1             string  "/bar scroll buflist * -100%"            │
│  weechat.key.f10            string  "/bar scroll title * +30%"               │
│  weechat.key.f11            string  "/bar scroll nicklist * -100%"           │
│  weechat.key.f12            string  "/bar scroll nicklist * +100%"           │
│  weechat.key.f2             string  "/bar scroll buflist * +100%"            │
│  weechat.key.f5             string  "/buffer -1"                             │
│  weechat.key.f6             string  "/buffer +1"                             │
│  weechat.key.f7             string  "/window -1"                             │
│  weechat.key.f8             string  "/window +1"                             │
│  weechat.key.f9             string  "/bar scroll title * -30%"               │
│  weechat.key.home           string  "/input move_beginning_of_line"          │
│ [20:54] [2] [fset] 2:fset                                                    │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Backward compatibility

#### Convert legacy keys

A function is added to convert a legacy key code to the new one:

- add commas between keys in a combo (for example: `ctrl-Cb` → `ctrl-c,b`)
- convert any comma used as char into `comma` (for example: `meta-,` → `meta-comma`)
- convert name to human readable name (for example: `meta2-1;3D` → `meta-left`)

Example of conversion:

Legacy key           | New key          | Description
-------------------- | ---------------- | -------------------------------
`a`                  | `a`              | Letter `a`
`meta-a`             | `meta-a`         | Alt + `a`
`meta-A`             | `meta-A`         | Alt + shift + `a` (= Alt + `A`)
`ctrl-Cb`            | `ctrl-c,b`       | Ctrl + `c` then `b`
`meta2-D`            | `left`           | Left arrow
`meta2-Z`            | `shift-tab`      | Shift + `Tab`
`ctrl-M`             | `return`         | Return
`ctrl-J`             | `return`         | Return
`meta-wmeta-meta2-A` | `meta-w,meta-up` | Alt + `w` then Alt + up arrow

#### Configuration file version

A version is added in configuration files to track incompatible changes, like the new key format being introduced.

A new function `config_file_set_version` is added and must be called after `config_file_new`.\
The prototype is the following:

```C
int config_file_set_version (
    struct t_config_file *config_file,
    int version,
    struct t_hashtable *(*callback_update)(const void *pointer,
                                           void *data,
                                           struct t_config_file *config_file,
                                           int version_read,
                                           struct t_hashtable *data_read),
    const void *callback_update_pointer,
    void *callback_update_data);
```

It returns 1 if OK, 0 if error.

The function sets the configuration version and the callback inside `config_file`.
Callback is optional and can be NULL.

Then when the file is read from disk, WeeChat looks for a new option called `config_version` which is an integer and must be located before the first config section.

This is used to set the `version_read` in the `config_file` structure.
If missing, default version is `1`.

When a section or an option is read, the callback is called if the `version_read` is less than the config `version`.

The parameters given to the callback are:

Parameter           | Type      | Description
------------------- | --------- | ---------------------------------------------------------------------------------
`pointer`           | Pointer   | Pointer given to function `config_file_set_version`
`data`              | Pointer   | Data given to function `config_file_set_version` (freed when the config is freed)
`config_file`       | Pointer   | Pointer to configuration file being read
`version_read`      | Integer   | Version read in file (defaults to `1` if not found)
`data_read`         | Hashtable | Data read from file (see below).

The hashtable `data_read` can have the following keys (keys and values are strings):

Key                 | Availability    | Value
------------------- | --------------- | -------------------------------------------------------------
`config`            | Always set      | Name of configuration file, without extension (eg: `weechat`)
`section`           | Always set      | Name of section being read
`option`            | For option only | Name of the option
`value`             | For option only | Value of the option (if not NULL)
`value_null`        | For option only | Option has NULL value (value is always `1`)

#### Upgrade of weechat.conf

When upgrading WeeChat from an old release (with old keys), WeeChat converts all keys to the new format.

Multiple legacy names may point to the same new name, for example `ctrl-M` and `ctrl-J` both point to new key `return`.
In this case, only one key `return` is created with the latest key read.

That means before upgrade there are such keys in weechat.conf (partial extract):

```text
[key]
ctrl-J = "/input return"
ctrl-M = "/input return"
meta-ctrl-M = "/input insert \n"
meta-wmeta-meta2-A = "/window up"
(…)
```

And after upgrade, the keys are now:

```text
[key]
return = "/input return"
meta-return = "/input insert \n"
meta-w,meta-up = "/window up"
(…)
```

In addition, legacy commands (still there for compatibility) are automatically converted to the new command:

Old command                                         | New command
--------------------------------------------------- | -------------------------------------
`/input jump_smart`                                 | `/buffer jump smart`
`/input jump_last_buffer`                           | `/buffer +`
`/window ${_window_number};/input jump_last_buffer` | `/window ${_window_number};/buffer +`
`/input jump_last_buffer_displayed`                 | `/buffer jump last_displayed`
`/input jump_previously_visited_buffer`             | `/buffer jump prev_visited`
`/input jump_next_visited_buffer`                   | `/buffer jump next_visited`
`/input hotlist_clear`                              | `/hotlist clear`
`/input hotlist_remove_buffer`                      | `/hotlist remove`
`/input hotlist_restore_buffer`                     | `/hotlist restore`
`/input hotlist_restore_all`                        | `/hotlist restore -all`
`/input set_unread_current_buffer`                  | `/buffer set unread`
`/input set_unread`                                 | `/allbuf /buffer set unread`
`/input switch_active_buffer`                       | `/buffer switch`
`/input switch_active_buffer_previous`              | `/buffer switch -previous`
`/input zoom_merged_buffer`                         | `/buffer zoom`

Keys converted or removed are display on first upgrade of config, for example when upgrading from version 3.7 to 4.0.0:

```text
Legacy key converted: "ctrl-?" => "backspace"
Legacy key converted: "ctrl-A" => "ctrl-a"
Legacy key converted: "ctrl-B" => "ctrl-b"
Legacy key converted: "ctrl-C_" => "ctrl-c,_"
Legacy key converted: "ctrl-Cb" => "ctrl-c,b"
Legacy key converted: "ctrl-Cc" => "ctrl-c,c"
Legacy key converted: "ctrl-Ci" => "ctrl-c,i"
Legacy key converted: "ctrl-Co" => "ctrl-c,o"
Legacy key converted: "ctrl-Cv" => "ctrl-c,v"
Legacy key converted: "ctrl-D" => "ctrl-d"
Legacy key converted: "ctrl-E" => "ctrl-e"
Legacy key converted: "ctrl-F" => "ctrl-f"
Legacy key converted: "ctrl-H" => "backspace"
Legacy key converted: "ctrl-I" => "tab"
Legacy key converted: "ctrl-J" => "return"
Legacy key converted: "ctrl-K" => "ctrl-k"
Legacy key converted: "ctrl-L" => "ctrl-l"
Legacy key converted: "ctrl-M" => "return"
Legacy key converted: "ctrl-N" => "ctrl-n"
Legacy key converted: "ctrl-P" => "ctrl-p"
Legacy key converted: "ctrl-R" => "ctrl-r"
Legacy key converted: "ctrl-Sctrl-U" => "ctrl-s,ctrl-u"
Command converted for key "ctrl-s,ctrl-u": "/input set_unread" => "/allbuf /buffer set unread"
Legacy key converted: "ctrl-T" => "ctrl-t"
Legacy key converted: "ctrl-U" => "ctrl-u"
Legacy key converted: "ctrl-W" => "ctrl-w"
Legacy key converted: "ctrl-X" => "ctrl-x"
Command converted for key "ctrl-x": "/input switch_active_buffer" => "/buffer switch"
Legacy key converted: "ctrl-Y" => "ctrl-y"
Legacy key converted: "meta-ctrl-?" => "meta-backspace"
Legacy key converted: "meta-ctrl-M" => "meta-return"
Legacy key converted: "meta-meta-OP" => "meta-f1"
Legacy key converted: "meta-meta-OQ" => "meta-f2"
Legacy key converted: "meta-meta2-11~" => "meta-f1"
Legacy key converted: "meta-meta2-12~" => "meta-f2"
Legacy key converted: "meta-meta2-1~" => "meta-home"
Legacy key converted: "meta-meta2-23~" => "meta-f11"
Legacy key converted: "meta-meta2-24~" => "meta-f12"
Legacy key converted: "meta-meta2-4~" => "meta-end"
Legacy key converted: "meta-meta2-5~" => "meta-pgup"
Legacy key converted: "meta-meta2-6~" => "meta-pgdn"
Legacy key converted: "meta-meta2-7~" => "meta-home"
Legacy key converted: "meta-meta2-8~" => "meta-end"
Legacy key converted: "meta-meta2-A" => "meta-up"
Legacy key converted: "meta-meta2-B" => "meta-down"
Legacy key converted: "meta-meta2-C" => "meta-right"
Legacy key converted: "meta-meta2-D" => "meta-left"
Command converted for key "meta-/": "/input jump_last_buffer_displayed" => "/buffer jump last_displayed"
Command converted for key "meta-<": "/input jump_previously_visited_buffer" => "/buffer jump prev_visited"
Command converted for key "meta->": "/input jump_next_visited_buffer" => "/buffer jump next_visited"
Legacy key converted: "meta-OA" => "up"
Legacy key converted: "meta-OB" => "down"
Legacy key converted: "meta-OC" => "right"
Legacy key converted: "meta-OD" => "left"
Legacy key converted: "meta-OF" => "end"
Legacy key converted: "meta-OH" => "home"
Legacy key converted: "meta-OP" => "f1"
Legacy key converted: "meta-OQ" => "f2"
Legacy key converted: "meta-Oa" => "ctrl-up"
Legacy key converted: "meta-Ob" => "ctrl-down"
Legacy key converted: "meta-Oc" => "ctrl-right"
Legacy key converted: "meta-Od" => "ctrl-left"
Legacy key converted: "meta2-11^" => "ctrl-f1"
Legacy key converted: "meta2-11~" => "f1"
Legacy key converted: "meta2-12^" => "ctrl-f2"
Legacy key converted: "meta2-12~" => "f2"
Legacy key converted: "meta2-15~" => "f5"
Legacy key converted: "meta2-17~" => "f6"
Legacy key converted: "meta2-18~" => "f7"
Legacy key converted: "meta2-19~" => "f8"
Legacy key converted: "meta2-1;3A" => "meta-up"
Legacy key converted: "meta2-1;3B" => "meta-down"
Legacy key converted: "meta2-1;3C" => "meta-right"
Legacy key converted: "meta2-1;3D" => "meta-left"
Legacy key converted: "meta2-1;3F" => "meta-end"
Legacy key converted: "meta2-1;3H" => "meta-home"
Legacy key converted: "meta2-1;3P" => "meta-f1"
Legacy key converted: "meta2-1;3Q" => "meta-f2"
Legacy key converted: "meta2-1;5A" => "ctrl-up"
Legacy key converted: "meta2-1;5B" => "ctrl-down"
Legacy key converted: "meta2-1;5C" => "ctrl-right"
Legacy key converted: "meta2-1;5D" => "ctrl-left"
Legacy key converted: "meta2-1;5P" => "ctrl-f1"
Legacy key converted: "meta2-1;5Q" => "ctrl-f2"
Legacy key converted: "meta2-1~" => "home"
Legacy key removed: "meta2-200~"
Legacy key removed: "meta2-201~"
Legacy key converted: "meta2-20~" => "f9"
Legacy key converted: "meta2-21~" => "f10"
Legacy key converted: "meta2-23;3~" => "meta-f11"
Legacy key converted: "meta2-23;5~" => "ctrl-f11"
Legacy key converted: "meta2-23^" => "ctrl-f11"
Legacy key converted: "meta2-23~" => "f11"
Legacy key converted: "meta2-24;3~" => "meta-f12"
Legacy key converted: "meta2-24;5~" => "ctrl-f12"
Legacy key converted: "meta2-24^" => "ctrl-f12"
Legacy key converted: "meta2-24~" => "f12"
Legacy key converted: "meta2-3~" => "delete"
Legacy key converted: "meta2-4~" => "end"
Legacy key converted: "meta2-5;3~" => "meta-pgup"
Legacy key converted: "meta2-5~" => "pgup"
Legacy key converted: "meta2-6;3~" => "meta-pgdn"
Legacy key converted: "meta2-6~" => "pgdn"
Legacy key converted: "meta2-7~" => "home"
Legacy key converted: "meta2-8~" => "end"
Legacy key converted: "meta2-A" => "up"
Legacy key converted: "meta2-B" => "down"
Legacy key converted: "meta2-C" => "right"
Legacy key converted: "meta2-D" => "left"
Legacy key converted: "meta2-F" => "end"
Legacy key removed: "meta2-G"
Legacy key converted: "meta2-H" => "home"
Legacy key removed: "meta2-I"
Legacy key converted: "meta2-Z" => "shift-tab"
Legacy key converted: "meta2-[E" => "f5"
Command converted for key "meta-a": "/input jump_smart" => "/buffer jump smart"
Legacy key converted: "meta-hmeta-R" => "meta-h,meta-R"
Command converted for key "meta-h,meta-R": "/input hotlist_restore_all" => "/hotlist restore -all"
Legacy key converted: "meta-hmeta-c" => "meta-h,meta-c"
Command converted for key "meta-h,meta-c": "/input hotlist_clear" => "/hotlist clear"
Legacy key converted: "meta-hmeta-m" => "meta-h,meta-m"
Command converted for key "meta-h,meta-m": "/input hotlist_remove_buffer" => "/hotlist remove"
Legacy key converted: "meta-hmeta-r" => "meta-h,meta-r"
Command converted for key "meta-h,meta-r": "/input hotlist_restore_buffer" => "/hotlist restore"
Legacy key converted: "meta-jmeta-f" => "meta-j,meta-f"
Legacy key converted: "meta-jmeta-l" => "meta-j,meta-l"
Legacy key converted: "meta-jmeta-r" => "meta-j,meta-r"
Legacy key converted: "meta-jmeta-s" => "meta-j,meta-s"
Legacy key converted: "meta-j01" => "meta-j,0,1"
Legacy key converted: "meta-j02" => "meta-j,0,2"
Legacy key converted: "meta-j03" => "meta-j,0,3"
Legacy key converted: "meta-j04" => "meta-j,0,4"
Legacy key converted: "meta-j05" => "meta-j,0,5"
Legacy key converted: "meta-j06" => "meta-j,0,6"
Legacy key converted: "meta-j07" => "meta-j,0,7"
Legacy key converted: "meta-j08" => "meta-j,0,8"
Legacy key converted: "meta-j09" => "meta-j,0,9"
Legacy key converted: "meta-j10" => "meta-j,1,0"
Legacy key converted: "meta-j11" => "meta-j,1,1"
Legacy key converted: "meta-j12" => "meta-j,1,2"
Legacy key converted: "meta-j13" => "meta-j,1,3"
Legacy key converted: "meta-j14" => "meta-j,1,4"
Legacy key converted: "meta-j15" => "meta-j,1,5"
Legacy key converted: "meta-j16" => "meta-j,1,6"
Legacy key converted: "meta-j17" => "meta-j,1,7"
Legacy key converted: "meta-j18" => "meta-j,1,8"
Legacy key converted: "meta-j19" => "meta-j,1,9"
Legacy key converted: "meta-j20" => "meta-j,2,0"
Legacy key converted: "meta-j21" => "meta-j,2,1"
Legacy key converted: "meta-j22" => "meta-j,2,2"
Legacy key converted: "meta-j23" => "meta-j,2,3"
Legacy key converted: "meta-j24" => "meta-j,2,4"
Legacy key converted: "meta-j25" => "meta-j,2,5"
Legacy key converted: "meta-j26" => "meta-j,2,6"
Legacy key converted: "meta-j27" => "meta-j,2,7"
Legacy key converted: "meta-j28" => "meta-j,2,8"
Legacy key converted: "meta-j29" => "meta-j,2,9"
Legacy key converted: "meta-j30" => "meta-j,3,0"
Legacy key converted: "meta-j31" => "meta-j,3,1"
Legacy key converted: "meta-j32" => "meta-j,3,2"
Legacy key converted: "meta-j33" => "meta-j,3,3"
Legacy key converted: "meta-j34" => "meta-j,3,4"
Legacy key converted: "meta-j35" => "meta-j,3,5"
Legacy key converted: "meta-j36" => "meta-j,3,6"
Legacy key converted: "meta-j37" => "meta-j,3,7"
Legacy key converted: "meta-j38" => "meta-j,3,8"
Legacy key converted: "meta-j39" => "meta-j,3,9"
Legacy key converted: "meta-j40" => "meta-j,4,0"
Legacy key converted: "meta-j41" => "meta-j,4,1"
Legacy key converted: "meta-j42" => "meta-j,4,2"
Legacy key converted: "meta-j43" => "meta-j,4,3"
Legacy key converted: "meta-j44" => "meta-j,4,4"
Legacy key converted: "meta-j45" => "meta-j,4,5"
Legacy key converted: "meta-j46" => "meta-j,4,6"
Legacy key converted: "meta-j47" => "meta-j,4,7"
Legacy key converted: "meta-j48" => "meta-j,4,8"
Legacy key converted: "meta-j49" => "meta-j,4,9"
Legacy key converted: "meta-j50" => "meta-j,5,0"
Legacy key converted: "meta-j51" => "meta-j,5,1"
Legacy key converted: "meta-j52" => "meta-j,5,2"
Legacy key converted: "meta-j53" => "meta-j,5,3"
Legacy key converted: "meta-j54" => "meta-j,5,4"
Legacy key converted: "meta-j55" => "meta-j,5,5"
Legacy key converted: "meta-j56" => "meta-j,5,6"
Legacy key converted: "meta-j57" => "meta-j,5,7"
Legacy key converted: "meta-j58" => "meta-j,5,8"
Legacy key converted: "meta-j59" => "meta-j,5,9"
Legacy key converted: "meta-j60" => "meta-j,6,0"
Legacy key converted: "meta-j61" => "meta-j,6,1"
Legacy key converted: "meta-j62" => "meta-j,6,2"
Legacy key converted: "meta-j63" => "meta-j,6,3"
Legacy key converted: "meta-j64" => "meta-j,6,4"
Legacy key converted: "meta-j65" => "meta-j,6,5"
Legacy key converted: "meta-j66" => "meta-j,6,6"
Legacy key converted: "meta-j67" => "meta-j,6,7"
Legacy key converted: "meta-j68" => "meta-j,6,8"
Legacy key converted: "meta-j69" => "meta-j,6,9"
Legacy key converted: "meta-j70" => "meta-j,7,0"
Legacy key converted: "meta-j71" => "meta-j,7,1"
Legacy key converted: "meta-j72" => "meta-j,7,2"
Legacy key converted: "meta-j73" => "meta-j,7,3"
Legacy key converted: "meta-j74" => "meta-j,7,4"
Legacy key converted: "meta-j75" => "meta-j,7,5"
Legacy key converted: "meta-j76" => "meta-j,7,6"
Legacy key converted: "meta-j77" => "meta-j,7,7"
Legacy key converted: "meta-j78" => "meta-j,7,8"
Legacy key converted: "meta-j79" => "meta-j,7,9"
Legacy key converted: "meta-j80" => "meta-j,8,0"
Legacy key converted: "meta-j81" => "meta-j,8,1"
Legacy key converted: "meta-j82" => "meta-j,8,2"
Legacy key converted: "meta-j83" => "meta-j,8,3"
Legacy key converted: "meta-j84" => "meta-j,8,4"
Legacy key converted: "meta-j85" => "meta-j,8,5"
Legacy key converted: "meta-j86" => "meta-j,8,6"
Legacy key converted: "meta-j87" => "meta-j,8,7"
Legacy key converted: "meta-j88" => "meta-j,8,8"
Legacy key converted: "meta-j89" => "meta-j,8,9"
Legacy key converted: "meta-j90" => "meta-j,9,0"
Legacy key converted: "meta-j91" => "meta-j,9,1"
Legacy key converted: "meta-j92" => "meta-j,9,2"
Legacy key converted: "meta-j93" => "meta-j,9,3"
Legacy key converted: "meta-j94" => "meta-j,9,4"
Legacy key converted: "meta-j95" => "meta-j,9,5"
Legacy key converted: "meta-j96" => "meta-j,9,6"
Legacy key converted: "meta-j97" => "meta-j,9,7"
Legacy key converted: "meta-j98" => "meta-j,9,8"
Legacy key converted: "meta-j99" => "meta-j,9,9"
Legacy key converted: "meta-wmeta-meta2-A" => "meta-w,meta-up"
Legacy key converted: "meta-wmeta-meta2-B" => "meta-w,meta-down"
Legacy key converted: "meta-wmeta-meta2-C" => "meta-w,meta-right"
Legacy key converted: "meta-wmeta-meta2-D" => "meta-w,meta-left"
Legacy key converted: "meta-wmeta2-1;3A" => "meta-w,meta-up"
Legacy key converted: "meta-wmeta2-1;3B" => "meta-w,meta-down"
Legacy key converted: "meta-wmeta2-1;3C" => "meta-w,meta-right"
Legacy key converted: "meta-wmeta2-1;3D" => "meta-w,meta-left"
Legacy key converted: "meta-wmeta-b" => "meta-w,meta-b"
Legacy key converted: "meta-wmeta-s" => "meta-w,meta-s"
Command converted for key "meta-x": "/input zoom_merged_buffer" => "/buffer zoom"
Legacy key converted: "ctrl-I" => "tab"
Legacy key converted: "ctrl-J" => "return"
Legacy key converted: "ctrl-M" => "return"
Legacy key converted: "ctrl-Q" => "ctrl-q"
Legacy key converted: "ctrl-R" => "ctrl-r"
Legacy key converted: "meta2-A" => "up"
Legacy key converted: "meta2-B" => "down"
Legacy key converted: "ctrl-J" => "return"
Legacy key converted: "ctrl-M" => "return"
Legacy key converted: "meta-meta2-A" => "meta-up"
Legacy key converted: "meta-meta2-B" => "meta-down"
Legacy key converted: "meta-meta2-C" => "meta-right"
Legacy key converted: "meta-meta2-D" => "meta-left"
Legacy key converted: "meta2-1;3A" => "meta-up"
Legacy key converted: "meta2-1;3B" => "meta-down"
Legacy key converted: "meta2-1;3C" => "meta-right"
Legacy key converted: "meta2-1;3D" => "meta-left"
Legacy key converted: "meta2-A" => "up"
Legacy key converted: "meta2-B" => "down"
Legacy key converted: "meta2-C" => "right"
Legacy key converted: "meta2-D" => "left"
Command converted for key "@chat:button1-gesture-right-long": "/window ${_window_number};/input jump_last_buffer" => "/window ${_window_number};/buffer +"
```

### Scripts using legacy keys

Some [official scripts](https://github.com/weechat/scripts) are using legacy keys but they are not broken because keys are automatically fixed when they are added:

- "` `" (space) is automatically replaced with "space" (eg: `meta- ` becomes `meta-space`)
- `ctrl-` + upper case letter is automatically converted to `ctrl-` + lower case letter (eg: `ctrl-A` becomes `ctrl-a`)
- `meta2-` is automatically converted to `meta-[` (eg: `meta2-A` becomes `meta-[A`)
- order of modifiers for mouse keys is `alt` then `ctrl` (eg: `@chat:ctrl-alt-wheelup` becomes `@chat:alt-ctrl-wheelup`)

List of scripts using legacy keys:

Script         | Reason
-------------- | ------------------------------------------------------
coords.pl      | Legacy keys like "` `" (space), `ctrl-A` and `meta2-A`
floodit.py     | Legacy keys like `meta2-C`
grep_filter.py | Legacy key: `ctrl-G`
listbuffer.py  | Legacy keys like `ctrl-L`, `meta2-A` and `meta-ctrl-J`
mastermind.pl  | Legacy keys like `meta2-A` and `meta-ctrl-M`
menu.pl        | Legacy keys like "` `" (space), `ctrl-H` and `meta2-A`
minesweeper.py | Legacy keys like `meta- ` (meta-space) and `meta2-A`
multiline.pl   | Legacy keys like "` `" (space), `ctrl-H` and `meta2-A`
pastebuf.lua   | Legacy keys like `meta2-A`
rssagg.pl      | Legacy keys like `meta2-A`
snake.pl       | Legacy keys like `meta2-A`
spell_menu.pl  | Legacy keys like "` `" (space) and `meta2-A`
urlgrab.py     | Legacy keys like `ctrl-R`, `meta2-A` and `meta-ctrl-J`
urlselect.lua  | Legacy keys like `ctrl-B` and `meta2-A`

Some other scripts, not part of official scripts, could be broken if they use key combos without the comma, for example: `meta-c1` (`meta-c` then `1`) instead of `meta-c,1`.

The fix would be something like this in Python:

```python
version = weechat.info_get("version_number", "") or 0
if int(version) >= 0x03090000:
    # new keys (WeeChat ≥ 4.0.0)
    weechat.buffer_set(buffer, "key_bind_meta-c,1", "/test1")
    weechat.buffer_set(buffer, "key_bind_meta-c,2", "/test2")
    weechat.buffer_set(buffer, "key_bind_meta-c,3", "/test3")
else:
    # legacy keys (WeeChat < 4.0.0)
    weechat.buffer_set(buffer, "key_bind_meta-c1", "/test1")
    weechat.buffer_set(buffer, "key_bind_meta-c2", "/test2")
    weechat.buffer_set(buffer, "key_bind_meta-c3", "/test3")
```

## Planning

The changes must be implemented in this order:

1. Convert control keys to lower case
2. Add keyboard debug mode
3. Add function to get expanded raw key code to key name with and without alias
4. Add function to convert a legacy key name to a key name with alias
5. Display new key name in output of `/key`
6. Add configuration file version
7. Convert all legacy keys to new format when reading configuration, use new format everywhere
8. Make keys standard options, so they can be managed with `/set` and `/fset` commands
9. Make command `/key` without arguments open the fset buffer with all keys
10. Fix scripts using legacy keys (optional, no urgency)

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-002-key-bindings-improvements.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-002-key-bindings-improvements.md)
