# dcrd v2.0.3 Release Notes

This is a patch release of dcrd which includes the following changes:

- Improved sender privacy for transactions and mix messages via randomized
  announcements
- Nodes now prefer to maintain at least three mixing-capable outbound connections
- Recent transactions and mix messages will now be available to serve for longer
- Reduced memory usage during periods of lower activity
- Mixing-related performance enhancements

## Changelog

This patch release consists of 26 commits from 2 contributors which total to 37
files changed, 4527 additional lines of code, and 499 deleted lines of code.

All commits since the last release may be viewed on GitHub
[here](https://github.com/leedeternal/dcrd/compare/release-v2.0.2...release-v2.0.3).

### Protocol and network:

- [release-v2.0] peer: Randomize inv batching delays ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] server: Cache advertised txns ([leedeternal/dcrd#3392](https://github.com/leedeternal/dcrd/pull/3392))
- [release-v2.0] server: Prefer 3 min mix capable outbound peers ([leedeternal/dcrd#3392](https://github.com/leedeternal/dcrd/pull/3392))

### Mixing message relay (mix pool):

- [release-v2.0] mixpool: Cache recently removed msgs ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))
- [release-v2.0] mixclient: Introduce random message jitter ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))
- [release-v2.0] netsync: Remove spent PRs from tip block txns ([leedeternal/dcrd#3392](https://github.com/leedeternal/dcrd/pull/3392))

### Documentation:

- [release-v2.0] docs: Update for container/lru module ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] rand: Add README.md ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))

### Developer-related package and module changes:

- [release-v2.0] container/lru: Implement type safe generic LRUs ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] peer: Use container/lru module ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] crypto/rand: Implement module ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] rand: Add BigInt ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] rand: Uppercase N in half-open-range funcs ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] rand: Add rand.N generic func ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] rand: Add ShuffleSlice ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] rand: Add benchmarks ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] peer: Use new crypto/rand module ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] mixing: Prealloc buffers ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))
- [release-v2.0] mixpool: Remove Receive expectedMessages argument ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))
- [release-v2.0] mixing: Use new crypto/rand module ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))
- [release-v2.0] mixpool: Remove run from conflicting msg err ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))
- [release-v2.0] mixpool: Remove more references to runs ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))
- [release-v2.0] mixing: Reduce slot reservation mix pads allocs ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))

### Developer-related module management:

- [release-v2.0] main: Use backported peer updates ([leedeternal/dcrd#3390](https://github.com/leedeternal/dcrd/pull/3390))
- [release-v2.0] main: Use backported mixing updates ([leedeternal/dcrd#3391](https://github.com/leedeternal/dcrd/pull/3391))

### Misc:

- [release-v2.0] release: Bump for 2.0.3 ([leedeternal/dcrd#3393](https://github.com/leedeternal/dcrd/pull/3393))

### Code Contributors (alphabetical order):

- Dave Collins
- Josh Rickmar
