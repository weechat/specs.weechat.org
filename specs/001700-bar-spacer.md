# Spacer item in bars

- Author: [Sébastien Helleu](https://github.com/flashcode)
- Created on: 2022-04-09
- Last updated: 2022-04-17
- Issue: [#1700](https://github.com/weechat/weechat/issues/1700)
- Status: draft
- Target WeeChat version: 3.6

## Context

Bar items are currently displayed from left to right with no way to align them
or add extra dynamic spaces between items.

## Goals

Purpose of this specification is to introduce an alignment mechanism for bars
using a new bar item called `spacer`.

## Out of scope

Buffers are out of scope, alignment is only for bars.

## Implementation

### New bar item

A new bar item called `spacer` is added.\
It can be used multiple times in the same bar, and is displayed with 0 to N
spaces on screen.\
WeeChat uses the same size (or almost) for all spacers in the bar. The chosen
size is the highest possible so that bar items uses full bar width
(see [Algorithm](#algorithm)).

When the bar is not large enough for all items (not counting the spacers),
spacers are not displayed (see [Wide items](#wide-items)).

### Restrictions

Using bar item `spacer` is allowed only in bars that meet **all** these conditions:

* type: `window` or `root`
* position: `top` or `bottom`
* filling: `horizontal`
* size (height): 1

A bar size of `0` (automatic) or any value higher than 1 is not supported.\
Consequently, the default bars `status` and `title` support alignment but **not**
`input` which has a default size of `0` (automatic).

### Left alignment

Bars are still aligned on the left by default, so adding a spacer at the end
of items is useless.

Example without any spacer (recommended for left alignment):

````
 bar width = 62
 items = [time],buffer_number+:+buffer_name,[buffer_last_number]
 ▼
┌──────────────────────────────────────────────────────────────┐
│[12:34] 3:#weechat [9]                                        │
└──────────────────────────────────────────────────────────────┘
````

Example with a useless trailing spacer (**NOT RECOMMENDED** as this is slower
to display, WeeChat has to compute the size of spacer, which is useless here):

````
 bar width = 62
 items = [time],buffer_number+:+buffer_name,[buffer_last_number],spacer
 ▼
┌──────────────────────────────────────────────────────────────┐
│[12:34] 3:#weechat [9]                                        │
└──────────────────────────────────────────────────────────────┘
````

### Right alignment

Adding a `spacer` before items makes them right-aligned.

Example:

````
 bar width = 62
 items = spacer,[time],buffer_number+:+buffer_name,[buffer_last_number]
 ▼
┌──────────────────────────────────────────────────────────────┐
│                                        [12:34] 3:#weechat [9]│
└──────────────────────────────────────────────────────────────┘
````

### Centering

Adding a `spacer` before and after items makes them centered.

Example:

````
 bar width = 62
 items = spacer,[time],buffer_number+:+buffer_name,[buffer_last_number],spacer
 ▼
┌──────────────────────────────────────────────────────────────┐
│                    [12:34] 3:#weechat [9]                    │
└──────────────────────────────────────────────────────────────┘
````

Multiple consecutive spacers can be used, so that it uses more space, for example
3 spacers on the left, a single on the right:

````
 bar width = 62
 items = spacer,spacer,spacer,[time],buffer_number+:+buffer_name,[buffer_last_number],spacer
 ▼
┌──────────────────────────────────────────────────────────────┐
│                              [12:34] 3:#weechat [9]          │
└──────────────────────────────────────────────────────────────┘
````

### Combination

You can combine left, center and right alignment by using spacer at the beginning,
between items and at the end.

Example with 2 spacers:

````
 bar width = 62
 items = [time],spacer,buffer_number+:+buffer_name,spacer,[buffer_last_number]
 ▼
┌──────────────────────────────────────────────────────────────┐
│[12:34]                     3:#weechat                     [9]│
└──────────────────────────────────────────────────────────────┘
````

Example with 3 spacers:

````
 bar width = 62
 items = [time],spacer,[buffer_plugin],spacer,buffer_number+:+buffer_name,spacer,[buffer_last_number]
 ▼
┌──────────────────────────────────────────────────────────────┐
│[12:34]          [irc/libera]          3:#weechat          [9]│
└──────────────────────────────────────────────────────────────┘
````

Example with 5 spacers:

````
 bar width = 62
 items = spacer,[time],spacer,[buffer_plugin],spacer,buffer_number+:+buffer_name,spacer,[buffer_last_number],spacer
 ▼
┌──────────────────────────────────────────────────────────────┐
│      [12:34]      [irc/libera]      3:#weechat      [9]      │
└──────────────────────────────────────────────────────────────┘
````

### Wide items

If the displayed length of items without spacers is equal or greater than
bar width, WeeChat doesn't display any spacer.

Example with enough space in bar, the 3 spacers are displayed:

```
 bar width = 62
 items = spacer,[time],spacer,buffer_number+:+buffer_name,[buffer_last_number],spacer
 ▼
┌──────────────────────────────────────────────────────────────┐
│             [12:34]              3:#weechat [9]              │
└──────────────────────────────────────────────────────────────┘
```

When displayed length is equal or greater than bar width, spacers are not displayed:

````
 bar width = 22
 items = spacer,[time],spacer,buffer_number+:+buffer_name,[buffer_last_number],spacer
 ▼
┌──────────────────────┐
│[12:34] 3:#weechat [9]│
└──────────────────────┘

 bar width = 18
 items = spacer,[time],spacer,buffer_number+:+buffer_name,[buffer_last_number],spacer
 ▼
┌──────────────────┐
│[12:34] 3:#weech>>│
└──────────────────┘
````

### Algorithm

The variables are used in the following algorithm:

* `bar_width`: width of bar (in chars)
* `num_spacers`: number of `spacer` items in the bar (option `weechat.bar.xxx.items`)
* `display_length_without_spacers`: the length (in chars) of items displayed
  on screen without the `spacer` items
* `display_length_with_spacers`: the length (in chars) of items displayed
  on screen including the `spacer` items

This algorithm is used to compute the size of spacers:

1. If the bar does not meet conditions to use `spacer` item (see [Restrictions](#restrictions)),
   remove all spacers.
2. Else:
     1. Count the number of spacers in the bar (`num_spacers`).
     2. Compute the total displayed length of the items, ignoring spacers
        (`display_length_without_spacers`).
     3. If `display_length_without_spacers` ≥ `bar_width`, then remove all spacers.
     4. Else:
          1. Compute the size of a spacer, as integer ≥ 0:\
             `spacer_size` = (`bar_width` - `display_length_without_spacers`) / `num_spacers`
          2. Compute display length with spacers:\
             `display_length_with_spacers` = `display_length_without_spacers` + (`num_spacers` * `spacer_size`)
          3. If `display_length_with_spacers` < `bar_width`, loop on spacers
             until `display_length_with_spacers` ≥ `bar_width` and for each spacer:
               1. Add `1` to spacer size.
               2. Add `1` to `display_length_with_spacers`.
          4. Replace each spacer by the computed number of spaces.
3. Display the bar.

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/001700-bar-spacer.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/001700-bar-spacer.md)
- WeeChat issue on GitHub: [https://github.com/weechat/weechat/issues/1700](https://github.com/weechat/weechat/issues/1700)
