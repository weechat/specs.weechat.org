# Fix unicode display

- Author: [Sébastien Helleu](https://github.com/flashcode)
- License: CC BY-NC-SA 4.0
- Created on: 2022-11-27
- Last updated: 2022-12-10
- Issues:
  - [#1659](https://github.com/weechat/weechat/issues/1659): soft-hyphens (U+00AD)
  - [#1669](https://github.com/weechat/weechat/issues/1669),
    [#1770](https://github.com/weechat/weechat/issues/1770): zero width spaces (U+200B)
- Status: in progress
- Target WeeChat version: 3.8

## Context

WeeChat has some issues with the display of Unicode chars and the way to display
some of them is different between chat area and bars.

## Goals

Purpose of this specification is to improve the display of Unicode chars
and fix display bugs:

- display chars the same way in chat and bars
- replace tabulations by spaces (everywhere)
- display low chars with a letter or symbol with reverse video attribute (everywhere)
- do not display soft hyphens
- do not display zero width spaces
- do not display non-printable chars
- fix some issues (detail in the following chapters).

## Out of scope

Some other bugs with unicode chars are not covered by this specification:

- [#625](https://github.com/weechat/weechat/issues/625): regular expressions in WeeChat do not (entirely) support unicode
- [#793](https://github.com/weechat/weechat/issues/793): wrong color with unicode chars on wrapped line
- [#946](https://github.com/weechat/weechat/issues/946): ignore zero width spaces for buffer search
- [#947](https://github.com/weechat/weechat/issues/947): escape zero width spaces in raw log
- [#1857](https://github.com/weechat/weechat/issues/1857): unicode chars with zero-width joiner are displayed as separate chars

## Unicode display

The unicode debug displayed in the following chapters is the output of command
`/debug unicode ${\u1234}`, with the result of the following functions:

- `strlen`: number of bytes in the string
- `utf8_strlen`: number of chars in the string
- `gui_chat_strlen`: number of chars in the string (WeeChat color codes are skipped)
- `wcwidth`: number of columns needed to display the char in the terminal
- `utf8_strlen_screen`: number of columns needed to display the char in the terminal
- `gui_chat_strlen_screen`: number of columns needed to display the string in the terminal (WeeChat color codes are skipped)

The result of `utf8_char_size_screen` is added before `utf8_strlen_screen`
because the result is different for non printable chars: `-1` instead of `0`.

The command output in next chapters includes this change.

### Chars U+0001 (1) to U+001F (31)

Unicode debug (with option `weechat.look.tab_width` set to `4`):

```text
/debug unicode ${chars:${\u0001}-${\u001F}}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  " " (U+0001, 1, 0x01): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0002, 2, 0x02): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0003, 3, 0x03): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0004, 4, 0x04): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0005, 5, 0x05): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0006, 6, 0x06): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0007, 7, 0x07): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0008, 8, 0x08): 1 / 1, 1 / -1, 1, 1, 1
  "    " (U+0009, 9, 0x09): 1 / 1, 1 / -1, 4, 4, 4
  "
" (U+000A, 10, 0x0A): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+000B, 11, 0x0B): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+000C, 12, 0x0C): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+000D, 13, 0x0D): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+000E, 14, 0x0E): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+000F, 15, 0x0F): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0010, 16, 0x10): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0011, 17, 0x11): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0012, 18, 0x12): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0013, 19, 0x13): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0014, 20, 0x14): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0015, 21, 0x15): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0016, 22, 0x16): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0017, 23, 0x17): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+0018, 24, 0x18): 1 / 1, 1 / -1, 1, 1, 1
  "" (U+0019, 25, 0x19): 1 / 1, 0 / -1, 1, 1, 0
  "" (U+001A, 26, 0x1A): 1 / 1, 0 / -1, 1, 1, 0
  "" (U+001B, 27, 0x1B): 1 / 1, 0 / -1, 1, 1, 0
  "" (U+001C, 28, 0x1C): 1 / 1, 0 / -1, 1, 1, 0
  " " (U+001D, 29, 0x1D): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+001E, 30, 0x1E): 1 / 1, 1 / -1, 1, 1, 1
  " " (U+001F, 31, 0x1F): 1 / 1, 1 / -1, 1, 1, 1
```

Current and new behavior (chars other than spaces are displayed with reverse video attribute):

Char                                | Old: chat     | Old: bars     | New: chat + bars
----------------------------------- | ------------- | ------------- | --------------------
U+0001 (1,  start of heading)       | 1 space       | `A`           | `A`
U+0002 (2,  start of text)          | 1 space       | `B`           | `B`
U+0003 (3,  end of text)            | 1 space       | `C`           | `C`
U+0004 (4,  end of transmission)    | 1 space       | `D`           | `D`
U+0005 (5,  enquiry)                | 1 space       | `E`           | `E`
U+0006 (6,  acknowledge)            | 1 space       | `F`           | `F`
U+0007 (7,  bell)                   | 1 space       | `G`           | `G`
U+0008 (8,  backspace)              | 1 space       | `H`           | `H`
U+0009 (9,  horizontal tab)         | N spaces      | `I`           | N spaces
U+000A (10, NL line feed, new line) | New line      | Item sep.     | New line / item sep.
U+000B (11, vertical tab)           | 1 space       | `K`           | `K`
U+000C (12, NP form feed, new page) | 1 space       | `L`           | `L`
U+000D (13, carriage return)        | 1 space       | New line      | `M` / New line
U+000E (14, shift out)              | 1 space       | `N`           | `N`
U+000F (15, shift in)               | 1 space       | `O`           | `O`
U+0010 (16, data link escape)       | 1 space       | `P`           | `P`
U+0011 (17, device control 1)       | 1 space       | `Q`           | `Q`
U+0012 (18, device control 2)       | 1 space       | `R`           | `R`
U+0013 (19, device control 3)       | 1 space       | `S`           | `S`
U+0014 (20, device control 4)       | 1 space       | `T`           | `T`
U+0015 (21, negative acknowledge)   | 1 space       | `U`           | `U`
U+0016 (22, synchronous idle)       | 1 space       | `V`           | `V`
U+0017 (23, end of trans. block)    | 1 space       | `W`           | `W`
U+0018 (24, cancel)                 | 1 space       | `X`           | `X`
U+0019 (25, end of medium)          | Not displayed | Not displayed | Not displayed
U+001A (26, substitute)             | Not displayed | Not displayed | Not displayed
U+001B (27, escape)                 | Not displayed | Not displayed | Not displayed
U+001C (28, file separator)         | Not displayed | Not displayed | Not displayed
U+001D (29, group separator)        | 1 space       | `]`           | `]`
U+001E (30, record separator)       | 1 space       | `^`           | `^`
U+001F (31, unit separator)         | 1 space       | `_`           | `_`

Notes:

- U+0009 (Tabulation):
  - The number of spaces follows the option `weechat.look.tab_width`.
  - Bug in bars: a single letter "🅸" is displayed, but the number of spaces
    configured in option `weechat.look.tab_width` is used to compute item length
    on screen, resulting in display issues.
  - Bug in bars: the reverse video is applied and disabled even if it was already
    enabled, and then it is disabled after the char while it should not be.
- U+0019 to U+001C:
  - WeeChat internal color codes and are never displayed as-is.

Expected unicode debug after changes:

```test
/debug unicode ${chars:${\u0001}-${\u001F}}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  "A" (U+0001, 1, 0x01): 1 / 1, 1 / -1, 1, 1, 1
  "B" (U+0002, 2, 0x02): 1 / 1, 1 / -1, 1, 1, 1
  "C" (U+0003, 3, 0x03): 1 / 1, 1 / -1, 1, 1, 1
  "D" (U+0004, 4, 0x04): 1 / 1, 1 / -1, 1, 1, 1
  "E" (U+0005, 5, 0x05): 1 / 1, 1 / -1, 1, 1, 1
  "F" (U+0006, 6, 0x06): 1 / 1, 1 / -1, 1, 1, 1
  "G" (U+0007, 7, 0x07): 1 / 1, 1 / -1, 1, 1, 1
  "H" (U+0008, 8, 0x08): 1 / 1, 1 / -1, 1, 1, 1
  "    " (U+0009, 9, 0x09): 1 / 1, 1 / -1, 4, 4, 4
  "
" (U+000A, 10, 0x0A): 1 / 1, 1 / -1, 1, 1, 1
  "K" (U+000B, 11, 0x0B): 1 / 1, 1 / -1, 1, 1, 1
  "L" (U+000C, 12, 0x0C): 1 / 1, 1 / -1, 1, 1, 1
  "M" (U+000D, 13, 0x0D): 1 / 1, 1 / -1, 1, 1, 1
  "N" (U+000E, 14, 0x0E): 1 / 1, 1 / -1, 1, 1, 1
  "O" (U+000F, 15, 0x0F): 1 / 1, 1 / -1, 1, 1, 1
  "P" (U+0010, 16, 0x10): 1 / 1, 1 / -1, 1, 1, 1
  "Q" (U+0011, 17, 0x11): 1 / 1, 1 / -1, 1, 1, 1
  "R" (U+0012, 18, 0x12): 1 / 1, 1 / -1, 1, 1, 1
  "S" (U+0013, 19, 0x13): 1 / 1, 1 / -1, 1, 1, 1
  "T" (U+0014, 20, 0x14): 1 / 1, 1 / -1, 1, 1, 1
  "U" (U+0015, 21, 0x15): 1 / 1, 1 / -1, 1, 1, 1
  "V" (U+0016, 22, 0x16): 1 / 1, 1 / -1, 1, 1, 1
  "W" (U+0017, 23, 0x17): 1 / 1, 1 / -1, 1, 1, 1
  "X" (U+0018, 24, 0x18): 1 / 1, 1 / -1, 1, 1, 1
  "" (U+0019, 25, 0x19): 1 / 1, 0 / -1, 1, 1, 0
  "" (U+001A, 26, 0x1A): 1 / 1, 0 / -1, 1, 1, 0
  "" (U+001B, 27, 0x1B): 1 / 1, 0 / -1, 1, 1, 0
  "" (U+001C, 28, 0x1C): 1 / 1, 0 / -1, 1, 1, 0
  "]" (U+001D, 29, 0x1D): 1 / 1, 1 / -1, 1, 1, 1
  "^" (U+001E, 30, 0x1E): 1 / 1, 1 / -1, 1, 1, 1
  "_" (U+001F, 31, 0x1F): 1 / 1, 1 / -1, 1, 1, 1
```

Note: the letters and symbols between double quotes are displayed with reverse video attribute.

### Char U+007F (127, delete)

Unicode debug:

```text
/debug unicode ${\u007F}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  " " (U+007F, 127, 0x7F): 1 / 1, 1 / -1, 1, 1, 1
```

Current and new behavior:

Char                 | Old: chat | Old: bars | New: chat + bars
-------------------- | --------- | --------- | ----------------
U+007F (127, delete) | 1 space   | 1 space   | Not displayed

Expected unicode debug after changes:

```text
/debug unicode ${\u007F}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  "" (U+007F, 127, 0x7F): 1 / 1, 1 / -1, -1, 0, 0
```

### Char U+0092 (146, private use two)

Unicode debug:

```text
/debug unicode ${\u0092}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  " " (U+0092, 146, 0xC2 0x92): 2 / 1, 1 / -1, 1, 1, 1
```

Current and new behavior:

Char                          | Old: chat | Old: bars | New: chat + bars
----------------------------- | --------- | --------- | ----------------
U+0092 (146, private use two) | 1 space   | 1 space   | Not displayed

Expected unicode debug after changes:

```text
/debug unicode ${\u0092}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  "" (U+0092, 146, 0xC2 0x92): 2 / 1, 1 / -1, -1, 0, 0
```

### Char U+00AD (173, soft hyphen)

Unicode debug:

```text
/debug unicode ${\u00AD}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  "­" (U+00AD, 173, 0xC2 0xAD): 2 / 1, 1 / 1, 1, 1, 1
```

This char is supposed to be displayed (`wcwidth` == 1), but as WeeChat
does not use it to break lines, it must be treated as a special character
and not displayed at all.

Current and new behavior:

Char                      | Old: chat | Old: bars | New: chat + bars
------------------------- | --------- | --------- | ----------------
U+00AD (173, soft hyphen) | Hyphen    | Hyphen    | Not displayed

Note: the hyphen displayed can also be a space or not displayed, according
to the terminal and font used.

Expected unicode debug after changes:

```text
/debug unicode ${\u00AD}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  "" (U+00AD, 173, 0xC2 0xAD): 2 / 1, 1 / 1, -1, 0, 0
```

### Char U+200B (8203, zero width space)

Unicode debug:

```text
/debug unicode ${\u200B}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  "​" (U+200B, 8203, 0xE2 0x80 0x8B): 3 / 1, 1 / 0, 0, 0, 0
```

This char is supposed to be displayed (`wcwidth` == 0), but in some cases causes
display issues, so it must be treated as a special character and not displayed at all.

For more information, see issue [#1770](https://github.com/weechat/weechat/issues/1770).

Current and new behavior:

Char                            | Old: chat | Old: bars | New: chat + bars
------------------------------- | --------- | --------- | ----------------
U+200B (8203, zero width space) | 1 space   | 1 space   | Not displayed

Note: the space may not be displayed or cause display issues, according
to the terminal and font used.

Expected unicode debug after changes:

```text
/debug unicode ${\u200B}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  "" (U+200B, 8203, 0xE2 0x80 0x8B): 3 / 1, 1 / 0, -1, 0, 0
```

### Other non printable chars

All other non printable chars (when `wcwidth` == -1) must not be displayed.

For example U+0085 (133, next line), unicode debug:

```text
/debug unicode ${\u0085}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  " " (U+0085, 133, 0xC2 0x85): 2 / 1, 1 / -1, 1, 1, 1
```

Current and new behavior:

Char                    | Old: chat | Old: bars | New: chat + bars
----------------------- | --------- | --------- | ----------------
U+0085 (133, next line) | Space     | Space     | Not displayed

Expected unicode debug after changes:

```text
/debug unicode ${\u0085}

Unicode: "char" (hex codepoint, codepoint, UTF-8 sequence): strlen / utf8_strlen, gui_chat_strlen / wcwidth, utf8_char_size_screen, utf8_strlen_screen, gui_chat_strlen_screen:
  "" (U+0085, 133, 0xC2 0x85): 2 / 1, 1 / -1, -1, 0, 0
```

## Functions

### utf8_char_size_screen

This function currently considers any non printable char (`wcwidth` == -1)
needs one column to be displayed, because we display them as a space.

The new behavior for non printable chars:

- char U+0009 (9, Tabulation): return value of option `weechat.look.tab_width`
- chars U+0001 (1) to U+001F (31), except U+0009 (Tabulation): return 1
- char U+00AD (173, soft hyphen): return -1 (consider it's non printable char, and it is not displayed)
- char U+200B (8203, zero width space): return -1 (consider it's non printable char, and it is not displayed)
- any other non printable char: return -1

So the function will return:

- `-1` for any non printable char (must not be displayed)
- `0` for any printable char that is not visible (for example Combining Diacritical Marks)
- `≥ 1` for any other char that is visible

### utf8_strlen_screen

This function must return the sum of `wcwidth` for all chars in the string.
Any char with `wcwidth` == -1 is considered as 0 column on screen.

This function has a bug: when the string has at least two unicode chars and
contains at least one non printable char (`wcwidth` == -1), then the result
is smaller than expected. For example `utf8_strlen_screen("abc\x01")` returns
1 instead of 4.

### gui_chat_utf_char_valid

Function is removed.

### gui_chat_char_size_screen

Function is removed.

### gui_chat_strlen_screen

This function skips WeeChat color codes, this is the only difference with
the function `utf8_strlen_screen`.

## Planning

The changes must be implemented in this order:

1. add output of `utf8_char_size_screen` in command `/debug unicode`
2. fix and optimize function `utf8_strlen_screen`
3. add API function `utf8_strncpy`
4. expand tabulations as spaces in bars
5. display chars < 32 with letter/symbol and reverse video attribute in chat
6. do not display non printable chars (chat + bars), including specific ones
   (soft hyphens, zero width spaces)

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/2022-003-fix-unicode-display.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/2022-003-fix-unicode-display.md)
