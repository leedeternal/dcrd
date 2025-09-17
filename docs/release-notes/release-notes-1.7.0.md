# dcrd v1.7.0

This is a new major release of dcrd.  Some of the key highlights are:

* Four new consensus vote agendas which allow stakeholders to decide whether or
  not to activate support for the following:
  * Reverting the Treasury maximum expenditure policy
  * Enforcing explicit version upgrades
  * Support for automatic ticket revocations for missed votes
  * Changing the Proof-of-Work and Proof-of-Stake subsidy split from 60%/30% to 10%/80%
* Substantially reduced initial sync time
* Major performance enhancements to unspent transaction output handling
* Faster cryptographic signature validation
* Significant improvements to network synchronization
* Support for a configurable assumed valid block
* Block index memory usage reduction
* Asynchronous indexing
* Version 1 block filters removal
* Various updates to the RPC server:
  * Additional per-connection read limits
  * A more strict cross origin request policy
  * A new alternative client authentication mechanism based on TLS certificates
  * Availability of the scripting language version for transaction outputs
  * Several other notable updates, additions, and removals related to the JSON-RPC API
* New developer modules:
  * Age-Partitioned Bloom Filters
  * Fixed-Precision Unsigned 256-bit Integers
  * Standard Scripts
  * Standard Addresses
* Infrastructure improvements
* Quality assurance changes

For those unfamiliar with the
[voting process](https://docs.decred.org/governance/consensus-rule-voting/overview/)
in Decred, all code needed in order to support each of the aforementioned
consensus changes is already included in this release, however it will remain
dormant until the stakeholders vote to activate it.

For reference, the consensus change work for each of the four changes was
originally proposed and approved for initial implementation via the following
Politeia proposals:
- [Decentralized Treasury Spending](https://proposals-archive.decred.org/proposals/c96290a)
- [Explicit Version Upgrades Consensus Change](https://proposals.decred.org/record/3a98861)
- [Automatic Ticket Revocations Consensus Change](https://proposals.decred.org/record/e2d7b7d)
- [Change PoW/PoS Subsidy Split From 60/30 to 10/80](https://proposals.decred.org/record/427e1d4)

The following Decred Change Proposals (DCPs) describe the proposed changes in
detail and provide full technical specifications:
- [DCP0007](https://github.com/leedeternal/dcps/blob/master/dcp-0007/dcp-0007.mediawiki)
- [DCP0008](https://github.com/leedeternal/dcps/blob/master/dcp-0008/dcp-0008.mediawiki)
- [DCP0009](https://github.com/leedeternal/dcps/blob/master/dcp-0009/dcp-0009.mediawiki)
- [DCP0010](https://github.com/leedeternal/dcps/blob/master/dcp-0010/dcp-0010.mediawiki)

## Upgrade Required

**It is extremely important for everyone to upgrade their software to this
latest release even if you don't intend to vote in favor of the agenda.  This
particularly applies to PoW miners as failure to upgrade will result in lost
rewards after block height 635775.  That is estimated to be around Feb 21st,
2022.**

## Downgrade Warning

The database format in v1.7.0 is not compatible with previous versions of the
software.  This only affects downgrades as users upgrading from previous
versions will see a one time database migration.

Once this migration has been completed, it will no longer be possible to
downgrade to a previous version of the software without having to delete the
database and redownload the chain.

The database migration typically takes around 40-50 minutes on HDDs and 20-30
minutes on SSDs.

## Notable Changes

### Four New Consensus Change Votes

Four new consensus change votes are now available as of this release.  After
upgrading, stakeholders may set their preferences through their wallet.

#### Revert Treasury Maximum Expenditure Policy Vote

The first new vote available as of this release has the id `reverttreasurypolicy`.

The primary goal of this change is to revert the currently active maximum
expenditure policy of the decentralized Treasury to the one specified in the
[original Politeia proposal](https://proposals-archive.decred.org/proposals/c96290a).

See [DCP0007](https://github.com/leedeternal/dcps/blob/master/dcp-0007/dcp-0007.mediawiki) for
the full technical specification.

#### Explicit Version Upgrades Vote

The second new vote available as of this release has the id `explicitverupgrades`.

The primary goals of this change are to:

* Provide an easy, reliable, and efficient method for software and hardware to
  determine exactly which rules should be applied to transaction and script
  versions
* Further embrace the increased security and other desirable properties that
  hard forks provide over soft forks

See the following for more details:

* [Politeia proposal](https://proposals.decred.org/record/3a98861)
* [DCP0008](https://github.com/leedeternal/dcps/blob/master/dcp-0008/dcp-0008.mediawiki)

#### Automatic Ticket Revocations Vote

The third new vote available as of this release has the id `autorevocations`.

The primary goals of this change are to:

* Improve the Decred stakeholder user experience by removing the requirement for
  stakeholders to manually revoke missed and expired tickets
* Enable the recovery of funds for users who lost their redeem script for the
  legacy VSP system (before the release of vspd, which removed the need for the
  redeem script)

See the following for more details:

* [Politeia proposal](https://proposals.decred.org/record/e2d7b7d)
* [DCP0009](https://github.com/leedeternal/dcps/blob/master/dcp-0009/dcp-0009.mediawiki)

#### Change PoW/PoS Subsidy Split to 10/80 Vote

The fourth new vote available as of this release has the id `changesubsidysplit`.

The proposed modification to the subsidy split is intended to substantially
diminish the ability to attack Decred's markets with mined coins and improve
decentralization of the issuance process.

See the following for more details:

* [Politeia proposal](https://proposals.decred.org/record/427e1d4)
* [DCP0010](https://github.com/leedeternal/dcps/blob/master/dcp-0010/dcp-0010.mediawiki)

### Substantially Reduced Initial Sync Time

The amount of time it takes to complete the initial chain synchronization
process has been substantially reduced.  With default settings, it is around 48%
faster versus the previous release.

### Unspent Transaction Output Overhaul

The way unspent transaction outputs (UTXOs) are handled has been significantly
reworked to provide major performance enhancements to both steady-state
operation as well as the initial chain sync process as follows:

* Each UTXO is now tracked independently on a per-output basis
* The UTXOs now reside in a dedicated database
* All UTXO reads and writes now make use of a cache

#### Unspent Transaction Output Cache

All reads and writes of unspent transaction outputs (utxos) now go through a
cache that sits on top of the utxo set database which drastically reduces the
amount of reading and writing to disk, especially during the initial sync
process when a very large number of blocks are being processed in quick
succession.

This utxo cache provides significant runtime performance benefits at the cost of
some additional memory usage.  The maximum size of the cache can be configured
with the new `--utxocachemaxsize` command-line configuration option.  The
default value is 150 MiB, the minimum value is 25 MiB, and the maximum value is
32768 MiB (32 GiB).

Some key properties of the cache are as follows:

* For reads, the UTXO cache acts as a read-through cache
  * All UTXO reads go through the cache
  * Cache misses load the missing data from the disk and cache it for future lookups
* For writes, the UTXO cache acts as a write-back cache
  * Writes to the cache are acknowledged by the cache immediately, but are only
    periodically flushed to disk
* Allows intermediate steps to effectively be skipped thereby avoiding the need
  to write millions of entries to disk
* On average, recent UTXOs are much more likely to be spent in upcoming blocks
  than older UTXOs, so only the oldest UTXOs are evicted as needed in order to
  maximize the hit ratio of the cache
* The cache is periodically flushed with conditional eviction:
  * When the cache is NOT full, nothing is evicted, but the changes are still
    written to the disk set to allow for a quicker reconciliation in the case of
    an unclean shutdown
  * When the cache is full, 15% of the oldest UTXOs are evicted

### Faster Cryptographic Signature Validation

Some aspects of the underlying crypto code has been updated to further improve
its execution speed and reduce the number of memory allocations resulting in
about a 1% reduction to signature verification time.

The primary benefits are:

* Improved vote times since blocks and transactions propagate more quickly
  throughout the network
* Approximately a 2% reduction to the duration of the initial sync process

### Significant Improvements to Network Synchronization

The method used to obtain blocks from other peers on the network is now guided
entirely by block headers.  This provides a wide variety of benefits, but the
most notable ones for most users are:

* Faster initial synchronization
* Reduced bandwidth usage
* Enhanced protection against attempted DoS attacks
* Percentage-based progress reporting
* Improved steady state logging

### Support for Configurable Assumed Valid Block

This release introduces a new model for deciding when several historical
validation checks may be skipped for blocks that are an ancestor of a known good
block.

Specifically, a new `AssumeValid` parameter is now used to specify the
aforementioned known good block.  The default value of the parameter is updated
with each release to a recent block that is part of the main chain.

The default value of the parameter can be overridden with the `--assumevalid`
command-line option by setting it as follows:

* `--assumevalid=0`: Disable the feature resulting in no skipped validation checks
* `--assumevalid=[blockhash]`:  Set `AssumeValid` to the specified block hash

Specifying a block hash closer to the current best chain tip allows for faster
syncing.  This is useful since the validation requirements increase the longer a
particular release build is out as the default known good block becomes deeper
in the chain.

### Block Index Memory Usage Reduction

The block index that keeps track of block status and connectivity now occupies
around 30MiB less memory and scales better as more blocks are added to the
chain.

### Asynchronous Indexing

The various optional indexes are now created asynchronously versus when
blocks are processed as was previously the case.

This permits blocks to be validated more quickly when the indexes are enabled
since the validation no longer needs to wait for the indexing operations to
complete.

In order to help keep consistent behavior for RPC clients, RPCs that involve
interacting with the indexes will not return results until the associated
indexing operation completes when the indexing tip is close to the current best
chain tip.

One side effect of this change that RPC clients should be aware of is that it is
now possible to receive sync timeout errors on RPCs that involve interacting
with the indexes if the associated indexing tip gets so far behind it would end
up delaying results for too long.  In practice, errors of this type are rare and
should only ever be observed during the initial sync process before the
associated indexes are current.  However, callers should be aware of the
possibility and handle the error accordingly.

The following RPCs are affected:

* `existsaddress`
* `existsaddresses`
* `getrawtransaction`
* `searchrawtransactions`

### Version 1 Block Filters Removal

The previously deprecated version 1 block filters are no longer available on the
peer-to-peer network.  Use
[version 2 block filters](https://github.com/leedeternal/dcps/blob/master/dcp-0005/dcp-0005.mediawiki#version-2-block-filters)
with their associated
[block header commitment](https://github.com/leedeternal/dcps/blob/master/dcp-0005/dcp-0005.mediawiki#block-header-commitments)
and [inclusion proof](https://github.com/leedeternal/dcps/blob/master/dcp-0005/dcp-0005.mediawiki#verifying-commitment-root-inclusion-proofs)
instead.

### RPC Server Changes

The RPC server version as of this release is 7.0.0.

#### Max Request Limits

The RPC server now imposes the following additional per-connection read limits
to help further harden it against potential abuse in non-standard configurations
on poorly-configured networks:

* 0 B / 8 MiB for pre and post auth HTTP connections
* 4 KiB / 16 MiB for pre and post auth WebSocket connections

In practice, these changes will not have any noticeable effect for the vast
majority of nodes since the RPC server is not publicly accessible by default and
also requires authentication.

Nevertheless, it can still be useful for scenarios such as authenticated fuzz
testing and improperly-configured networks that have disabled all other security
measures.

#### More Strict Cross Origin Request (CORS) Policy

The CORS policy for WebSocket clients is now more strict and rejects requests
from other domains.

In practice, CORS requests will be rejected before ever reaching that point due
to the use of a self-signed TLS certificate and the requirement for
authentication to issue any commands.  However, additional protection mechanisms
make it that much more difficult to attack by providing defense in depth.

#### Alternative Client Authentication Method Based on TLS Certificates

A new alternative method for TLS clients to authenticate to the RPC server by
presenting a client certificate in the TLS handshake is now available.

Under this authentication method, the certificate authority for a client
certificate must be added to the RPC server as a trusted root in order for it to
trust the client.  Once activated, clients will no longer be required to provide
HTTP Basic authentication nor use the `authenticate` RPC in the case of
WebSocket clients.

Note that while TLS client authentication has the potential to ultimately allow
more fine grained access controls on a per-client basis, it currently only
supports clients with full administrative privileges.  In other words, it is not
currently compatible with the `--rpclimituser` and `--rpclimitpass` mechanism,
so users depending on the limited user settings should avoid the new
authentication method for now.

The new authentication type can be activated with the `--authtype=clientcert`
configuration option.

By default, the trusted roots are loaded from the `clients.pem` file in dcrd's
application data directory, however, that location can be modified via the
`--clientcafile` option if desired.

#### Updates to Transaction Output Query RPC (`gettxout`)

The `gettxout` RPC has the following modifications:

* An additional `tree` parameter is now required in order to explicitly identify
  the exact transaction output being requested
* The transaction `version` field is no longer available in the primary JSON
  object of the results
* The child `scriptPubKey` JSON object in the results now includes a new
  `version` field that identifies the scripting language version

See the
[gettxout JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#gettxout)
for API details.

#### Removal of Stake Difficulty Notification RPCs (`notifystakedifficulty` and `stakedifficulty`)

The deprecated `notifystakedifficulty` and `stakedifficulty` WebSocket-only RPCs
are no longer available.  This notification is unnecessary since the difficulty
change interval is well defined.  Callers may obtain the difficulty via
`getstakedifficulty` at the appropriate difficulty change intervals instead.

See the
[getstakedifficulty JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getstakedifficulty)
for API details.

#### Removal of Version 1 Filter RPCs (`getcfilter` and `getcfilterheader`)

The deprecated `getcfilter` and `getcfilterheader` RPCs, which were previously
used to obtain version 1 block filters via RPC are no longer available. Use
`getcfilterv2` instead.

See the
[getcfilterv2 JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getcfilterv2)
for API details.

#### New Median Time Field on Block Query RPCs (`getblock` and `getblockheader`)

The verbose results of the `getblock` and `getblockheader` RPCs now include a
`mediantime` field that specifies the median block time associated with the
block.

See the following for API details:

* [getblock JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getblock)
* [getblockheader JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getblockheader)

#### New Scripting Language Version Field on Raw Transaction RPCs (`getrawtransaction`, `decoderawtransaction`, `searchrawtransactions`, and `getblock`)

The verbose results of the `getrawtransaction`, `decoderawtransaction`,
`searchrawtransactions`, and `getblock` RPCs now include a `version` field in
the child `scriptPubKey` JSON object that identifies the scripting language
version.

See the following for API details:

* [getrawtransaction JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getrawtransaction)
* [decoderawtransaction JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#decoderawtransaction)
* [searchrawtransactions JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#searchrawtransactions)
* [getblock JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getblock)

#### New Treasury Add Transaction Filter on Mempool Query RPC (`getrawmempool`)

The transaction type parameter of the `getrawmempool` RPC now accepts `tadd` to
only include treasury add transactions in the results.

See the
[getrawmempool JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getrawmempool)
for API details.

#### New Manual Block Invalidation and Reconsideration RPCs (`invalidateblock` and `reconsiderblock`)

A new pair of RPCs named `invalidateblock` and `reconsiderblock` are now
available.  These RPCs can be used to manually invalidate a block as if it had
violated consensus rules and reconsider a block for validation and best chain
selection by removing any invalid status from it and its ancestors, respectively.

This capability is provided for development, testing, and debugging.  It can be
particularly useful when developing services that build on top of Decred to more
easily ensure edge conditions associated with invalid blocks and chain
reorganization are being handled properly.

These RPCs do not apply to regular users and can safely be ignored outside of
development.

See the following for API details:

* [invalidateblock JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#invalidateblock)
* [reconsiderblock JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#reconsiderblock)

### Reject Protocol Message Deprecated (`reject`)

The `reject` peer-to-peer protocol message is now deprecated and is scheduled to
be removed in a future release.

This message is a holdover from the original codebase where it was required, but
it really is not a useful message and has several downsides:

* Nodes on the network must be trustless, which means anything relying on such a
  message is setting itself up for failure because nodes are not obligated to
  send it at all nor be truthful as to the reason
* It can be harmful to privacy as it allows additional node fingerprinting
* It can lead to security issues for implementations that don't handle it with
  proper sanitization practices
* It can easily give software implementations the fully incorrect impression
  that it can be relied on for determining if transactions and blocks are valid
* The only way it is actually used currently is to show a debug log message,
  however, all of that information is already available via the peer and/or wire
  logging anyway
* It carries a non-trivial amount of development overhead to continue to support
  it when nothing actually uses it

### No DNS Seeds Command-Line Option Deprecated (`--nodnsseed`)

The `--nodnsseed` command-line configuration option is now deprecated and will
be removed in a future release.  Use `--noseeders` instead.

DNS seeding has not been used since the previous release.

## Notable New Developer Modules

### Age-Partitioned Bloom Filters

A new `github.com/leedeternal/dcrd/container/apbf` module is now available that
provides Age-Partitioned Bloom Filters (APBFs).

An APBF is a probabilistic lookup device that can quickly determine if it
contains an element.  It permits tracking large amounts of data while using very
little memory at the cost of a controlled rate of false positives.  Unlike
classic Bloom filters, it is able to handle an unbounded amount of data by aging
and discarding old items.

For a concrete example of actual savings achieved in Decred by making use of an
APBF, the memory to track addresses known by 125 peers was reduced from ~200 MiB
to ~5 MiB.

See the
[apbf module documentation](https://pkg.go.dev/github.com/leedeternal/dcrd/container/apbf)
for full details on usage, accuracy under workloads, expected memory usage, and
performance benchmarks.

### Fixed-Precision Unsigned 256-bit Integers

A new `github.com/leedeternal/dcrd/math/uint256` module is now available that provides
highly optimized allocation free fixed precision unsigned 256-bit integer
arithmetic.

The package has a strong focus on performance and correctness and features
arithmetic, boolean comparison, bitwise logic, bitwise shifts, conversion
to/from relevant types, and full formatting support - all served with an
ergonomic API, full test coverage, and benchmarks.

Every operation is faster than the standard library `big.Int` equivalent and the
primary math operations provide reductions of over 90% in the calculation time.
Most other operations are also significantly faster.

See the
[uint256 module documentation](https://pkg.go.dev/github.com/leedeternal/dcrd/math/uint256)
for full details on usage, including a categorized summary, and performance
benchmarks.

### Standard Scripts

A new `github.com/leedeternal/dcrd/txscript/v4/stdscript` package is now available
that provides facilities for identifying and extracting data from transaction
scripts that are considered standard by the default policy of most nodes.

The package is part of the `github.com/leedeternal/dcrd/txscript/v4` module.

See the
[stdscript package documentation](https://pkg.go.dev/github.com/leedeternal/dcrd/txscript/v4/stdscript)
for full details on usage and a list of the recognized standard scripts.

### Standard Addresses

A new `github.com/leedeternal/dcrd/txscript/v4/stdaddr` package is now available that
provides facilities for working with human-readable Decred payment addresses.

The package is part of the `github.com/leedeternal/dcrd/txscript/v4` module.

See the
[stdaddr package documentation](https://pkg.go.dev/github.com/leedeternal/dcrd/txscript/v4/stdaddr)
for full details on usage and a list of the supported addresses.

## Changelog

This release consists of 877 commits from 16 contributors which total to 492
files changed, 77937 additional lines of code, and 30961 deleted lines of code.

All commits since the last release may be viewed on GitHub
[here](https://github.com/leedeternal/dcrd/compare/release-v1.6.0...release-v1.7.0).

### Protocol and network:

- chaincfg: Add extra seeders ([leedeternal/dcrd#2532](https://github.com/leedeternal/dcrd/pull/2532))
- server: Stop serving v1 cfilters over p2p ([leedeternal/dcrd#2525](https://github.com/leedeternal/dcrd/pull/2525))
- blockchain: Decouple processing and download logic ([leedeternal/dcrd#2518](https://github.com/leedeternal/dcrd/pull/2518))
- blockchain: Improve current detection ([leedeternal/dcrd#2518](https://github.com/leedeternal/dcrd/pull/2518))
- netsync: Rework inventory announcement handling ([leedeternal/dcrd#2548](https://github.com/leedeternal/dcrd/pull/2548))
- peer: Add inv type summary to debug message ([leedeternal/dcrd#2556](https://github.com/leedeternal/dcrd/pull/2556))
- netsync: Remove unused submit block flags param ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- netsync: Remove submit/processblock orphan flag ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- netsync: Remove orphan block handling ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- netsync: Rework sync model to use hdr annoucements ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- progresslog: Add support for header sync progress ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- netsync: Add header sync progress log ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- multi: Add chain verify progress percentage ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- peer: Remove getheaders response deadline ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- chaincfg: Update seed URL ([leedeternal/dcrd#2564](https://github.com/leedeternal/dcrd/pull/2564))
- upnp: Don't return loopback IPs in getOurIP ([leedeternal/dcrd#2566](https://github.com/leedeternal/dcrd/pull/2566))
- server: Prevent duplicate pending conns ([leedeternal/dcrd#2563](https://github.com/leedeternal/dcrd/pull/2563))
- multi: Use an APBF for recently confirmed txns ([leedeternal/dcrd#2580](https://github.com/leedeternal/dcrd/pull/2580))
- multi: Use an APBF for per peer known addrs ([leedeternal/dcrd#2583](https://github.com/leedeternal/dcrd/pull/2583))
- peer: Stop sending and logging reject messages ([leedeternal/dcrd#2586](https://github.com/leedeternal/dcrd/pull/2586))
- netsync: Stop sending reject messages ([leedeternal/dcrd#2586](https://github.com/leedeternal/dcrd/pull/2586))
- server: Stop sending reject messages ([leedeternal/dcrd#2586](https://github.com/leedeternal/dcrd/pull/2586))
- peer: Remove deprecated onversion reject return ([leedeternal/dcrd#2586](https://github.com/leedeternal/dcrd/pull/2586))
- peer: Remove unneeded PushRejectMsg ([leedeternal/dcrd#2586](https://github.com/leedeternal/dcrd/pull/2586))
- wire: Deprecate reject message ([leedeternal/dcrd#2586](https://github.com/leedeternal/dcrd/pull/2586))
- server: Respond to getheaders when same chain tip ([leedeternal/dcrd#2587](https://github.com/leedeternal/dcrd/pull/2587))
- netsync: Use an APBF for recently rejected txns ([leedeternal/dcrd#2590](https://github.com/leedeternal/dcrd/pull/2590))
- server: Only send fast block anns to full nodes ([leedeternal/dcrd#2606](https://github.com/leedeternal/dcrd/pull/2606))
- upnp: More accurate getOurIP ([leedeternal/dcrd#2571](https://github.com/leedeternal/dcrd/pull/2571))
- server: Correct tx not found ban reason ([leedeternal/dcrd#2677](https://github.com/leedeternal/dcrd/pull/2677))
- chaincfg: Add DCP0007 deployment ([leedeternal/dcrd#2679](https://github.com/leedeternal/dcrd/pull/2679))
- chaincfg: Introduce explicit ver upgrades agenda ([leedeternal/dcrd#2713](https://github.com/leedeternal/dcrd/pull/2713))
- blockchain: Implement reject new tx vers vote ([leedeternal/dcrd#2716](https://github.com/leedeternal/dcrd/pull/2716))
- blockchain: Implement reject new script vers vote ([leedeternal/dcrd#2716](https://github.com/leedeternal/dcrd/pull/2716))
- chaincfg: Add agenda for auto ticket revocations ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- multi: DCP0009 Auto revocations consensus change ([leedeternal/dcrd#2720](https://github.com/leedeternal/dcrd/pull/2720))
- chaincfg: Use single latest checkpoint ([leedeternal/dcrd#2762](https://github.com/leedeternal/dcrd/pull/2762))
- peer: Offset ping interval from idle timeout ([leedeternal/dcrd#2796](https://github.com/leedeternal/dcrd/pull/2796))
- chaincfg: Update checkpoint for upcoming release ([leedeternal/dcrd#2794](https://github.com/leedeternal/dcrd/pull/2794))
- chaincfg: Update min known chain work for release ([leedeternal/dcrd#2795](https://github.com/leedeternal/dcrd/pull/2795))
- netsync: Request init state immediately upon sync ([leedeternal/dcrd#2812](https://github.com/leedeternal/dcrd/pull/2812))
- blockchain: Reject old block vers for HFV ([leedeternal/dcrd#2752](https://github.com/leedeternal/dcrd/pull/2752))
- netsync: Rework next block download logic ([leedeternal/dcrd#2828](https://github.com/leedeternal/dcrd/pull/2828))
- chaincfg: Add AssumeValid param ([leedeternal/dcrd#2839](https://github.com/leedeternal/dcrd/pull/2839))
- chaincfg: Introduce subsidy split change agenda ([leedeternal/dcrd#2847](https://github.com/leedeternal/dcrd/pull/2847))
- multi: Implement DCP0010 subsidy consensus vote ([leedeternal/dcrd#2848](https://github.com/leedeternal/dcrd/pull/2848))
- server: Force PoW upgrade to v9 ([leedeternal/dcrd#2875](https://github.com/leedeternal/dcrd/pull/2875))

### Transaction relay (memory pool):

- mempool: Limit ancestor tracking in mempool ([leedeternal/dcrd#2458](https://github.com/leedeternal/dcrd/pull/2458))
- mempool: Remove old fix sequence lock rejection ([leedeternal/dcrd#2496](https://github.com/leedeternal/dcrd/pull/2496))
- mempool: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- mempool: Enforce explicit versions ([leedeternal/dcrd#2716](https://github.com/leedeternal/dcrd/pull/2716))
- mempool: Remove unneeded max tx ver std checks ([leedeternal/dcrd#2716](https://github.com/leedeternal/dcrd/pull/2716))
- mempool: Update fraud proof data ([leedeternal/dcrd#2804](https://github.com/leedeternal/dcrd/pull/2804))
- mempool: CheckTransactionInputs check fraud proof ([leedeternal/dcrd#2804](https://github.com/leedeternal/dcrd/pull/2804))

### Mining:

- mining: Move txPriorityQueue to a separate file ([leedeternal/dcrd#2431](https://github.com/leedeternal/dcrd/pull/2431))
- mining: Move interfaces to mining/interface.go ([leedeternal/dcrd#2431](https://github.com/leedeternal/dcrd/pull/2431))
- mining: Add method comments to blockManagerFacade ([leedeternal/dcrd#2431](https://github.com/leedeternal/dcrd/pull/2431))
- mining: Move BgBlkTmplGenerator to separate file ([leedeternal/dcrd#2431](https://github.com/leedeternal/dcrd/pull/2431))
- mining: Prevent panic in child prio item handling ([leedeternal/dcrd#2434](https://github.com/leedeternal/dcrd/pull/2434))
- mining: Add Config struct to house mining params ([leedeternal/dcrd#2436](https://github.com/leedeternal/dcrd/pull/2436))
- mining: Move block chain functions to Config ([leedeternal/dcrd#2436](https://github.com/leedeternal/dcrd/pull/2436))
- mining: Move txMiningView from mempool package ([leedeternal/dcrd#2467](https://github.com/leedeternal/dcrd/pull/2467))
- mining: Switch to custom waitGroup impl ([leedeternal/dcrd#2477](https://github.com/leedeternal/dcrd/pull/2477))
- mining: Remove leftover block manager facade iface ([leedeternal/dcrd#2510](https://github.com/leedeternal/dcrd/pull/2510))
- mining: No error log on expected head reorg errors ([leedeternal/dcrd#2560](https://github.com/leedeternal/dcrd/pull/2560))
- mining: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- mining: Add error kinds for auto revocations ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- mining: Add auto revocation priority to tx queue ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- mining: Add HeaderByHash to Config ([leedeternal/dcrd#2720](https://github.com/leedeternal/dcrd/pull/2720))
- mining: Prevent unnecessary reorg with equal votes ([leedeternal/dcrd#2840](https://github.com/leedeternal/dcrd/pull/2840))
- mining: Update to latest block vers for HFV ([leedeternal/dcrd#2753](https://github.com/leedeternal/dcrd/pull/2753))

### RPC:

- rpcserver: Upgrade is deprecated; switch to Upgrader ([leedeternal/dcrd#2409](https://github.com/leedeternal/dcrd/pull/2409))
- multi: Add TAdd support to getrawmempool ([leedeternal/dcrd#2448](https://github.com/leedeternal/dcrd/pull/2448))
- rpcserver: Update getrawmempool txtype help ([leedeternal/dcrd#2452](https://github.com/leedeternal/dcrd/pull/2452))
- rpcserver: Hash auth using random-keyed MAC ([leedeternal/dcrd#2486](https://github.com/leedeternal/dcrd/pull/2486))
- rpcserver: Use next stake diff from snapshot ([leedeternal/dcrd#2493](https://github.com/leedeternal/dcrd/pull/2493))
- rpcserver: Make authenticate match header auth ([leedeternal/dcrd#2502](https://github.com/leedeternal/dcrd/pull/2502))
- rpcserver: Check unauthorized access in const time ([leedeternal/dcrd#2509](https://github.com/leedeternal/dcrd/pull/2509))
- multi: Subscribe for work ntfns in rpcserver ([leedeternal/dcrd#2501](https://github.com/leedeternal/dcrd/pull/2501))
- rpcserver: Prune block templates in websocket path ([leedeternal/dcrd#2503](https://github.com/leedeternal/dcrd/pull/2503))
- rpcserver: Remove version from gettxout result ([leedeternal/dcrd#2517](https://github.com/leedeternal/dcrd/pull/2517))
- rpcserver: Add tree param to gettxout ([leedeternal/dcrd#2517](https://github.com/leedeternal/dcrd/pull/2517))
- rpcserver/netsync: Remove notifystakedifficulty ([leedeternal/dcrd#2519](https://github.com/leedeternal/dcrd/pull/2519))
- rpcserver: Remove v1 getcfilter{,header} ([leedeternal/dcrd#2525](https://github.com/leedeternal/dcrd/pull/2525))
- rpcserver: Remove unused Filterer interface ([leedeternal/dcrd#2525](https://github.com/leedeternal/dcrd/pull/2525))
- rpcserver: Update getblockchaininfo best header ([leedeternal/dcrd#2518](https://github.com/leedeternal/dcrd/pull/2518))
- rpcserver: Remove unused LocateBlocks iface method ([leedeternal/dcrd#2538](https://github.com/leedeternal/dcrd/pull/2538))
- rpcserver: Allow TLS client cert authentication ([leedeternal/dcrd#2482](https://github.com/leedeternal/dcrd/pull/2482))
- rpcserver: Add invalidate/reconsiderblock support ([leedeternal/dcrd#2536](https://github.com/leedeternal/dcrd/pull/2536))
- rpcserver: Support getblockchaininfo genesis block ([leedeternal/dcrd#2550](https://github.com/leedeternal/dcrd/pull/2550))
- rpcserver: Calc verify progress based on best hdr ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- rpcserver: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- rpcserver: Allow gettreasurybalance empty blk str ([leedeternal/dcrd#2640](https://github.com/leedeternal/dcrd/pull/2640))
- rpcserver: Add median time to verbose results ([leedeternal/dcrd#2638](https://github.com/leedeternal/dcrd/pull/2638))
- rpcserver: Allow interface names for dial addresses ([leedeternal/dcrd#2623](https://github.com/leedeternal/dcrd/pull/2623))
- rpcserver: Add script version to gettxout ([leedeternal/dcrd#2650](https://github.com/leedeternal/dcrd/pull/2650))
- rpcserver: Remove unused help entry ([leedeternal/dcrd#2648](https://github.com/leedeternal/dcrd/pull/2648))
- rpcserver: Set script version in raw tx results ([leedeternal/dcrd#2663](https://github.com/leedeternal/dcrd/pull/2663))
- rpcserver: Impose additional read limits ([leedeternal/dcrd#2675](https://github.com/leedeternal/dcrd/pull/2675))
- rpcserver: Add more strict request origin check ([leedeternal/dcrd#2676](https://github.com/leedeternal/dcrd/pull/2676))
- rpcserver: Use duplicate tx error for recently mined transactions ([leedeternal/dcrd#2705](https://github.com/leedeternal/dcrd/pull/2705))
- rpcserver: Wait for sync on rpc request ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- rpcserver: Update websocket ping timeout handling ([leedeternal/dcrd#2866](https://github.com/leedeternal/dcrd/pull/2866))

### dcrd command-line flags and configuration:

- multi: Rename BMGR subsystem to SYNC ([leedeternal/dcrd#2500](https://github.com/leedeternal/dcrd/pull/2500))
- server/indexers: Remove v1 cfilter indexing support ([leedeternal/dcrd#2525](https://github.com/leedeternal/dcrd/pull/2525))
- config: Add utxocachemaxsize ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- main: Update slog for LOGFLAGS=nodatetime support ([leedeternal/dcrd#2608](https://github.com/leedeternal/dcrd/pull/2608))
- config: Allow interface names for listener addresses ([leedeternal/dcrd#2623](https://github.com/leedeternal/dcrd/pull/2623))
- config: Correct dir create failure error message ([leedeternal/dcrd#2682](https://github.com/leedeternal/dcrd/pull/2682))
- config: Add logsize config option ([leedeternal/dcrd#2711](https://github.com/leedeternal/dcrd/pull/2711))
- config: conditionally generate rpc credentials ([leedeternal/dcrd#2779](https://github.com/leedeternal/dcrd/pull/2779))
- multi: Add assumevalid config option ([leedeternal/dcrd#2839](https://github.com/leedeternal/dcrd/pull/2839))

### gencerts utility changes:

- gencerts: Add certificate authority capabilities ([leedeternal/dcrd#2478](https://github.com/leedeternal/dcrd/pull/2478))
- gencerts: Add RSA support (4096 bit keys only) ([leedeternal/dcrd#2551](https://github.com/leedeternal/dcrd/pull/2551))

### addblock utility changes:

- cmd/addblock: update block importer ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- addblock: Run index subscriber as a goroutine ([leedeternal/dcrd#2760](https://github.com/leedeternal/dcrd/pull/2760))
- addblock: Fix blockchain initialization ([leedeternal/dcrd#2760](https://github.com/leedeternal/dcrd/pull/2760))
- addblock: Use chain bulk import mode ([leedeternal/dcrd#2782](https://github.com/leedeternal/dcrd/pull/2782))

### findcheckpoint utility changes:

- findcheckpoint: Fix blockchain initialization ([leedeternal/dcrd#2759](https://github.com/leedeternal/dcrd/pull/2759))

### Documentation:

- docs: Fix JSON-RPC API gettxoutsetinfo description ([leedeternal/dcrd#2443](https://github.com/leedeternal/dcrd/pull/2443))
- docs: Add JSON-RPC API getpeerinfo missing fields ([leedeternal/dcrd#2443](https://github.com/leedeternal/dcrd/pull/2443))
- docs: Fix JSON-RPC API gettreasurybalance fmt ([leedeternal/dcrd#2443](https://github.com/leedeternal/dcrd/pull/2443))
- docs: Fix JSON-RPC API gettreasuryspendvotes fmt ([leedeternal/dcrd#2443](https://github.com/leedeternal/dcrd/pull/2443))
- docs: Add JSON-RPC API searchrawtxns req limit ([leedeternal/dcrd#2443](https://github.com/leedeternal/dcrd/pull/2443))
- docs: Update JSON-RPC API getrawmempool ([leedeternal/dcrd#2453](https://github.com/leedeternal/dcrd/pull/2453))
- progresslog: Add package documentation ([leedeternal/dcrd#2499](https://github.com/leedeternal/dcrd/pull/2499))
- netsync: Add package documentation ([leedeternal/dcrd#2500](https://github.com/leedeternal/dcrd/pull/2500))
- multi: update error code related documentation ([leedeternal/dcrd#2515](https://github.com/leedeternal/dcrd/pull/2515))
- docs: Update JSON-RPC API getwork to match reality ([leedeternal/dcrd#2526](https://github.com/leedeternal/dcrd/pull/2526))
- docs: Remove notifystakedifficulty JSON-RPC API ([leedeternal/dcrd#2519](https://github.com/leedeternal/dcrd/pull/2519))
- docs: Remove v1 getcfilter{,header} JSON-RPC API ([leedeternal/dcrd#2525](https://github.com/leedeternal/dcrd/pull/2525))
- chaincfg: Update doc.go ([leedeternal/dcrd#2528](https://github.com/leedeternal/dcrd/pull/2528))
- blockchain: Update README.md and doc.go ([leedeternal/dcrd#2518](https://github.com/leedeternal/dcrd/pull/2518))
- docs: Add invalidate/reconsiderblock JSON-RPC API ([leedeternal/dcrd#2536](https://github.com/leedeternal/dcrd/pull/2536))
- docs: Add release notes for v1.6.0 ([leedeternal/dcrd#2451](https://github.com/leedeternal/dcrd/pull/2451))
- multi: Update README.md files for go modules ([leedeternal/dcrd#2559](https://github.com/leedeternal/dcrd/pull/2559))
- apbf: Add README.md ([leedeternal/dcrd#2579](https://github.com/leedeternal/dcrd/pull/2579))
- docs: Add release notes for v1.6.1 ([leedeternal/dcrd#2601](https://github.com/leedeternal/dcrd/pull/2601))
- docs: Update min recommended specs in README.md ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- stdaddr: Add README.md ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add serialized pubkey info to README.md ([leedeternal/dcrd#2619](https://github.com/leedeternal/dcrd/pull/2619))
- docs: Add release notes for v1.6.2 ([leedeternal/dcrd#2630](https://github.com/leedeternal/dcrd/pull/2630))
- docs: Add scriptpubkey json returns ([leedeternal/dcrd#2650](https://github.com/leedeternal/dcrd/pull/2650))
- stdscript: Add README.md ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stake: Comment on max SSGen outputs with treasury ([leedeternal/dcrd#2664](https://github.com/leedeternal/dcrd/pull/2664))
- docs: Update JSON-RPC API for script version ([leedeternal/dcrd#2663](https://github.com/leedeternal/dcrd/pull/2663))
- docs: Update JSON-RPC API for max request limits ([leedeternal/dcrd#2675](https://github.com/leedeternal/dcrd/pull/2675))
- docs: Add SECURITY.md file ([leedeternal/dcrd#2717](https://github.com/leedeternal/dcrd/pull/2717))
- sampleconfig: Add missing log options ([leedeternal/dcrd#2723](https://github.com/leedeternal/dcrd/pull/2723))
- docs: Update go versions in README.md ([leedeternal/dcrd#2722](https://github.com/leedeternal/dcrd/pull/2722))
- docs: Correct generate description ([leedeternal/dcrd#2724](https://github.com/leedeternal/dcrd/pull/2724))
- database: Correct README rpcclient link ([leedeternal/dcrd#2725](https://github.com/leedeternal/dcrd/pull/2725))
- docs: Add accuracy and reliability to README.md ([leedeternal/dcrd#2726](https://github.com/leedeternal/dcrd/pull/2726))
- sampleconfig: Update for deprecated nodnsseed ([leedeternal/dcrd#2728](https://github.com/leedeternal/dcrd/pull/2728))
- docs: Update for secp256k1 v4 module ([leedeternal/dcrd#2732](https://github.com/leedeternal/dcrd/pull/2732))
- docs: Update for new modules ([leedeternal/dcrd#2744](https://github.com/leedeternal/dcrd/pull/2744))
- sampleconfig: update rpc credentials documentation ([leedeternal/dcrd#2779](https://github.com/leedeternal/dcrd/pull/2779))
- docs: Update for addrmgr v2 module ([leedeternal/dcrd#2797](https://github.com/leedeternal/dcrd/pull/2797))
- docs: Update for rpc/jsonrpc/types v3 module ([leedeternal/dcrd#2801](https://github.com/leedeternal/dcrd/pull/2801))
- stdscript: Update README.md for provably pruneable ([leedeternal/dcrd#2803](https://github.com/leedeternal/dcrd/pull/2803))
- docs: Update for txscript v3 module ([leedeternal/dcrd#2815](https://github.com/leedeternal/dcrd/pull/2815))
- docs: Update for dcrutil v4 module ([leedeternal/dcrd#2818](https://github.com/leedeternal/dcrd/pull/2818))
- uint256: Add README.md ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- docs: Update for peer v3 module ([leedeternal/dcrd#2820](https://github.com/leedeternal/dcrd/pull/2820))
- docs: Update for database v3 module ([leedeternal/dcrd#2822](https://github.com/leedeternal/dcrd/pull/2822))
- docs: Update for blockchain/stake v4 module ([leedeternal/dcrd#2824](https://github.com/leedeternal/dcrd/pull/2824))
- docs: Update for gcs v3 module ([leedeternal/dcrd#2830](https://github.com/leedeternal/dcrd/pull/2830))
- docs: Fix typos and trailing whitespace ([leedeternal/dcrd#2843](https://github.com/leedeternal/dcrd/pull/2843))
- docs: Add max line length and wrapping guidelines ([leedeternal/dcrd#2843](https://github.com/leedeternal/dcrd/pull/2843))
- docs: Update for math/uint256 module ([leedeternal/dcrd#2842](https://github.com/leedeternal/dcrd/pull/2842))
- docs: Update simnet env docs for subsidy split ([leedeternal/dcrd#2848](https://github.com/leedeternal/dcrd/pull/2848))
- docs: Update for blockchain v4 module ([leedeternal/dcrd#2831](https://github.com/leedeternal/dcrd/pull/2831))
- docs: Update for rpcclient v7 module ([leedeternal/dcrd#2851](https://github.com/leedeternal/dcrd/pull/2851))
- primitives: Add skeleton README.md ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))

### Contrib changes:

- contrib: Update OpenBSD rc script for 6.9 features ([leedeternal/dcrd#2646](https://github.com/leedeternal/dcrd/pull/2646))
- contrib: Bump Dockerfile.alpine to alpine:3.14.0 ([leedeternal/dcrd#2681](https://github.com/leedeternal/dcrd/pull/2681))
- build: Use go 1.17 in Dockerfiles ([leedeternal/dcrd#2722](https://github.com/leedeternal/dcrd/pull/2722))
- build: Pin docker images with SHA instead of tag ([leedeternal/dcrd#2735](https://github.com/leedeternal/dcrd/pull/2735))
- build/contrib: Improve docker support ([leedeternal/dcrd#2740](https://github.com/leedeternal/dcrd/pull/2740))

### Developer-related package and module changes:

- dcrjson: Reject dup method type registrations ([leedeternal/dcrd#2417](https://github.com/leedeternal/dcrd/pull/2417))
- peer: various cleanups ([leedeternal/dcrd#2396](https://github.com/leedeternal/dcrd/pull/2396))
- blockchain: Create treasury buckets during upgrade ([leedeternal/dcrd#2441](https://github.com/leedeternal/dcrd/pull/2441))
- blockchain: Fix stxosToScriptSource ([leedeternal/dcrd#2444](https://github.com/leedeternal/dcrd/pull/2444))
- rpcserver: add NtfnManager interface ([leedeternal/dcrd#2410](https://github.com/leedeternal/dcrd/pull/2410))
- lru: Fix lookup race on small caches ([leedeternal/dcrd#2464](https://github.com/leedeternal/dcrd/pull/2464))
- gcs: update error types ([leedeternal/dcrd#2262](https://github.com/leedeternal/dcrd/pull/2262))
- main: Switch windows service dependency ([leedeternal/dcrd#2479](https://github.com/leedeternal/dcrd/pull/2479))
- blockchain: Simplify upgrade single run stage code ([leedeternal/dcrd#2457](https://github.com/leedeternal/dcrd/pull/2457))
- blockchain: Simplify upgrade batching logic ([leedeternal/dcrd#2457](https://github.com/leedeternal/dcrd/pull/2457))
- blockchain: Use new batching logic for filter init ([leedeternal/dcrd#2457](https://github.com/leedeternal/dcrd/pull/2457))
- blockchain: Use new batch logic for blkidx upgrade ([leedeternal/dcrd#2457](https://github.com/leedeternal/dcrd/pull/2457))
- blockchain: Use new batch logic for utxos upgrade ([leedeternal/dcrd#2457](https://github.com/leedeternal/dcrd/pull/2457))
- blockchain: Use new batch logic for spends upgrade ([leedeternal/dcrd#2457](https://github.com/leedeternal/dcrd/pull/2457))
- blockchain: Use new batch logic for clr failed ([leedeternal/dcrd#2457](https://github.com/leedeternal/dcrd/pull/2457))
- windows: Switch to os.Executable ([leedeternal/dcrd#2485](https://github.com/leedeternal/dcrd/pull/2485))
- blockchain: Revert fast add reversal ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Less order dependent full blocks tests ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Move context free tx sanity checks ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Move context free block sanity checks ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Rework contextual tx checks ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Move {coin,trsy}base contextual checks ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Move staketx-related contextual checks ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Move sigop-related contextual checks ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Make CheckBlockSanity context free ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Context free CheckTransactionSanity ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- blockchain: Move contextual treasury spend checks ([leedeternal/dcrd#2481](https://github.com/leedeternal/dcrd/pull/2481))
- mempool: Comment and stylistic updates ([leedeternal/dcrd#2480](https://github.com/leedeternal/dcrd/pull/2480))
- mining: Rename TxMiningView Remove method ([leedeternal/dcrd#2490](https://github.com/leedeternal/dcrd/pull/2490))
- mining: Unexport TxMiningView methods ([leedeternal/dcrd#2490](https://github.com/leedeternal/dcrd/pull/2490))
- mining: Update mergeUtxoView comment ([leedeternal/dcrd#2490](https://github.com/leedeternal/dcrd/pull/2490))
- blockchain: Consolidate deployment errors ([leedeternal/dcrd#2487](https://github.com/leedeternal/dcrd/pull/2487))
- blockchain: Consolidate unknown block errors ([leedeternal/dcrd#2487](https://github.com/leedeternal/dcrd/pull/2487))
- blockchain: Consolidate no filter errors ([leedeternal/dcrd#2487](https://github.com/leedeternal/dcrd/pull/2487))
- blockchain: Consolidate no treasury bal errors ([leedeternal/dcrd#2487](https://github.com/leedeternal/dcrd/pull/2487))
- blockchain: Convert to LRU block cache ([leedeternal/dcrd#2488](https://github.com/leedeternal/dcrd/pull/2488))
- blockchain: Remove unused error returns ([leedeternal/dcrd#2489](https://github.com/leedeternal/dcrd/pull/2489))
- blockmanager: Remove unused stakediff infra ([leedeternal/dcrd#2493](https://github.com/leedeternal/dcrd/pull/2493))
- server: Use next stake diff from snapshot ([leedeternal/dcrd#2493](https://github.com/leedeternal/dcrd/pull/2493))
- blockchain: Explicit hash in next stake diff calcs ([leedeternal/dcrd#2494](https://github.com/leedeternal/dcrd/pull/2494))
- blockchain: Explicit hash in LN agenda active func ([leedeternal/dcrd#2495](https://github.com/leedeternal/dcrd/pull/2495))
- blockmanager: Remove unused config field ([leedeternal/dcrd#2497](https://github.com/leedeternal/dcrd/pull/2497))
- blockmanager: Decouple block database code ([leedeternal/dcrd#2497](https://github.com/leedeternal/dcrd/pull/2497))
- blockmanager: Decouple from global config var ([leedeternal/dcrd#2497](https://github.com/leedeternal/dcrd/pull/2497))
- blockchain: Explicit hash in max block size func ([leedeternal/dcrd#2507](https://github.com/leedeternal/dcrd/pull/2507))
- progresslog: Make block progress log internal ([leedeternal/dcrd#2499](https://github.com/leedeternal/dcrd/pull/2499))
- server: Do not use unexported block manager cfg ([leedeternal/dcrd#2498](https://github.com/leedeternal/dcrd/pull/2498))
- blockmanager: Rework chain current logic ([leedeternal/dcrd#2498](https://github.com/leedeternal/dcrd/pull/2498))
- multi: Handle chain ntfn callback in server ([leedeternal/dcrd#2498](https://github.com/leedeternal/dcrd/pull/2498))
- server: Rename blockManager field to syncManager ([leedeternal/dcrd#2500](https://github.com/leedeternal/dcrd/pull/2500))
- server: Add temp sync manager interface ([leedeternal/dcrd#2500](https://github.com/leedeternal/dcrd/pull/2500))
- netsync: Split blockmanager into separate package ([leedeternal/dcrd#2500](https://github.com/leedeternal/dcrd/pull/2500))
- netsync: Rename blockManager to SyncManager ([leedeternal/dcrd#2500](https://github.com/leedeternal/dcrd/pull/2500))
- internal/ticketdb: update error types ([leedeternal/dcrd#2279](https://github.com/leedeternal/dcrd/pull/2279))
- secp256k1/ecdsa: update error types ([leedeternal/dcrd#2281](https://github.com/leedeternal/dcrd/pull/2281))
- secp256k1/schnorr: update error types ([leedeternal/dcrd#2282](https://github.com/leedeternal/dcrd/pull/2282))
- dcrjson: update error types ([leedeternal/dcrd#2271](https://github.com/leedeternal/dcrd/pull/2271))
- dcrec/secp256k1: update error types ([leedeternal/dcrd#2265](https://github.com/leedeternal/dcrd/pull/2265))
- blockchain/stake: update error types ([leedeternal/dcrd#2264](https://github.com/leedeternal/dcrd/pull/2264))
- multi: update database error types ([leedeternal/dcrd#2261](https://github.com/leedeternal/dcrd/pull/2261))
- blockchain: Remove unused treasury active func ([leedeternal/dcrd#2514](https://github.com/leedeternal/dcrd/pull/2514))
- stake: update ticket lottery errors ([leedeternal/dcrd#2433](https://github.com/leedeternal/dcrd/pull/2433))
- netsync: Improve is current detection ([leedeternal/dcrd#2513](https://github.com/leedeternal/dcrd/pull/2513))
- internal/mining: update mining error types ([leedeternal/dcrd#2515](https://github.com/leedeternal/dcrd/pull/2515))
- multi: sprinkle on more errors.As/Is ([leedeternal/dcrd#2522](https://github.com/leedeternal/dcrd/pull/2522))
- mining: Correct fee calculations during reorgs ([leedeternal/dcrd#2530](https://github.com/leedeternal/dcrd/pull/2530))
- fees: Remove deprecated DisableLog ([leedeternal/dcrd#2529](https://github.com/leedeternal/dcrd/pull/2529))
- rpcclient: Remove deprecated DisableLog ([leedeternal/dcrd#2527](https://github.com/leedeternal/dcrd/pull/2527))
- rpcclient: Remove notifystakedifficulty ([leedeternal/dcrd#2519](https://github.com/leedeternal/dcrd/pull/2519))
- rpc/jsonrpc/types: Remove notifystakedifficulty ([leedeternal/dcrd#2519](https://github.com/leedeternal/dcrd/pull/2519))
- netsync: Remove unneeded ForceReorganization ([leedeternal/dcrd#2520](https://github.com/leedeternal/dcrd/pull/2520))
- mining: Remove duplicate method ([leedeternal/dcrd#2520](https://github.com/leedeternal/dcrd/pull/2520))
- multi: use EstimateSmartFeeResult ([leedeternal/dcrd#2283](https://github.com/leedeternal/dcrd/pull/2283))
- rpcclient: Remove v1 getcfilter{,header} ([leedeternal/dcrd#2525](https://github.com/leedeternal/dcrd/pull/2525))
- rpc/jsonrpc/types: Remove v1 getcfilter{,header} ([leedeternal/dcrd#2525](https://github.com/leedeternal/dcrd/pull/2525))
- gcs: Remove unused v1 blockcf package ([leedeternal/dcrd#2525](https://github.com/leedeternal/dcrd/pull/2525))
- blockchain: Remove legacy sequence lock view ([leedeternal/dcrd#2534](https://github.com/leedeternal/dcrd/pull/2534))
- blockchain: Remove IsFixSeqLocksAgendaActive ([leedeternal/dcrd#2534](https://github.com/leedeternal/dcrd/pull/2534))
- blockchain: Explicit hash in estimate stake diff ([leedeternal/dcrd#2524](https://github.com/leedeternal/dcrd/pull/2524))
- netsync: Remove unneeded TipGeneration ([leedeternal/dcrd#2537](https://github.com/leedeternal/dcrd/pull/2537))
- netsync: Remove unused TicketPoolValue ([leedeternal/dcrd#2544](https://github.com/leedeternal/dcrd/pull/2544))
- netsync: Embed peers vs separate peer states ([leedeternal/dcrd#2541](https://github.com/leedeternal/dcrd/pull/2541))
- netsync/server: Update peer heights directly ([leedeternal/dcrd#2542](https://github.com/leedeternal/dcrd/pull/2542))
- netsync: Move proactive sigcache evict to server ([leedeternal/dcrd#2543](https://github.com/leedeternal/dcrd/pull/2543))
- blockchain: Add invalidate/reconsider infrastruct ([leedeternal/dcrd#2536](https://github.com/leedeternal/dcrd/pull/2536))
- rpc/jsonrpc/types: Add invalidate/reconsiderblock ([leedeternal/dcrd#2536](https://github.com/leedeternal/dcrd/pull/2536))
- netsync: Convert lifecycle to context ([leedeternal/dcrd#2545](https://github.com/leedeternal/dcrd/pull/2545))
- multi: Rework utxoset/view to use outpoints ([leedeternal/dcrd#2540](https://github.com/leedeternal/dcrd/pull/2540))
- blockchain: Remove compression version param ([leedeternal/dcrd#2547](https://github.com/leedeternal/dcrd/pull/2547))
- blockchain: Remove error from LatestBlockLocator ([leedeternal/dcrd#2548](https://github.com/leedeternal/dcrd/pull/2548))
- blockchain: Fix incorrect decompressScript calls ([leedeternal/dcrd#2552](https://github.com/leedeternal/dcrd/pull/2552))
- blockchain: Fix V3 spend journal migration ([leedeternal/dcrd#2552](https://github.com/leedeternal/dcrd/pull/2552))
- multi: Remove blockChain field from UtxoViewpoint ([leedeternal/dcrd#2553](https://github.com/leedeternal/dcrd/pull/2553))
- blockchain: Move UtxoEntry to a separate file ([leedeternal/dcrd#2553](https://github.com/leedeternal/dcrd/pull/2553))
- blockchain: Update UtxoEntry Clone method comment ([leedeternal/dcrd#2553](https://github.com/leedeternal/dcrd/pull/2553))
- progresslog: Make logger more generic ([leedeternal/dcrd#2555](https://github.com/leedeternal/dcrd/pull/2555))
- server: Remove several unused funcs ([leedeternal/dcrd#2561](https://github.com/leedeternal/dcrd/pull/2561))
- mempool: Store staged transactions as TxDesc ([leedeternal/dcrd#2319](https://github.com/leedeternal/dcrd/pull/2319))
- connmgr: Add func to iterate conn reqs ([leedeternal/dcrd#2562](https://github.com/leedeternal/dcrd/pull/2562))
- netsync: Correct check for needTx ([leedeternal/dcrd#2568](https://github.com/leedeternal/dcrd/pull/2568))
- rpcclient: Update EstimateSmartFee return type ([leedeternal/dcrd#2255](https://github.com/leedeternal/dcrd/pull/2255))
- server: Notify sync mgr later and track ntfn ([leedeternal/dcrd#2582](https://github.com/leedeternal/dcrd/pull/2582))
- apbf: Introduce Age-Partitioned Bloom Filters ([leedeternal/dcrd#2579](https://github.com/leedeternal/dcrd/pull/2579))
- apbf: Add basic usage example ([leedeternal/dcrd#2579](https://github.com/leedeternal/dcrd/pull/2579))
- apbf: Add support to go generate a KL table ([leedeternal/dcrd#2579](https://github.com/leedeternal/dcrd/pull/2579))
- apbf: Switch to fast reduce method ([leedeternal/dcrd#2584](https://github.com/leedeternal/dcrd/pull/2584))
- server: Remove unneeded child context ([leedeternal/dcrd#2593](https://github.com/leedeternal/dcrd/pull/2593))
- blockchain: Separate utxo state from tx flags ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- blockchain: Add utxoStateFresh to UtxoEntry ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- blockchain: Add size method to UtxoEntry ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- blockchain: Deep copy view entry script from tx ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- blockchain: Add utxoSetState to the database ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- multi: Add UtxoCache ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- blockchain: Make InitUtxoCache a UtxoCache method ([leedeternal/dcrd#2599](https://github.com/leedeternal/dcrd/pull/2599))
- blockchain: Add UtxoCacher interface ([leedeternal/dcrd#2599](https://github.com/leedeternal/dcrd/pull/2599))
- dcrutil: Correct ed25519 address constructor ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Introduce package infra for std addrs ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add infrastructure for v0 decoding ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add v0 p2pk-ecdsa-secp256k1 support ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add v0 p2pk-ed25519 support ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add v0 p2pk-schnorr-secp256k1 support ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add v0 p2pkh-ecdsa-secp256k1 support ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add v0 p2pkh-ed25519 support ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add v0 p2pkh-schnorr-secp256k1 support ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add v0 p2sh support ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- stdaddr: Add decode address example ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- txscript: Rename script bldr add data to unchecked ([leedeternal/dcrd#2611](https://github.com/leedeternal/dcrd/pull/2611))
- txscript: Add script bldr unchecked op add ([leedeternal/dcrd#2611](https://github.com/leedeternal/dcrd/pull/2611))
- rpcserver: Remove uncompressed pubkeys fast path ([leedeternal/dcrd#2617](https://github.com/leedeternal/dcrd/pull/2617))
- blockchain: Allow alternate tips for current check ([leedeternal/dcrd#2612](https://github.com/leedeternal/dcrd/pull/2612))
- txscript: Accept raw public keys in MultiSigScript ([leedeternal/dcrd#2615](https://github.com/leedeternal/dcrd/pull/2615))
- cpuminer: Remove unused MiningAddrs from Config ([leedeternal/dcrd#2616](https://github.com/leedeternal/dcrd/pull/2616))
- stdaddr: Add ability to obtain raw public key ([leedeternal/dcrd#2619](https://github.com/leedeternal/dcrd/pull/2619))
- stdaddr: Move from internal/staging to txscript ([leedeternal/dcrd#2620](https://github.com/leedeternal/dcrd/pull/2620))
- stdaddr: Accept vote and revoke limits separately ([leedeternal/dcrd#2624](https://github.com/leedeternal/dcrd/pull/2624))
- stake: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- indexers: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- blockchain: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- rpcclient: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- hdkeychain: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToSStx ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToSStxChange ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToSSGen ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToSSGenSHDirect ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToSSGenPKHDirect ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToSSRtx ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToSSRtxPKHDirect ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToSSRtxSHDirect ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToAddrScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused PayToScriptHashScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused payToSchnorrPubKeyScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused payToEdwardsPubKeyScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused payToPubKeyScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused payToScriptHashScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused payToPubKeyHashSchnorrScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused payToPubKeyHashEdwardsScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused payToPubKeyHashScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused GenerateSStxAddrPush ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Remove unused ErrUnsupportedAddress ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- txscript: Break dcrutil dependency ([leedeternal/dcrd#2626](https://github.com/leedeternal/dcrd/pull/2626))
- stdaddr: Replace Address method with String ([leedeternal/dcrd#2633](https://github.com/leedeternal/dcrd/pull/2633))
- dcrutil: Convert to use new stdaddr package ([leedeternal/dcrd#2628](https://github.com/leedeternal/dcrd/pull/2628))
- dcrutil: Remove all code related to Address ([leedeternal/dcrd#2628](https://github.com/leedeternal/dcrd/pull/2628))
- blockchain: Trsy always inactive for genesis blk ([leedeternal/dcrd#2636](https://github.com/leedeternal/dcrd/pull/2636))
- blockchain: Use agenda flags for tx check context ([leedeternal/dcrd#2639](https://github.com/leedeternal/dcrd/pull/2639))
- blockchain: Move UTXO DB methods to separate file ([leedeternal/dcrd#2632](https://github.com/leedeternal/dcrd/pull/2632))
- blockchain: Move UTXO DB tests to separate file ([leedeternal/dcrd#2632](https://github.com/leedeternal/dcrd/pull/2632))
- ipc: Fix lifetimeEvent comments ([leedeternal/dcrd#2632](https://github.com/leedeternal/dcrd/pull/2632))
- blockchain: Add utxoDatabaseInfo ([leedeternal/dcrd#2632](https://github.com/leedeternal/dcrd/pull/2632))
- multi: Introduce UTXO database ([leedeternal/dcrd#2632](https://github.com/leedeternal/dcrd/pull/2632))
- blockchain: Decouple stxo and utxo migrations ([leedeternal/dcrd#2632](https://github.com/leedeternal/dcrd/pull/2632))
- multi: Migrate to UTXO database ([leedeternal/dcrd#2632](https://github.com/leedeternal/dcrd/pull/2632))
- main: Handle SIGHUP with clean shutdown ([leedeternal/dcrd#2645](https://github.com/leedeternal/dcrd/pull/2645))
- txscript: Split signing code to sign subpackage ([leedeternal/dcrd#2642](https://github.com/leedeternal/dcrd/pull/2642))
- database: Add Flush to DB interface ([leedeternal/dcrd#2649](https://github.com/leedeternal/dcrd/pull/2649))
- multi: Flush block DB before UTXO DB ([leedeternal/dcrd#2649](https://github.com/leedeternal/dcrd/pull/2649))
- blockchain: Flush UTXO DB after init utxoSetState ([leedeternal/dcrd#2649](https://github.com/leedeternal/dcrd/pull/2649))
- blockchain: Force flush in separateUtxoDatabase ([leedeternal/dcrd#2649](https://github.com/leedeternal/dcrd/pull/2649))
- version: Rework to support single version override ([leedeternal/dcrd#2651](https://github.com/leedeternal/dcrd/pull/2651))
- blockchain: Remove UtxoCacher DB Tx dependency ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Add UtxoBackend interface ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Export UtxoSetState ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Add FetchEntry to UtxoBackend ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Add PutUtxos to UtxoBackend ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Add FetchState to UtxoBackend ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Add FetchStats to UtxoBackend ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Add FetchInfo to UtxoBackend ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Move LoadUtxoDB to UtxoBackend ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Add Upgrade to UtxoBackend ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- multi: Remove UTXO db in BlockChain and UtxoCache ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- blockchain: Export ViewFilteredSet ([leedeternal/dcrd#2652](https://github.com/leedeternal/dcrd/pull/2652))
- stake: Return StakeAddress from cmtmt conversion ([leedeternal/dcrd#2655](https://github.com/leedeternal/dcrd/pull/2655))
- stdscript: Introduce pkg infra for std scripts ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pk-ecdsa-secp256k1 support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pk-ed25519 support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pk-schnorr-secp256k1 support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pkh-ecdsa-secp256k1 support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pkh-ed25519 support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pkh-schnorr-secp256k1 support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2sh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 ecdsa multisig support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 ecdsa multisig redeem support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 nulldata support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake sub p2pkh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake sub p2sh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake gen p2pkh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake gen p2sh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake revoke p2pkh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake revoke p2sh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake change p2pkh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake change p2sh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 treasury add support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 treasury gen p2pkh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 treasury gen p2sh support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add ecdsa multisig creation script ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 atomic swap redeem support ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add example for determining script type ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add example for p2pkh extract ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add example of script hash extract ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- blockchain: Use scripts in tickets address query ([leedeternal/dcrd#2657](https://github.com/leedeternal/dcrd/pull/2657))
- stake: Do not use standardness code in consensus ([leedeternal/dcrd#2658](https://github.com/leedeternal/dcrd/pull/2658))
- blockchain: Remove unneeded OP_TADD maturity check ([leedeternal/dcrd#2659](https://github.com/leedeternal/dcrd/pull/2659))
- stake: Add is treasury gen script ([leedeternal/dcrd#2660](https://github.com/leedeternal/dcrd/pull/2660))
- blockchain: No standardness code in consensus ([leedeternal/dcrd#2661](https://github.com/leedeternal/dcrd/pull/2661))
- gcs: No standardness code in consensus ([leedeternal/dcrd#2662](https://github.com/leedeternal/dcrd/pull/2662))
- stake: Remove stale TODOs from CheckSSGenVotes ([leedeternal/dcrd#2665](https://github.com/leedeternal/dcrd/pull/2665))
- stake: Remove stale TODOs from CheckSSRtx ([leedeternal/dcrd#2665](https://github.com/leedeternal/dcrd/pull/2665))
- txscript: Move contains stake opcode to consensus ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- txscript: Move stake blockref script to consensus ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- txscript: Move stake votebits script to consensus ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- txscript: Remove unused IsPubKeyHashScript ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- txscript: Remove unused IsStakeChangeScript ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- txscript: Remove unused PushedData ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- blockchain: Flush UtxoCache when latch to current ([leedeternal/dcrd#2671](https://github.com/leedeternal/dcrd/pull/2671))
- dcrjson: Minor jsonerr.go update ([leedeternal/dcrd#2672](https://github.com/leedeternal/dcrd/pull/2672))
- rpcclient: Cancel client context on shutdown ([leedeternal/dcrd#2678](https://github.com/leedeternal/dcrd/pull/2678))
- blockchain: Remove serializeUtxoEntry error ([leedeternal/dcrd#2683](https://github.com/leedeternal/dcrd/pull/2683))
- blockchain: Add IsTreasuryEnabled to AgendaFlags ([leedeternal/dcrd#2686](https://github.com/leedeternal/dcrd/pull/2686))
- multi: Update block ntfns to contain AgendaFlags ([leedeternal/dcrd#2686](https://github.com/leedeternal/dcrd/pull/2686))
- multi: Update ProcessOrphans to use AgendaFlags ([leedeternal/dcrd#2686](https://github.com/leedeternal/dcrd/pull/2686))
- mempool: Add maybeAcceptTransaction AgendaFlags ([leedeternal/dcrd#2686](https://github.com/leedeternal/dcrd/pull/2686))
- secp256k1: Allow code generation to compile again ([leedeternal/dcrd#2687](https://github.com/leedeternal/dcrd/pull/2687))
- jsonrpc/types: Add missing Method type to vars ([leedeternal/dcrd#2688](https://github.com/leedeternal/dcrd/pull/2688))
- blockchain: Add UTXO backend error kinds ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- blockchain: Add helper to convert leveldb errors ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- blockchain: Add UtxoBackendIterator interface ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- blockchain: Add UtxoBackendTx interface ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- blockchain: Add levelDbUtxoBackendTx type ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- multi: Update UtxoBackend to use leveldb directly ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- multi: Move UTXO database ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- blockchain: Unexport levelDbUtxoBackend ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- blockchain: Always use node lookup methods ([leedeternal/dcrd#2685](https://github.com/leedeternal/dcrd/pull/2685))
- blockchain: Use short keys for block index ([leedeternal/dcrd#2685](https://github.com/leedeternal/dcrd/pull/2685))
- rpcclient: Shutdown breaks reconnect sleep ([leedeternal/dcrd#2696](https://github.com/leedeternal/dcrd/pull/2696))
- secp256k1: No deps on adaptor code for precomps ([leedeternal/dcrd#2690](https://github.com/leedeternal/dcrd/pull/2690))
- secp256k1: Always initialize adaptor instance ([leedeternal/dcrd#2690](https://github.com/leedeternal/dcrd/pull/2690))
- secp256k1: Optimize precomp values to use affine ([leedeternal/dcrd#2690](https://github.com/leedeternal/dcrd/pull/2690))
- rpcserver: Handle getwork nil err during reorg ([leedeternal/dcrd#2700](https://github.com/leedeternal/dcrd/pull/2700))
- secp256k1: Use blake256 directly in examples ([leedeternal/dcrd#2697](https://github.com/leedeternal/dcrd/pull/2697))
- secp256k1: Improve scalar mult readability ([leedeternal/dcrd#2695](https://github.com/leedeternal/dcrd/pull/2695))
- secp256k1: Optimize NAF conversion ([leedeternal/dcrd#2695](https://github.com/leedeternal/dcrd/pull/2695))
- blockchain: Verify state of DCP0007 voting ([leedeternal/dcrd#2679](https://github.com/leedeternal/dcrd/pull/2679))
- blockchain: Rename max expenditure funcs ([leedeternal/dcrd#2679](https://github.com/leedeternal/dcrd/pull/2679))
- stake: Add ExpiringNextBlock method to Node ([leedeternal/dcrd#2701](https://github.com/leedeternal/dcrd/pull/2701))
- rpcclient: Add GetNetworkInfo call ([leedeternal/dcrd#2703](https://github.com/leedeternal/dcrd/pull/2703))
- stake: Pre-allocate lottery ticket index slice ([leedeternal/dcrd#2710](https://github.com/leedeternal/dcrd/pull/2710))
- blockchain: Switch to treasuryValueType.IsDebit ([leedeternal/dcrd#2680](https://github.com/leedeternal/dcrd/pull/2680))
- blockchain: Sum amounts added to treasury ([leedeternal/dcrd#2680](https://github.com/leedeternal/dcrd/pull/2680))
- blockchain: Add maxTreasuryExpenditureDCP0007 ([leedeternal/dcrd#2680](https://github.com/leedeternal/dcrd/pull/2680))
- blockchain: Use new expenditure policy if activated ([leedeternal/dcrd#2680](https://github.com/leedeternal/dcrd/pull/2680))
- blockchain: Add checkTicketRedeemers ([leedeternal/dcrd#2702](https://github.com/leedeternal/dcrd/pull/2702))
- blockchain: Add NextExpiringTickets to BestState ([leedeternal/dcrd#2708](https://github.com/leedeternal/dcrd/pull/2708))
- multi: Add FetchUtxoEntry to mining Config ([leedeternal/dcrd#2709](https://github.com/leedeternal/dcrd/pull/2709))
- stake: Add func to create revocation from ticket ([leedeternal/dcrd#2707](https://github.com/leedeternal/dcrd/pull/2707))
- rpcserver: Use CreateRevocationFromTicket ([leedeternal/dcrd#2707](https://github.com/leedeternal/dcrd/pull/2707))
- multi: Don't use deprecated ioutil package ([leedeternal/dcrd#2722](https://github.com/leedeternal/dcrd/pull/2722))
- blockchain: Consolidate tx check flag construction ([leedeternal/dcrd#2716](https://github.com/leedeternal/dcrd/pull/2716))
- stake: Export Hash256PRNG UniformRandom ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- blockchain: Check auto revocations agenda state ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- multi: Add mempool IsAutoRevocationsAgendaActive ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- multi: Add auto revocations to agenda flags ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- multi: Check tx inputs auto revocations flag ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- blockchain: Add auto revocation error kinds ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- stake: Add auto revocation error kinds ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- blockchain: Move revocation checks block context ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- multi: Add isAutoRevocationsEnabled to CheckSSRtx ([leedeternal/dcrd#2719](https://github.com/leedeternal/dcrd/pull/2719))
- addrmgr: Remove deprecated code ([leedeternal/dcrd#2729](https://github.com/leedeternal/dcrd/pull/2729))
- peer: Remove deprecated DisableLog ([leedeternal/dcrd#2730](https://github.com/leedeternal/dcrd/pull/2730))
- database: Remove deprecated DisableLog ([leedeternal/dcrd#2731](https://github.com/leedeternal/dcrd/pull/2731))
- addrmgr: Decouple IP network checks from wire ([leedeternal/dcrd#2596](https://github.com/leedeternal/dcrd/pull/2596))
- addrmgr: Rename network address type ([leedeternal/dcrd#2596](https://github.com/leedeternal/dcrd/pull/2596))
- addrmgr: Decouple addrmgr from wire NetAddress ([leedeternal/dcrd#2596](https://github.com/leedeternal/dcrd/pull/2596))
- multi: add spend pruner ([leedeternal/dcrd#2641](https://github.com/leedeternal/dcrd/pull/2641))
- multi: synchronize spend prunes and notifications ([leedeternal/dcrd#2641](https://github.com/leedeternal/dcrd/pull/2641))
- blockchain: workSorterLess -> betterCandidate ([leedeternal/dcrd#2747](https://github.com/leedeternal/dcrd/pull/2747))
- mempool: Add HeaderByHash to Config ([leedeternal/dcrd#2720](https://github.com/leedeternal/dcrd/pull/2720))
- rpctest: Remove unused BlockVersion const ([leedeternal/dcrd#2754](https://github.com/leedeternal/dcrd/pull/2754))
- blockchain: Handle genesis auto revocation agenda ([leedeternal/dcrd#2755](https://github.com/leedeternal/dcrd/pull/2755))
- indexers: remove index manager ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- indexers: add index subscriber ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- indexers: refactor interfaces ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- indexers: async transaction index ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- indexers: update address index ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- indexers: async exists address index ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- multi: integrate index subscriber ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- multi: avoid using subscriber lifecycle in catchup ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- multi: remove spend deps on index disc. notifs ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- multi: copy snapshot pkScript ([leedeternal/dcrd#2219](https://github.com/leedeternal/dcrd/pull/2219))
- blockchain: Conditionally log difficulty retarget ([leedeternal/dcrd#2761](https://github.com/leedeternal/dcrd/pull/2761))
- multi: Use single latest checkpoint ([leedeternal/dcrd#2763](https://github.com/leedeternal/dcrd/pull/2763))
- blockchain: Move diff retarget log to connect ([leedeternal/dcrd#2765](https://github.com/leedeternal/dcrd/pull/2765))
- multi: source index notif. from block notif ([leedeternal/dcrd#2256](https://github.com/leedeternal/dcrd/pull/2256))
- server: fix wireToAddrmgrNetAddress data race ([leedeternal/dcrd#2758](https://github.com/leedeternal/dcrd/pull/2758))
- multi: Flush cache before fetching UTXO stats ([leedeternal/dcrd#2767](https://github.com/leedeternal/dcrd/pull/2767))
- blockchain: Don't use deprecated ioutil package ([leedeternal/dcrd#2769](https://github.com/leedeternal/dcrd/pull/2769))
- blockchain: Fix ticket db disconnect revocations ([leedeternal/dcrd#2768](https://github.com/leedeternal/dcrd/pull/2768))
- blockchain: Add convenience ancestor of func ([leedeternal/dcrd#2771](https://github.com/leedeternal/dcrd/pull/2771))
- blockchain: Use new ancestor of convenience func ([leedeternal/dcrd#2771](https://github.com/leedeternal/dcrd/pull/2771))
- blockchain: Remove unused latest blk locator func ([leedeternal/dcrd#2772](https://github.com/leedeternal/dcrd/pull/2772))
- blockchain: Remove unused next lottery data func ([leedeternal/dcrd#2773](https://github.com/leedeternal/dcrd/pull/2773))
- secp256k1: Correct 96-bit accum double overflow ([leedeternal/dcrd#2778](https://github.com/leedeternal/dcrd/pull/2778))
- blockchain: Further decouple upgrade code ([leedeternal/dcrd#2776](https://github.com/leedeternal/dcrd/pull/2776))
- blockchain: Add bulk import mode ([leedeternal/dcrd#2782](https://github.com/leedeternal/dcrd/pull/2782))
- multi: Remove flags from SyncManager ProcessBlock ([leedeternal/dcrd#2783](https://github.com/leedeternal/dcrd/pull/2783))
- netsync: Remove flags from processBlockMsg ([leedeternal/dcrd#2783](https://github.com/leedeternal/dcrd/pull/2783))
- multi: Remove flags from blockchain ProcessBlock ([leedeternal/dcrd#2783](https://github.com/leedeternal/dcrd/pull/2783))
- multi: Remove flags from ProcessBlockHeader ([leedeternal/dcrd#2785](https://github.com/leedeternal/dcrd/pull/2785))
- blockchain: Remove flags maybeAcceptBlockHeader ([leedeternal/dcrd#2785](https://github.com/leedeternal/dcrd/pull/2785))
- version: Use uint32 for major/minor/patch ([leedeternal/dcrd#2789](https://github.com/leedeternal/dcrd/pull/2789))
- wire: Write message header directly ([leedeternal/dcrd#2790](https://github.com/leedeternal/dcrd/pull/2790))
- stake: Correct treasury enabled vote discovery ([leedeternal/dcrd#2780](https://github.com/leedeternal/dcrd/pull/2780))
- blockchain: Correct treasury spend vote data ([leedeternal/dcrd#2780](https://github.com/leedeternal/dcrd/pull/2780))
- blockchain: UTXO database migration fix ([leedeternal/dcrd#2798](https://github.com/leedeternal/dcrd/pull/2798))
- blockchain: Handle zero-length UTXO backend state ([leedeternal/dcrd#2798](https://github.com/leedeternal/dcrd/pull/2798))
- mining: Remove unnecessary tx copy ([leedeternal/dcrd#2792](https://github.com/leedeternal/dcrd/pull/2792))
- multi: Use dcrutil Tx in NewTxDeepTxIns ([leedeternal/dcrd#2802](https://github.com/leedeternal/dcrd/pull/2802))
- indexers: synchronize index subscriber ntfn sends/receives ([leedeternal/dcrd#2806](https://github.com/leedeternal/dcrd/pull/2806))
- stdscript: Add exported MaxDataCarrierSizeV0 ([leedeternal/dcrd#2803](https://github.com/leedeternal/dcrd/pull/2803))
- stdscript: Add ProvablyPruneableScriptV0 ([leedeternal/dcrd#2803](https://github.com/leedeternal/dcrd/pull/2803))
- stdscript: Add num required sigs support ([leedeternal/dcrd#2805](https://github.com/leedeternal/dcrd/pull/2805))
- netsync: Remove unused RpcServer ([leedeternal/dcrd#2811](https://github.com/leedeternal/dcrd/pull/2811))
- netsync: Consolidate initial sync handling ([leedeternal/dcrd#2812](https://github.com/leedeternal/dcrd/pull/2812))
- stdscript: Add v0 p2pk-ed25519 extract ([leedeternal/dcrd#2807](https://github.com/leedeternal/dcrd/pull/2807))
- stdscript: Add v0 p2pk-schnorr-secp256k1 extract ([leedeternal/dcrd#2807](https://github.com/leedeternal/dcrd/pull/2807))
- stdscript: Add v0 p2pkh-ed25519 extract ([leedeternal/dcrd#2807](https://github.com/leedeternal/dcrd/pull/2807))
- stdscript: Add v0 p2pkh-schnorr-secp256k1 extract ([leedeternal/dcrd#2807](https://github.com/leedeternal/dcrd/pull/2807))
- stdscript: Add script to address conversion ([leedeternal/dcrd#2807](https://github.com/leedeternal/dcrd/pull/2807))
- stdscript: Move from internal/staging to txscript ([leedeternal/dcrd#2810](https://github.com/leedeternal/dcrd/pull/2810))
- mining: Convert to use stdscript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- mempool: Convert to use stdscript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- chaingen: Convert to use stdscript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- blockchain: Convert to use stdscript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- indexers: Convert to use stdscript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- indexers: Remove unused trsy enabled params ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript/sign: Convert to use stdscript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript/sign: Remove unused trsy enabled params ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- rpcserver: Convert to use stdscript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove deprecated ExtractAtomicSwapDataPushes ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused GenerateProvablyPruneableOut ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused MultiSigScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused MultisigRedeemScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused CalcMultiSigStats ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused IsMultisigScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused IsMultisigSigScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused ExtractPkScriptAltSigType ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused GetScriptClass ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused GetStakeOutSubclass ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused typeOfScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isTreasurySpendScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isMultisigScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused ExtractPkScriptAddrs ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused scriptHashToAddrs ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused pubKeyHashToAddrs ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isTreasuryAddScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractMultisigScriptDetails ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isStakeChangeScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isPubKeyHashScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isStakeRevocationScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isStakeGenScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isStakeSubmissionScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractStakeScriptHash ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractStakePubKeyHash ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isNullDataScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractPubKeyHash ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isPubKeyAltScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractPubKeyAltDetails ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isPubKeyScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractPubKey ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractUncompressedPubKey ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractCompressedPubKey ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isPubKeyHashAltScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused extractPubKeyHashAltDetails ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused isStandardAltSignatureType ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused MaxDataCarrierSize ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused ScriptClass ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused ErrNotMultisigScript ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused ErrTooManyRequiredSigs ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- txscript: Remove unused ErrTooMuchNullData ([leedeternal/dcrd#2808](https://github.com/leedeternal/dcrd/pull/2808))
- stdaddr: Use txscript for opcode definitions ([leedeternal/dcrd#2809](https://github.com/leedeternal/dcrd/pull/2809))
- stdscript: Add v0 stake-tagged p2pkh extract ([leedeternal/dcrd#2816](https://github.com/leedeternal/dcrd/pull/2816))
- stdscript: Add v0 stake-tagged p2sh extract ([leedeternal/dcrd#2816](https://github.com/leedeternal/dcrd/pull/2816))
- server: sync rebroadcast inv sends/receives ([leedeternal/dcrd#2814](https://github.com/leedeternal/dcrd/pull/2814))
- multi: Move last ann block from peer to netsync ([leedeternal/dcrd#2821](https://github.com/leedeternal/dcrd/pull/2821))
- uint256: Introduce package infrastructure ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add set from big endian bytes ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add set from little endian bytes ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add get big endian bytes ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add get little endian bytes ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add zero support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add uint32 casting support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add uint64 casting support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add equality comparison support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add less than comparison support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add less or equals comparison support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add greater than comparison support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add greater or equals comparison support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add general comparison support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add addition support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add subtraction support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add multiplication support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add squaring support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add division support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add negation support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add is odd support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise left shift support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise right shift support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise not support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise or support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise and support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise xor support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bit length support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add text formatting support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add conversion to stdlib big int support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add conversion from stdlib big int support ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add basic usage example ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- stake: Rename func to identify stake cmtmnt output ([leedeternal/dcrd#2824](https://github.com/leedeternal/dcrd/pull/2824))
- progresslog: Make header logging concurrent safe ([leedeternal/dcrd#2833](https://github.com/leedeternal/dcrd/pull/2833))
- netsync: Contiguous hashes for initial state reqs ([leedeternal/dcrd#2825](https://github.com/leedeternal/dcrd/pull/2825))
- multi: Allow discrete mining with invalidated tip ([leedeternal/dcrd#2838](https://github.com/leedeternal/dcrd/pull/2838))
- primitives: Add difficulty bits <-> uint256 ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))
- primitives: Add work calc from diff bits ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))
- primitives: Add hash to uint256 conversion ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))
- primitives: Add check proof of work ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))
- primitives: Add core merkle tree root calcs ([leedeternal/dcrd#2826](https://github.com/leedeternal/dcrd/pull/2826))
- primitives: Add inclusion proof funcs ([leedeternal/dcrd#2827](https://github.com/leedeternal/dcrd/pull/2827))
- indexers: update indexer error types ([leedeternal/dcrd#2770](https://github.com/leedeternal/dcrd/pull/2770))
- rpcserver: Submit transactions directly ([leedeternal/dcrd#2835](https://github.com/leedeternal/dcrd/pull/2835))
- netsync: Remove unused tx submission processing ([leedeternal/dcrd#2835](https://github.com/leedeternal/dcrd/pull/2835))
- internal/staging: add ban manager ([leedeternal/dcrd#2554](https://github.com/leedeternal/dcrd/pull/2554))
- uint256: Correct base 10 output formatting ([leedeternal/dcrd#2844](https://github.com/leedeternal/dcrd/pull/2844))
- multi: Add assumeValid to BlockChain ([leedeternal/dcrd#2839](https://github.com/leedeternal/dcrd/pull/2839))
- blockchain: Track assumed valid node ([leedeternal/dcrd#2839](https://github.com/leedeternal/dcrd/pull/2839))
- blockchain: Set BFFastAdd based on assume valid ([leedeternal/dcrd#2839](https://github.com/leedeternal/dcrd/pull/2839))
- blockchain: Assume valid skip script validation ([leedeternal/dcrd#2839](https://github.com/leedeternal/dcrd/pull/2839))
- blockchain: Bulk import skip script validation ([leedeternal/dcrd#2839](https://github.com/leedeternal/dcrd/pull/2839))
- hdkeychain: Add a strict BIP32 child derivation method ([leedeternal/dcrd#2845](https://github.com/leedeternal/dcrd/pull/2845))
- mempool: Consolidate tx check flag construction ([leedeternal/dcrd#2846](https://github.com/leedeternal/dcrd/pull/2846))
- standalone: Add modified subsidy split calcs ([leedeternal/dcrd#2848](https://github.com/leedeternal/dcrd/pull/2848))

### Developer-related module management:

- rpcclient: Prepare v6.0.1 ([leedeternal/dcrd#2455](https://github.com/leedeternal/dcrd/pull/2455))
- multi: Start blockchain v4 module dev cycle ([leedeternal/dcrd#2463](https://github.com/leedeternal/dcrd/pull/2463))
- multi: Start rpcclient v7 module dev cycle ([leedeternal/dcrd#2463](https://github.com/leedeternal/dcrd/pull/2463))
- multi: Start gcs v3 module dev cycle ([leedeternal/dcrd#2463](https://github.com/leedeternal/dcrd/pull/2463))
- multi: Start blockchain/stake v4 module dev cycle ([leedeternal/dcrd#2511](https://github.com/leedeternal/dcrd/pull/2511))
- multi: Start txscript v4 module dev cycle ([leedeternal/dcrd#2511](https://github.com/leedeternal/dcrd/pull/2511))
- multi: Start dcrutil v4 module dev cycle ([leedeternal/dcrd#2511](https://github.com/leedeternal/dcrd/pull/2511))
- multi: Start dcrec/secp256k1 v4 module dev cycle ([leedeternal/dcrd#2511](https://github.com/leedeternal/dcrd/pull/2511))
- rpc/jsonrpc/types: Start v3 module dev cycle ([leedeternal/dcrd#2517](https://github.com/leedeternal/dcrd/pull/2517))
- multi: Round 1 prerel module release ver updates ([leedeternal/dcrd#2569](https://github.com/leedeternal/dcrd/pull/2569))
- multi: Round 2 prerel module release ver updates ([leedeternal/dcrd#2570](https://github.com/leedeternal/dcrd/pull/2570))
- multi: Round 3 prerel module release ver updates ([leedeternal/dcrd#2572](https://github.com/leedeternal/dcrd/pull/2572))
- multi: Round 4 prerel module release ver updates ([leedeternal/dcrd#2573](https://github.com/leedeternal/dcrd/pull/2573))
- multi: Round 5 prerel module release ver updates ([leedeternal/dcrd#2574](https://github.com/leedeternal/dcrd/pull/2574))
- multi: Round 6 prerel module release ver updates ([leedeternal/dcrd#2575](https://github.com/leedeternal/dcrd/pull/2575))
- multi: Update to siphash v1.2.2 ([leedeternal/dcrd#2577](https://github.com/leedeternal/dcrd/pull/2577))
- peer: Start v3 module dev cycle ([leedeternal/dcrd#2585](https://github.com/leedeternal/dcrd/pull/2585))
- addrmgr: Start v2 module dev cycle ([leedeternal/dcrd#2592](https://github.com/leedeternal/dcrd/pull/2592))
- blockchain: Prerel module release ver updates ([leedeternal/dcrd#2634](https://github.com/leedeternal/dcrd/pull/2634))
- blockchain: Bump database module minor version ([leedeternal/dcrd#2654](https://github.com/leedeternal/dcrd/pull/2654))
- multi: Require last database/v2.0.3-x version ([leedeternal/dcrd#2689](https://github.com/leedeternal/dcrd/pull/2689))
- multi: Introduce database/v3 module ([leedeternal/dcrd#2689](https://github.com/leedeternal/dcrd/pull/2689))
- multi: Use database/v3 module ([leedeternal/dcrd#2693](https://github.com/leedeternal/dcrd/pull/2693))
- main: Use pseudo-versions in bumped mods ([leedeternal/dcrd#2698](https://github.com/leedeternal/dcrd/pull/2698))
- blockchain: Add replace to chaincfg dependency ([leedeternal/dcrd#2679](https://github.com/leedeternal/dcrd/pull/2679))
- dcrjson: Introduce v4 module ([leedeternal/dcrd#2733](https://github.com/leedeternal/dcrd/pull/2733))
- secp256k1: Prepare v4.0.0 ([leedeternal/dcrd#2732](https://github.com/leedeternal/dcrd/pull/2732))
- docs: Update for dcrjson v4 module ([leedeternal/dcrd#2734](https://github.com/leedeternal/dcrd/pull/2734))
- dcrjson: Prepare v4.0.0 ([leedeternal/dcrd#2734](https://github.com/leedeternal/dcrd/pull/2734))
- blockchain: Prerel module release ver updates ([leedeternal/dcrd#2748](https://github.com/leedeternal/dcrd/pull/2748))
- gcs: Prerel module release ver updates ([leedeternal/dcrd#2749](https://github.com/leedeternal/dcrd/pull/2749))
- multi: Update gcs prerel version ([leedeternal/dcrd#2750](https://github.com/leedeternal/dcrd/pull/2750))
- multi: update build tags to pref. go1.17 syntax ([leedeternal/dcrd#2764](https://github.com/leedeternal/dcrd/pull/2764))
- chaincfg: Prepare v3.1.0 ([leedeternal/dcrd#2799](https://github.com/leedeternal/dcrd/pull/2799))
- addrmgr: Prepare v2.0.0 ([leedeternal/dcrd#2797](https://github.com/leedeternal/dcrd/pull/2797))
- rpc/jsonrpc/types: Prepare v3.0.0 ([leedeternal/dcrd#2801](https://github.com/leedeternal/dcrd/pull/2801))
- txscript: Prepare v4.0.0 ([leedeternal/dcrd#2815](https://github.com/leedeternal/dcrd/pull/2815))
- hdkeychain: Prepare v3.0.1 ([leedeternal/dcrd#2817](https://github.com/leedeternal/dcrd/pull/2817))
- dcrutil: Prepare v4.0.0 ([leedeternal/dcrd#2818](https://github.com/leedeternal/dcrd/pull/2818))
- connmgr: Prepare v3.1.0 ([leedeternal/dcrd#2819](https://github.com/leedeternal/dcrd/pull/2819))
- peer: Prepare v3.0.0 ([leedeternal/dcrd#2820](https://github.com/leedeternal/dcrd/pull/2820))
- database: Prepare v3.0.0 ([leedeternal/dcrd#2822](https://github.com/leedeternal/dcrd/pull/2822))
- blockchain/stake: Prepare v4.0.0 ([leedeternal/dcrd#2824](https://github.com/leedeternal/dcrd/pull/2824))
- gcs: Prepare v3.0.0 ([leedeternal/dcrd#2830](https://github.com/leedeternal/dcrd/pull/2830))
- math/uint256: Prepare v1.0.0 ([leedeternal/dcrd#2842](https://github.com/leedeternal/dcrd/pull/2842))
- blockchain: Prepare v4.0.0 ([leedeternal/dcrd#2831](https://github.com/leedeternal/dcrd/pull/2831))
- rpcclient: Prepare v7.0.0 ([leedeternal/dcrd#2851](https://github.com/leedeternal/dcrd/pull/2851))
- version: Include VCS build info in version string ([leedeternal/dcrd#2841](https://github.com/leedeternal/dcrd/pull/2841))
- main: Update to use all new module versions ([leedeternal/dcrd#2853](https://github.com/leedeternal/dcrd/pull/2853))
- main: Remove module replacements ([leedeternal/dcrd#2855](https://github.com/leedeternal/dcrd/pull/2855))

### Testing and Quality Assurance:

- rpcserver: Add handleGetTreasuryBalance tests ([leedeternal/dcrd#2390](https://github.com/leedeternal/dcrd/pull/2390))
- rpcserver: Add handleGet{Generate,HashesPerSec} tests ([leedeternal/dcrd#2365](https://github.com/leedeternal/dcrd/pull/2365))
- mining: Cleanup txPriorityQueue tests ([leedeternal/dcrd#2431](https://github.com/leedeternal/dcrd/pull/2431))
- blockchain: fix errorlint warnings ([leedeternal/dcrd#2411](https://github.com/leedeternal/dcrd/pull/2411))
- rpcserver: Add handleGetHeaders test ([leedeternal/dcrd#2366](https://github.com/leedeternal/dcrd/pull/2366))
- rpcserver: add ticketsforaddress tests ([leedeternal/dcrd#2405](https://github.com/leedeternal/dcrd/pull/2405))
- rpcserver: add ticketvwap tests ([leedeternal/dcrd#2406](https://github.com/leedeternal/dcrd/pull/2406))
- rpcserver: add handleTxFeeInfo tests ([leedeternal/dcrd#2407](https://github.com/leedeternal/dcrd/pull/2407))
- rpcserver: add handleTicketFeeInfo tests ([leedeternal/dcrd#2408](https://github.com/leedeternal/dcrd/pull/2408))
- rpcserver: add handleVerifyMessage tests ([leedeternal/dcrd#2413](https://github.com/leedeternal/dcrd/pull/2413))
- rpcserver: add handleSendRawTransaction tests ([leedeternal/dcrd#2410](https://github.com/leedeternal/dcrd/pull/2410))
- rpcserver: add handleGetVoteInfo tests ([leedeternal/dcrd#2432](https://github.com/leedeternal/dcrd/pull/2432))
- database: Fix errorlint warnings ([leedeternal/dcrd#2484](https://github.com/leedeternal/dcrd/pull/2484))
- mining: Add mining test harness ([leedeternal/dcrd#2480](https://github.com/leedeternal/dcrd/pull/2480))
- mining: Add NewBlockTemplate tests ([leedeternal/dcrd#2480](https://github.com/leedeternal/dcrd/pull/2480))
- mining: Move TxMiningView tests to mining ([leedeternal/dcrd#2480](https://github.com/leedeternal/dcrd/pull/2480))
- rpcserver: add handleGetRawTransaction tests ([leedeternal/dcrd#2483](https://github.com/leedeternal/dcrd/pull/2483))
- blockchain: Improve synthetic treasury vote tests ([leedeternal/dcrd#2488](https://github.com/leedeternal/dcrd/pull/2488))
- rpcserver: Add handleGetMempoolInfo test ([leedeternal/dcrd#2492](https://github.com/leedeternal/dcrd/pull/2492))
- connmgr: Increase test timeouts ([leedeternal/dcrd#2505](https://github.com/leedeternal/dcrd/pull/2505))
- run_tests.sh: Avoid command substitution ([leedeternal/dcrd#2506](https://github.com/leedeternal/dcrd/pull/2506))
- mempool: Make sequence lock tests more consistent ([leedeternal/dcrd#2496](https://github.com/leedeternal/dcrd/pull/2496))
- mempool: Rework sequence lock acceptance tests ([leedeternal/dcrd#2496](https://github.com/leedeternal/dcrd/pull/2496))
- rpcserver: Add handleGetTxOut tests ([leedeternal/dcrd#2516](https://github.com/leedeternal/dcrd/pull/2516))
- rpcserver: Add handleGetNetworkHashPS test ([leedeternal/dcrd#2512](https://github.com/leedeternal/dcrd/pull/2512))
- rpcserver: Add handleGetMiningInfo test ([leedeternal/dcrd#2512](https://github.com/leedeternal/dcrd/pull/2512))
- blockchain: Simplify TestFixedSequenceLocks ([leedeternal/dcrd#2534](https://github.com/leedeternal/dcrd/pull/2534))
- chaingen: Support querying block test name by hash ([leedeternal/dcrd#2518](https://github.com/leedeternal/dcrd/pull/2518))
- blockchain: Improve test harness logging ([leedeternal/dcrd#2518](https://github.com/leedeternal/dcrd/pull/2518))
- blockchain: Support separate test block generation ([leedeternal/dcrd#2518](https://github.com/leedeternal/dcrd/pull/2518))
- rpcserver: add handleVersion, handleHelp rpc tests ([leedeternal/dcrd#2549](https://github.com/leedeternal/dcrd/pull/2549))
- blockchain: Use ReplaceVoteBits in utxoview tests ([leedeternal/dcrd#2553](https://github.com/leedeternal/dcrd/pull/2553))
- blockchain: Add unit test coverage for UtxoEntry ([leedeternal/dcrd#2553](https://github.com/leedeternal/dcrd/pull/2553))
- rpctest: Don't use installed node ([leedeternal/dcrd#2523](https://github.com/leedeternal/dcrd/pull/2523))
- apbf: Add comprehensive tests ([leedeternal/dcrd#2579](https://github.com/leedeternal/dcrd/pull/2579))
- apbf: Add benchmarks ([leedeternal/dcrd#2579](https://github.com/leedeternal/dcrd/pull/2579))
- rpcserver: Add handleGetRawMempool test ([leedeternal/dcrd#2589](https://github.com/leedeternal/dcrd/pull/2589))
- build: Test against go 1.16 ([leedeternal/dcrd#2598](https://github.com/leedeternal/dcrd/pull/2598))
- blockchain: Add test name to TestUtxoEntry errors ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- blockchain: Add UtxoCache test coverage ([leedeternal/dcrd#2591](https://github.com/leedeternal/dcrd/pull/2591))
- blockchain: Use new style for chainio test errors ([leedeternal/dcrd#2595](https://github.com/leedeternal/dcrd/pull/2595))
- rpcserver: Add handleInvalidateBlock test ([leedeternal/dcrd#2604](https://github.com/leedeternal/dcrd/pull/2604))
- blockchain: Mock time.Now for utxo cache tests ([leedeternal/dcrd#2605](https://github.com/leedeternal/dcrd/pull/2605))
- blockchain: Add UtxoCache Initialize tests ([leedeternal/dcrd#2599](https://github.com/leedeternal/dcrd/pull/2599))
- blockchain: Add TestShutdownUtxoCache tests ([leedeternal/dcrd#2599](https://github.com/leedeternal/dcrd/pull/2599))
- rpcserver: Add handleReconsiderBlock test ([leedeternal/dcrd#2613](https://github.com/leedeternal/dcrd/pull/2613))
- stdaddr: Add benchmarks ([leedeternal/dcrd#2610](https://github.com/leedeternal/dcrd/pull/2610))
- rpctest: Make tests work properly with latest code ([leedeternal/dcrd#2614](https://github.com/leedeternal/dcrd/pull/2614))
- mempool: Remove unused field from test struct ([leedeternal/dcrd#2618](https://github.com/leedeternal/dcrd/pull/2618))
- mempool: Remove unused func from tests ([leedeternal/dcrd#2621](https://github.com/leedeternal/dcrd/pull/2621))
- rpctest: Don't use Fatalf in goroutines ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- chaingen: Remove unused PurchaseCommitmentScript ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- rpctest: Convert to use new stdaddr package ([leedeternal/dcrd#2625](https://github.com/leedeternal/dcrd/pull/2625))
- dcrutil: Move address params iface and mock impls ([leedeternal/dcrd#2628](https://github.com/leedeternal/dcrd/pull/2628))
- stdscript: Add v0 p2pk-ecdsa-secp256k1 benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pk-ed25519 benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pk-schnorr-secp256k1 benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pkh-ecdsa-secp256k1 benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pkh-ed25519 benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2pkh-schnorr-secp256k1 benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 p2sh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 ecdsa multisig benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 ecdsa multisig redeem benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add extract v0 multisig redeem benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 nulldata benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake sub p2pkh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake sub p2sh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake gen p2pkh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake gen p2sh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake revoke p2pkh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake revoke p2sh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake change p2pkh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 stake change p2sh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 treasury add benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 treasury gen p2pkh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 treasury gen p2sh benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add determine script type benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- stdscript: Add v0 atomic swap redeem benchmark ([leedeternal/dcrd#2656](https://github.com/leedeternal/dcrd/pull/2656))
- txscript: Separate short form script parsing ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- txscript: Explicit consensus p2sh tests ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- txscript: Explicit consensus any kind p2sh tests ([leedeternal/dcrd#2666](https://github.com/leedeternal/dcrd/pull/2666))
- stake: No standardness code in tests ([leedeternal/dcrd#2667](https://github.com/leedeternal/dcrd/pull/2667))
- blockchain: Add outpointKey tests ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- blockchain: Add block index key collision tests ([leedeternal/dcrd#2685](https://github.com/leedeternal/dcrd/pull/2685))
- secp256k1: Rework NAF tests ([leedeternal/dcrd#2695](https://github.com/leedeternal/dcrd/pull/2695))
- secp256k1: Cleanup NAF benchmark ([leedeternal/dcrd#2695](https://github.com/leedeternal/dcrd/pull/2695))
- rpctest: Add P2PAddress() function ([leedeternal/dcrd#2704](https://github.com/leedeternal/dcrd/pull/2704))
- tests: Remove hardcoded CC=gcc from run_tests.sh ([leedeternal/dcrd#2706](https://github.com/leedeternal/dcrd/pull/2706))
- build: Test against Go 1.17 ([leedeternal/dcrd#2712](https://github.com/leedeternal/dcrd/pull/2712))
- blockchain: Support voting multiple agendas in test ([leedeternal/dcrd#2679](https://github.com/leedeternal/dcrd/pull/2679))
- blockchain: Single out treasury policy test ([leedeternal/dcrd#2679](https://github.com/leedeternal/dcrd/pull/2679))
- blockchain: Correct test harness err msg ([leedeternal/dcrd#2714](https://github.com/leedeternal/dcrd/pull/2714))
- blockchain: Test new max expenditure policy ([leedeternal/dcrd#2680](https://github.com/leedeternal/dcrd/pull/2680))
- chaingen: Add spendable coinbase out snapshots ([leedeternal/dcrd#2715](https://github.com/leedeternal/dcrd/pull/2715))
- mempool: Accept test mungers for create tickets ([leedeternal/dcrd#2721](https://github.com/leedeternal/dcrd/pull/2721))
- build: Don't set GO111MODULE unnecessarily ([leedeternal/dcrd#2722](https://github.com/leedeternal/dcrd/pull/2722))
- build: Don't manually test changing go.{mod,sum} ([leedeternal/dcrd#2722](https://github.com/leedeternal/dcrd/pull/2722))
- stake: Add CalculateRewards tests ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- stake: Add CheckSSRtx tests ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- blockchain: Test auto revocations deployment ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- chaingen: Add revocation mungers ([leedeternal/dcrd#2718](https://github.com/leedeternal/dcrd/pull/2718))
- addrmgr: Improve test coverage ([leedeternal/dcrd#2596](https://github.com/leedeternal/dcrd/pull/2596))
- addrmgr: Remove unnecessary test cases ([leedeternal/dcrd#2596](https://github.com/leedeternal/dcrd/pull/2596))
- rpcserver: Tune large tspend test amount ([leedeternal/dcrd#2679](https://github.com/leedeternal/dcrd/pull/2679))
- build: Pin GitHub Actions to SHA ([leedeternal/dcrd#2736](https://github.com/leedeternal/dcrd/pull/2736))
- blockchain: Add calcTicketReturnAmounts tests ([leedeternal/dcrd#2720](https://github.com/leedeternal/dcrd/pull/2720))
- blockchain: Add checkTicketRedeemers tests ([leedeternal/dcrd#2720](https://github.com/leedeternal/dcrd/pull/2720))
- blockchain: Add auto revocation validation tests ([leedeternal/dcrd#2720](https://github.com/leedeternal/dcrd/pull/2720))
- mining: Add auto revocation block template tests ([leedeternal/dcrd#2720](https://github.com/leedeternal/dcrd/pull/2720))
- mempool: Add tests with auto revocations enabled ([leedeternal/dcrd#2720](https://github.com/leedeternal/dcrd/pull/2720))
- txscript: Add versioned short form parsing ([leedeternal/dcrd#2756](https://github.com/leedeternal/dcrd/pull/2756))
- txscript: Test consistency and cleanup ([leedeternal/dcrd#2757](https://github.com/leedeternal/dcrd/pull/2757))
- mempool: Add blockHeight to AddFakeUTXO for tests ([leedeternal/dcrd#2804](https://github.com/leedeternal/dcrd/pull/2804))
- mempool: Test fraud proof handling ([leedeternal/dcrd#2804](https://github.com/leedeternal/dcrd/pull/2804))
- stdscript: Add extract v0 stake-tagged p2pkh bench ([leedeternal/dcrd#2816](https://github.com/leedeternal/dcrd/pull/2816))
- stdscript: Add extract v0 stake-tagged p2sh bench ([leedeternal/dcrd#2816](https://github.com/leedeternal/dcrd/pull/2816))
- mempool: Update test to check hash value ([leedeternal/dcrd#2804](https://github.com/leedeternal/dcrd/pull/2804))
- stdscript: Add num required sigs benchmark ([leedeternal/dcrd#2805](https://github.com/leedeternal/dcrd/pull/2805))
- uint256: Add big endian set benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add little endian set benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add big endian get benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add little endian get benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add zero benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add equality comparison benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add less than comparison benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add greater than comparison benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add general comparison benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add addition benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add subtraction benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add multiplication benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add squaring benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add division benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add negation benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add is odd benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise left shift benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise right shift benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise not benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise or benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise and benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bitwise xor benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add bit length benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add text formatting benchmarks ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add conversion to stdlib big int benchmark ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- uint256: Add conversion from stdlib big int benchmark ([leedeternal/dcrd#2787](https://github.com/leedeternal/dcrd/pull/2787))
- primitives: Add diff bits conversion benchmarks ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))
- primitives: Add work calc benchmark ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))
- primitives: Add hash to uint256 benchmark ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))
- primitives: Add check proof of work benchmark ([leedeternal/dcrd#2788](https://github.com/leedeternal/dcrd/pull/2788))
- primitives: Add merkle root benchmarks ([leedeternal/dcrd#2826](https://github.com/leedeternal/dcrd/pull/2826))
- primitives: Add inclusion proof benchmarks ([leedeternal/dcrd#2827](https://github.com/leedeternal/dcrd/pull/2827))
- blockchain: Add AssumeValid tests ([leedeternal/dcrd#2839](https://github.com/leedeternal/dcrd/pull/2839))
- chaingen: Add vote subsidy munger ([leedeternal/dcrd#2848](https://github.com/leedeternal/dcrd/pull/2848))

### Misc:

- release: Bump for 1.7 release cycle ([leedeternal/dcrd#2429](https://github.com/leedeternal/dcrd/pull/2429))
- secp256k1: Correct const name for doc comment ([leedeternal/dcrd#2445](https://github.com/leedeternal/dcrd/pull/2445))
- multi: Fix various typos ([leedeternal/dcrd#2607](https://github.com/leedeternal/dcrd/pull/2607))
- rpcserver: Fix createrawssrtx comments ([leedeternal/dcrd#2665](https://github.com/leedeternal/dcrd/pull/2665))
- blockchain: Fix comment formatting in generator ([leedeternal/dcrd#2665](https://github.com/leedeternal/dcrd/pull/2665))
- stake: Fix MaxOutputsPerSSRtx comment ([leedeternal/dcrd#2665](https://github.com/leedeternal/dcrd/pull/2665))
- stake: Fix CheckSSGenVotes function comment ([leedeternal/dcrd#2665](https://github.com/leedeternal/dcrd/pull/2665))
- stake: Fix CheckSSRtx function comment ([leedeternal/dcrd#2665](https://github.com/leedeternal/dcrd/pull/2665))
- database: Add comment on os.MkdirAll behavior ([leedeternal/dcrd#2670](https://github.com/leedeternal/dcrd/pull/2670))
- multi: Address some linter complaints ([leedeternal/dcrd#2684](https://github.com/leedeternal/dcrd/pull/2684))
- txscript: Fix a couple of a comment typos ([leedeternal/dcrd#2692](https://github.com/leedeternal/dcrd/pull/2692))
- blockchain: Remove inapplicable comment ([leedeternal/dcrd#2742](https://github.com/leedeternal/dcrd/pull/2742))
- mining: Fix error in comment ([leedeternal/dcrd#2743](https://github.com/leedeternal/dcrd/pull/2743))
- blockchain: Fix several typos ([leedeternal/dcrd#2745](https://github.com/leedeternal/dcrd/pull/2745))
- blockchain: Update a few BFFastAdd comments ([leedeternal/dcrd#2781](https://github.com/leedeternal/dcrd/pull/2781))
- multi: Address some linter complaints ([leedeternal/dcrd#2791](https://github.com/leedeternal/dcrd/pull/2791))
- netsync: Correct typo ([leedeternal/dcrd#2813](https://github.com/leedeternal/dcrd/pull/2813))
- netsync: Fix misc typos ([leedeternal/dcrd#2834](https://github.com/leedeternal/dcrd/pull/2834))
- mining: Fix typo ([leedeternal/dcrd#2834](https://github.com/leedeternal/dcrd/pull/2834))
- blockchain: Correct comment typos for find fork ([leedeternal/dcrd#2828](https://github.com/leedeternal/dcrd/pull/2828))
- rpcserver: Rename var to make linter happy ([leedeternal/dcrd#2835](https://github.com/leedeternal/dcrd/pull/2835))
- blockchain: Wrap at max line length ([leedeternal/dcrd#2843](https://github.com/leedeternal/dcrd/pull/2843))
- release: Bump for 1.7.0 ([leedeternal/dcrd#2856](https://github.com/leedeternal/dcrd/pull/2856))

### Code Contributors (alphabetical order):

- briancolecoinmetrics
- Dave Collins
- David Hill
- degeri
- Donald Adu-Poku
- J Fixby
- Jamie Holdstock
- Joe Gruffins
- Jonathan Chappelow
- Josh Rickmar
- lolandhold
- Matheus Degiovani
- Naveen
- Ryan Staudt
- Youssef Boukenken
- Wisdom Arerosuoghene
