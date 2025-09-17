# dcrd v2.0.4 Release Notes

This is a patch release of dcrd which includes the following changes:

- Improved session formation for StakeShuffle mix transactions
- Support for Internationalized Domain Names (IDNs) in hostnames
- StakeShuffle mixing performance enhancements

## Changelog

This patch release consists of 14 commits from 3 contributors which total to 17
files changed, 201 additional lines of code, and 97 deleted lines of code.

All commits since the last release may be viewed on GitHub
[here](https://github.com/leedeternal/dcrd/compare/release-v2.0.3...release-v2.0.4).

### Mixing message relay (mix pool):

- [release-v2.0] mixpool: Reject KEs submitted too early ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] mixclient: Use newest (fewest-PR) KEs to form alt sessions ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))

### RPC / gencerts utility changes:

- [release-v2.0] certgen,gencerts: Punycode non-ASCII hostnames ([leedeternal/dcrd#3432](https://github.com/leedeternal/dcrd/pull/3432))

### Developer-related package and module changes:

- [release-v2.0] mixclient: Remove completely unused var ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] mixpool: Remove error which is always returned nil ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] mixclient: Dont append to slice with non-zero length ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] mixing: Add missing copyright headers ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] mixclient: Add missing copyright headers ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] mixclient: Remove submit queue channel ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] mixclient: Do not submit PRs holding client mutex ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))

### Developer-related module management:

- [release-v2.0] main: Use backported mixing updates ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] main: Use backported certgen updates ([leedeternal/dcrd#3432](https://github.com/leedeternal/dcrd/pull/3432))

### Misc:

- [release-v2.0] mixing: Add missing periods to comments ([leedeternal/dcrd#3431](https://github.com/leedeternal/dcrd/pull/3431))
- [release-v2.0] release: Bump for 2.0.4 ([leedeternal/dcrd#3433](https://github.com/leedeternal/dcrd/pull/3433))

### Code Contributors (alphabetical order):

- Dave Collins
- Jamie Holdstock
- Josh Rickmar
