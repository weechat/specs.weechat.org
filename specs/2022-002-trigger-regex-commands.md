# Trigger regex commands

- Author: [Sébastien Helleu](https://github.com/flashcode)
- Created on: 2022-10-02
- Last updated: 2022-11-06
- Issue: [#1510](https://github.com/weechat/weechat/issues/1510)
- Status: implemented
- Target WeeChat version: 3.8

## Context

The regular expression used in triggers allows only replacement.

## Goals

Purpose of this specification is to add more commands and support a new
one which is translation of chars, similar to the command `y` of the `sed`
UN*X command.

## Implementation

### API function string_translate_chars

To be able to translate chars in the Trigger plugin, a new API function is
added (in C API only) with this prototype:

```C
char *string_translate_chars (const char *string, const char *chars1, const char *chars2);
```

Parameters:

- `string`: the string
- `chars1`: chars to translate
- `chars2`: replacement chars (must contain the same number of UTF-8 chars than `chars1`)

Returns:

- the `string` with translated chars: each char of string present in `chars1`
  is replaced with the char of `chars2` at the same position.

Examples:

```C
str = string_translate_chars ("test", "abcdef", "ABCDEF");  /* str == "tEst" */
str = string_translate_chars ("clean the boat", "abc", "ABC");  /* str == "CleAn the BoAt" */
```

### Range of chars in evaluated strings

As replacing case or custom range of chars could also be useful, a new evaluated
format is added to evaluated strings: `${chars:xxx}`, where `xxx` is one of:

- `digit`: all digits → `0123456789`
- `xdigit`: all hexadecimal digits → `0123456789abcdefABCDEF`
- `lower`: all lower case letters → `abcdefghijklmnopqrstuvwxyz`
- `upper`: all upper case letters → `ABCDEFGHIJKLMNOPQRSTUVWXYZ`
- `alpha`: all letters → `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ`
- `alnum`: all letters and digits → `abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789`
- a custom range with format `CHAR1-CHAR2` where `CHAR1` and `CHAR2` are single UTF-8 chars, with following restrictions:
  - code point of `CHAR1` must be lower or equal to code point of `CHAR2`
  - a range with max 4096 chars can be generated (with more chars, an empty string is returned)

Examples:

```text
>> ${chars:digit}
== 0123456789

>> ${chars:a-h}
== abcdefgh

${chars:J-V}
== JKLMNOPQRSTUV

${chars:é-é}
== é

${chars:←-↓}
== ←↑→↓

${chars:▁-▏}
== ▁▂▃▄▅▆▇█▉▊▋▌▍▎▏
```

### New trigger regex format

The first char of the regex is used to know the command.

If the first char is a letter (lower or upper case, in range a-z and A-Z),
this is used as the command, and the next char is the regex separator.

If the regex starts with a letter that is not a supported command (for now
different from `s` and `y`), an error is displayed and the trigger is not created.

Important: if you upgrade from an older version of WeeChat, be sure all your
triggers are valid, otherwise they'll be automatically ignored and then removed.

If the first char is not a letter, it is used as regex separator, and the
command is regex replacement.

The default action is "s" for regex replacement.

#### Backward compatibility

If the first char is not a letter, the regex works with any WeeChat version,
the default action being regex replacement, for example:

```text
/bug([0-9]+)/bug #${re:1}/
```

This example forces regex replacement (works only with WeeChat ≥ 3.8):

```text
s/bug([0-9]+)/bug #${re:1}/
```

Consequently, any trigger using a letter as separator will not work any more
and the "s" command must be forced.

For example such regex doesn't work any more:

```text
XabcXdefX
```

And must be converted to:

```text
sXabcXdefX
```

Note: a separator like `/` is better for readability anyway:

```text
/abc/def/
```

#### New command for translation

A new command `y` is added for chars translations.

It takes two strings: each char in the first string is replaced by the char
in the second string (at the same place).\
The two strings are evaluated when the trigger runs, and once evaluated, they
must contain the same number of UTF-8 chars.

Examples:

```text
# replace "a", "b" and "c" by upper case letter
y/abc/ABC/

# rotate arrows clockwise
y/←↑→↓/↑→↓←/

# convert all letters to lower case
y/${chars:upper}/${chars:lower}/

# shift each letter by one position, preserving case: a→b, b→c … y→z, z→a
y/${chars:a-z}${chars:A-Z}/${chars:b-z}a${chars:B-Z}A/
```

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/001510-trigger-regex-commands.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/001510-trigger-regex-commands.md)
- WeeChat issue on GitHub: [https://github.com/weechat/weechat/issues/1510](https://github.com/weechat/weechat/issues/1510)
