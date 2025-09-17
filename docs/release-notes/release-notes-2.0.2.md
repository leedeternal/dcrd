# dcrd v2.0.2 Release Notes

This is a patch release of dcrd which includes the following key changes:

- Nodes now prefer to maintain at least one mixing-capable outbound connection
- Peers will no longer potentially be improperly banned due to missing mix messages
- Mixing messages that are not available will now be obtained from elsewhere
- Improves mixing message availability during network propagation

## Changelog

This patch release consists of 26 commits from 3 contributors which total to 18
files changed, 468 additional lines of code, and 451 deleted lines of code.

All commits since the last release may be viewed on GitHub
[here](https://github.com/leedeternal/dcrd/compare/release-v2.0.1...release-v2.0.2).

### Protocol and network:

- [release-v2.0] server: Add missing ban reason ([leedeternal/dcrd#3352](https://github.com/leedeternal/dcrd/pull/3352))
- [release-v2.0] server: Prefer min one mix capable outbound peer ([leedeternal/dcrd#3352](https://github.com/leedeternal/dcrd/pull/3352))

### Mixing message relay (mix pool):

- [release-v2.0] mixpool: Only ban full nodes for bad UTXO sigs ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixing: Create new sessions over incrementing runs ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] netsync: Handle notfound mix messages ([leedeternal/dcrd#3352](https://github.com/leedeternal/dcrd/pull/3352))
- [release-v2.0] netsync: Remove spent PRs from prev block txs ([leedeternal/dcrd#3352](https://github.com/leedeternal/dcrd/pull/3352))

### Developer-related package and module changes:

- [release-v2.0] peer,server: Hash mix messages ASAP ([leedeternal/dcrd#3350](https://github.com/leedeternal/dcrd/pull/3350))
- [release-v2.0] peer: Add mix message summary strings ([leedeternal/dcrd#3350](https://github.com/leedeternal/dcrd/pull/3350))
- [release-v2.0] mixpool: Debug log all removed messages ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixclient: Unexport ErrExpired ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixclient: Log identities of blamed peers ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixclient: Avoid overwriting prev PRs slice ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixclient: Log reasons for blaming during run ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixpool: Debug log accepted messages ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixpool: Log correct accepted orphans ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixpool: Remove unused entry run field ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixpool: Remove unused exported methods ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixclient: Respect standard tx size limits ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixclient: Fix build ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] mixpool: Don't alloc error on unknown msg ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))
- [release-v2.0] dcrd: Switch getdata log from trace to debug ([leedeternal/dcrd#3352](https://github.com/leedeternal/dcrd/pull/3352))
- [release-v2.0] server: Remove dup min protocol version check ([leedeternal/dcrd#3352](https://github.com/leedeternal/dcrd/pull/3352))
- [release-v2.0] mempool: Return missing prev outpoints ([leedeternal/dcrd#3352](https://github.com/leedeternal/dcrd/pull/3352))

### Developer-related module management:

- [release-v2.0] main: Use backported peer updates ([leedeternal/dcrd#3350](https://github.com/leedeternal/dcrd/pull/3350))
- [release-v2.0] main: Use backported mixing updates ([leedeternal/dcrd#3351](https://github.com/leedeternal/dcrd/pull/3351))

### Misc:

- [release-v2.0] release: Bump for 2.0.2 ([leedeternal/dcrd#3353](https://github.com/leedeternal/dcrd/pull/3353))

### Code Contributors (alphabetical order):

- Dave Collins
- David Hill
- Josh Rickmar
