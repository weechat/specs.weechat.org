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
using two new bar items called `spacer` and `spacer1`.

## Out of scope

Buffers are out of scope, alignment is only for bars.

## Implementation

### New bar items

Two new bar items are added:

* `spacer` which has a min size of 0 (nothing displayed)
* `spacer1` which has a min size of 1 (one space); it must be used inside items
  to be sure at least one space is displayed; basically `spacer1` is just
  a space followed by a `spacer` which has a min size of 0.

The items `spacer` and `spacer1` can be used multiple times in the same bar.

WeeChat uses the same size (or almost) for all spacers in the bar. The chosen
size is the highest possible so that bar items uses full bar width
(see [Algorithm](#algorithm)).

When the bar is not large enough for all items (not counting the spacers),
the size is set to 0 for `spacer` items and 1 for `spacer1` items
(see [Wide items](#wide-items)).

The `spacer` and `spacer1` items can be separated from other items by a comma
or `+` (glued items), this doesn't matter: WeeChat never adds extra spaces
around the spacer.

### Restrictions

Using bar items `spacer` and `spacer1` is allowed only in bars that meet **all**
these conditions:

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

As the bar items `spacer` and `spacer1` can be used multiple times in the same bar,
you can combine left, center and right alignment.

Example with 2 spacers, using `spacer1` to be sure at least one space is displayed,
even when the bar width is too small (see [Wide items](#wide-items)):

````
 bar width = 62
 items = [time],spacer1,buffer_number+:+buffer_name,spacer1,[buffer_last_number]
 ▼
┌──────────────────────────────────────────────────────────────┐
│[12:34]                     3:#weechat                     [9]│
└──────────────────────────────────────────────────────────────┘
````

Example with 3 spacers:

````
 bar width = 62
 items = [time],spacer1,[buffer_plugin],spacer1,buffer_number+:+buffer_name,spacer1,[buffer_last_number]
 ▼
┌──────────────────────────────────────────────────────────────┐
│[12:34]          [irc/libera]          3:#weechat          [9]│
└──────────────────────────────────────────────────────────────┘
````

Example with 5 spacers, mixing `spacer` (beginning/end) and `spacer1` (inside items):

````
 bar width = 62
 items = spacer,[time],spacer1,[buffer_plugin],spacer1,buffer_number+:+buffer_name,spacer1,[buffer_last_number],spacer
 ▼
┌──────────────────────────────────────────────────────────────┐
│      [12:34]      [irc/libera]      3:#weechat      [9]      │
└──────────────────────────────────────────────────────────────┘
````

### Wide items

If the displayed length of items without spacers is equal or greater than
bar width, WeeChat sets size 0 for all `spacer` items and size 1 for all `spacer1`
items.

Example with enough space in bar, all spacers are displayed:

```
 bar width = 62
 items = spacer,[time],spacer1,buffer_number+:+buffer_name,[buffer_last_number],spacer
 ▼
┌──────────────────────────────────────────────────────────────┐
│             [12:34]              3:#weechat [9]              │
└──────────────────────────────────────────────────────────────┘
```

When displayed length is equal or greater than bar width, `spacer` items are not
displayed and `spacer1` is displayed with one space (between time and buffer number):

````
 bar width = 22
 items = spacer,[time],spacer1,buffer_number+:+buffer_name,[buffer_last_number],spacer
 ▼
┌──────────────────────┐
│[12:34] 3:#weechat [9]│
└──────────────────────┘

 bar width = 18
 items = spacer,[time],spacer1,buffer_number+:+buffer_name,[buffer_last_number],spacer
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
   `spacer` items are ignored and `spacer1` are displayed with one space.
2. Else:
     1. Count the number of spacers in the bar (`num_spacers`).
     2. Compute the total displayed length of the items assuming `spacer` have
        a size of 0 and `spacer` a size of 1 (`display_length_without_spacers`).
     3. If `display_length_without_spacers` ≥ `bar_width`, then:
          1. Set size 0 for all `spacer` items.
          2. Set size 1 for all `spacer1` items.
     4. Else:
          1. Compute the size of a spacer, as integer ≥ 0:\
             `spacer_size` = (`bar_width` - `display_length_without_spacers`) / `num_spacers`
          2. Compute display length with spacers:\
             `display_length_with_spacers` = `display_length_without_spacers` + (`num_spacers` * `spacer_size`)
          3. If `display_length_with_spacers` < `bar_width`:
               1. Loop on spacers until `display_length_with_spacers` ≥ `bar_width` and for each spacer:
                    1. Add `1` to spacer size.
                    2. Add `1` to `display_length_with_spacers`.
     5. Replace each spacer by the computed number of spaces.

Once the size of each spacer is computed, the bar can be displayed on screen.

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/001700-bar-spacer.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/001700-bar-spacer.md)
- WeeChat issue on GitHub: [https://github.com/weechat/weechat/issues/1700](https://github.com/weechat/weechat/issues/1700)
