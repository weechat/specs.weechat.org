# Practical semantic versioning

- Author: [Sébastien Helleu](https://github.com/flashcode)
- License: CC BY-NC-SA 4.0
- Created on: 2023-03-11
- Last updated: 2023-03-18
- Issue: [#1890](https://github.com/weechat/weechat/issues/1890): adopt a "practical" Semantic Versioning
- Status: implemented
- Target WeeChat version: 4.0.0

## Context

WeeChat uses two number for versions (X.Y) and sometimes three in case of severe bugs or security issues fixed in versions already released (X.Y.Z).
The numbers X and Y have same meaning, they are incremented in the same way.

## Goals

Purpose of this specification is to adopt a "practical" [Semantic Versioning](https://semver.org).\
As major breaking changes are introduced in version initially numbered 3.9, the goal is to bump to 4.0.0.\
All subsequent releases will follow this "practical" semantic versioning.

## Out of scope

The versions already released or fix releases based on versions already released are not concerned by these changes.

## Changes

### Version number

The version number is now always on three digits `X.Y.Z`, where:

- `X` is the major version
- `Y` is the minor version
- `Z` is the patch version.

The version follows the Semantic Versioning specification with some changes to be less strict, that's why it's called "practical" semantic versioning.

For example breaking changes in C API functions that are not widely used don't bump the major version number.

### Display of version

The version displayed by `weechat -v` is now always on three digits:

```text
$ weechat -v
4.0.0
```

Same for the output of command `/v` in WeeChat:

```text
WeeChat 4.0.0 [compiled on Mar 11 2023 10:58:35]
```

### Rules for version number

Rules to increment the version number:

- the **major version** number (`X`) is incremented only when intentional breaking changes target feature areas that are actively consumed by users, scripts or C plugin API
- the **minor version** number (`Y`) is incremented for any new release of WeeChat that includes new features and bug fixes, possibly breaking API with low impact on users
- the **patch version** number (`Z`) is reserved for releases that address severe bugs or security issues found after the release.

As WeeChat is constantly changing, new releases increment at least minor and sometimes major version number.\
The patch version is only made on maintenance branches (e.g. for releases done in past, not the future version).

Minor bugs found are not back-ported to old branches and are fixed only in the next release (new major and/or minor version).

### Major version number

These changes result in a bump of major version number:

- scripting API changes that breaks existing scripts and require changes
- removal of C or scripting API functions that are widely used
- major C API changes that breaks plugins and require a lot of changes
- incompatible formats that can not be automatically fixed on upgrade
- any changes that have high and direct impacts on users

Examples of changes that should have bumped major version number if semantic versioning was used:

- WeeChat 1.1:
  - new and incompatible format for regex replacement in triggers
- WeeChat 1.5:
  - add new pointer in many callbacks
- WeeChat 2.5:
  - `aspell` plugin renamed to `spell`
- WeeChat 2.9:
  - new background color for inactive bars (function bar_new updated)
  - new modifier_data for modifier "weechat_print"
- WeeChat 3.2:
  - support of XDG directories
- WeeChat 3.4:
  - new parameters in function hdata_search
- WeeChat 3.8:
  - lot of changes in string / UTF-8 functions
- WeeChat 3.9:
  - build with autotools removed
  - case sensitive identifiers
  - key bindings improvements

### Minor version number

These changes result in a bump of minor version number:

- add of new functions in C API or scripting API
- renaming of arguments in C or scripting API functions
- "small" breaking changes in C or scripting API functions that are not widely used, with low user impact.

Because of internal version numbering limitations, the max value for this minor version number is 255.\
In practice, as there are a lot of improvements in each WeeChat release, this number should rarely exceed 10 or 15.

### Patch version number

The patch version number is incremented only for severe bugs or security issues that are discovered on a version already released.

Because of internal version numbering limitations, the max value for this patch version number is 255.\
As most bugs are fixed in the next release, such versions are rare, and the patch version number should rarely exceed 1 or 2.

### Release process

After a release where major and/or minor have been bumped, the next version by default has minor number bumped.

For example after release of 4.0.0, the development version becomes 4.1.0-dev.

In case major breaking changes are introduced, the development version is immediately bumped to 5.0.0-dev.

In case a severe bug or a security issue must be fixed in 4.0.0, a new branch is created from v4.0.0 tag, and the version is set to 4.0.1-dev.\
The version on master branch remains 4.1.0-dev or 5.0.0-dev.

### Simulation of semantic versioning on old releases

If semantic version was used, versions would have been:

Version released | Would have been… | Major bumped because of…
---------------- | ---------------- | --------------------------------------------------------
1.0              | **1.0.0**        |
1.0.1            | 1.0.1            |
1.1              | **2.0.0**        | New trigger regex  format
1.1.1            | 2.0.1            |
1.2              | 2.1.0            |
1.3              | 2.2.0            |
1.4              | 2.3.0            |
1.5              | **3.0.0**        | New pointer in callbacks
1.6              | 3.1.0            |
1.7              | 3.2.0            |
1.7.1            | 3.2.1            |
1.8              | 3.3.0            |
1.9              | 3.4.0            |
1.9.1            | 3.4.1            |
2.0              | 3.5.0            |
2.0.1            | 3.5.1            |
2.1              | 3.6.0            |
2.2              | 3.7.0            |
2.3              | 3.8.0            |
2.4              | 3.9.0            |
2.5              | **4.0.0**        | Aspell renamed to Spell
2.6              | 4.1.0            |
2.7              | 4.2.0            |
2.7.1            | 4.2.1            |
2.8              | 4.3.0            |
2.9              | **5.0.0**        | Bars bg color (function bar_new), modifier weechat_print
3.0              | 5.1.0            |
3.0.1            | 5.1.1            |
3.1              | 5.2.0            |
3.2              | **6.0.0**        | XDG directories
3.2.1            | 6.0.1            |
3.3              | 6.1.0            |
3.4              | **7.0.0**        | Function hdata_search
3.4.1            | 7.0.1            |
3.5              | 7.1.0            |
3.6              | 7.2.0            |
3.7              | 7.3.0            |
3.7.1            | 7.3.1            |
3.8              | **8.0.0**        | String / UTF-8 functions
3.9              | **9.0.0**        | Autotools removed, case sensitive identifiers, new keys

### Breaking changes

To prevent incrementing major version too often, the major breaking changes will be grouped together.

For example when the major version is bumped, some deprecated functions can be removed (especially those marked as deprecated a long time ago).

## Planning

The changes must be implemented in this order:

1. Rename version 3.9 to 4.0.0.
2. Add note about semantic versioning in Contributing.adoc.

## References

- Source of this specification: [https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-003-practical-semantic-versioning.md](https://github.com/weechat/specs.weechat.org/blob/main/specs/2023-003-practical-semantic-versioning.md)
