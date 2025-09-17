# dcrd v1.8.0

This is a new major release of dcrd.  Some of the key highlights are:

* Two new consensus vote agendas which allow stakeholders to decide whether or
  not to activate support for the following:
  * Changing the Proof-of-Work hashing algorithm to BLAKE3 and the difficulty algorithm to ASERT
  * Changing the Proof-of-Work and Proof-of-Stake subsidy split from 10%/80% to 1%/89%
* Separation of block hash from Proof-of-Work hash
* BLAKE3 CPU mining support
* Initial sync time reduced by about 20%
* Runtime memory management optimizations
* Faster cryptographic signature validation
* Low fee transaction rejection
* Unspent transaction output set size reduction
* No more checkpoints
* Improved network protocol message responsiveness
* Header proof commitment hash storage
* Address index removal
* Several CLI options deprecated
* Various updates to the RPC server:
  * Total coin supply output correction
  * More stable global communication over WebSockets
  * Winning ticket notifications when unsynced mining on test networks
  * Several other notable updates, additions, and removals related to the JSON-RPC API
* Infrastructure improvements
* Miscellaneous network and protocol optimizations
* Quality assurance changes

For those unfamiliar with the
[voting process](https://docs.decred.org/governance/consensus-rule-voting/overview/)
in Decred, all code needed in order to support each of the aforementioned
consensus changes is already included in this release, however it will remain
dormant until the stakeholders vote to activate it.

For reference, the consensus change work was originally proposed and approved
for initial implementation via the following Politeia proposal:
- [Change PoW/PoS Subsidy Split to 1/89 and Change PoW Algorithm to BLAKE3](https://proposals.decred.org/record/a8501bc)

The following Decred Change Proposals (DCPs) describe the proposed changes in
detail and provide full technical specifications:
- [DCP0011: Change PoW to BLAKE3 and ASERT](https://github.com/leedeternal/dcps/blob/master/dcp-0011/dcp-0011.mediawiki)
- [DCP0012: Change PoW/PoS Subsidy Split To 1/89](https://github.com/leedeternal/dcps/blob/master/dcp-0012/dcp-0012.mediawiki)

## Upgrade Required

**It is extremely important for everyone to upgrade their software to this
latest release even if you don't intend to vote in favor of the agenda.  This
particularly applies to PoW miners as failure to upgrade will result in lost
rewards after block height 777240.  That is estimated to be around June 29th,
2023.**

## Downgrade Warning

The database format in v1.8.0 is not compatible with previous versions of the
software.  This only affects downgrades as users upgrading from previous
versions will see a one time database migration.

Once this migration has been completed, it will no longer be possible to
downgrade to a previous version of the software without having to delete the
database and redownload the chain.

The database migration typically takes around 4-6 minutes on HDDs and 2-3
minutes on SSDs.

## Notable Changes

### Two New Consensus Change Votes

Two new consensus change votes are now available as of this release.  After
upgrading, stakeholders may set their preferences through their wallet.

#### Change PoW to BLAKE3 and ASERT

The first new vote available as of this release has the id `blake3pow`.

The primary goals of this change are to:

* Increase decentralization of proof of work mining by obsoleting the current
  specialized hardware (ASICs) that is only realistically available to the
  existing highly centralized mining monopoly
* Improve the proof of work mining difficulty adjustment algorithm responsiveness
* Provide more equal profitability to steady state PoW miners versus hit and run
  miners

See the following for more details:

* [Politeia proposal](https://proposals.decred.org/record/a8501bc)
* [DCP0011](https://github.com/leedeternal/dcps/blob/master/dcp-0011/dcp-0011.mediawiki)

#### Change PoW/PoS Subsidy Split to 1/89 Vote

The second new vote available as of this release has the id `changesubsidysplitr2`.

The proposed modification to the subsidy split in tandem with the change to the
PoW hashing function is intended to break up the mining cartel and further
improve decentralization of the issuance process.

See the following for more details:

* [Politeia proposal](https://proposals.decred.org/record/a8501bc)
* [DCP0012](https://github.com/leedeternal/dcps/blob/master/dcp-0012/dcp-0012.mediawiki)

### Separation of Block Hash from Proof-of-Work Hash

A new Proof-of-Work (PoW) hash that is distinct from the existing block hash is
now used for all consensus rules related to PoW verification.

Block hashes have historically served multiple roles which include those related
to proof of work (PoW).  As of this release, the roles related to PoW are now
solely the domain of the new PoW hash.

Some key points related to this change are:

* The new PoW hash will be exactly the same as the existing block hash for all
  blocks prior to the activation of the stakeholder vote to change the PoW
  hashing algorithm
* The block hash continues to use the existing hashing algorithm
* The block hash will no longer have the typical pattern of leading zeros upon
  activation of the PoW hashing algorithm
* The PoW hash will have the typical pattern of leading zeros both before and
  after the activation of the new PoW hashing algorithm

### BLAKE3 CPU Mining Support

The internal CPU miner has been significantly optimized to provide much higher
hash rates, especially when using multiple cores, and now automatically mines
using the BLAKE3 algorithm when the `blake3pow` agenda is active.

### Initial Sync Time Reduced by About 20%

The amount of time it takes to complete the initial chain synchronization
process with default settings has been reduced by about 20% versus the previous
release.

### Runtime Memory Management Optimizations

The way memory is managed has been optimized to provide performance enhancements
to both steady-state operation as well as the initial chain sync process.

The primary benefits are:

* Lower maximum memory usage during transient periods of high demand
* Approximately a 10% reduction to the duration of the initial sync process
* Significantly reduced overall total memory allocations (~42%)
* Less overall CPU usage for the same amount of work

### Faster Cryptographic Signature Validation

Similar to the previous release, this release further improves some aspects of
the underlying crypto code to increase its execution speed and reduce the number
of memory allocations.  The overall result is a 52% reduction in allocations and
about a 1% reduction to the verification time for a single signature.

The primary benefits are:

* Improved vote times since blocks and transactions propagate more quickly
  throughout the network
* Approximately a 4% reduction to the duration of the initial sync process

### Low Fee Transaction Rejection

The default transaction acceptance and relay policy is no longer based on
priority and instead now immediately rejects all transactions that do not pay
the minimum required fee.

This provides a better user experience for transactions that do not pay enough
fees.

For some insight into the motivation for this change, prior to the introduction
of support for child pays for parent (CPFP), it was possible for transactions to
essentially become stuck forever if they didn't pay a high enough fee for miners
to include them in a block.

In order to prevent this, a policy was introduced that allowed relaying
transactions that do not pay enough fees based on a priority calculated from the
fee as well as the age of coins being spent.  The result is that the priority
slowly increased over time as the coins aged to ensure such transactions would
eventually be relayed and mined.  In order to prevent abuse the behavior could
otherwise allow, the policy also included additional rate-limiting of these
types of transactions.

While the policy served its purpose, it had some downsides such as:

* A confusing user experience where transactions that do not pay enough fees and
  also are not old enough to meet the dynamically changing priority requirements
  are rejected due to having insufficient priority instead of not paying enough
  fees as the user might expect
* The priority requirements dynamically change over time which leads to
  non-deterministic behavior and thus ultimately results in what appear to be
  intermittent/transient failures to users

The policy is no longer necessary or desirable given such transactions can now
use CPFP to increase the overall fee of the entire transaction chain thereby
ensuring they are mined.

### Unspent Transaction Output Set Size Reduction

The set of all unspent transaction outputs (UTXO set) no longer contains
unspendable `treasurybase` outputs.

A `treasurybase` output is a special output that increases the balance of the
decentralized treasury account which requires stakeholder approval to spend
funds.  As a result, they do not operate like normal transaction outputs and
therefore are never directly spendable.

Removing these unspendable outputs from the UTXO set reduces its overall size.

### No More Checkpoints

This release introduces a new model for deciding when to reject old forks to
make use of the hard-coded assumed valid block that is updated with each release
to a recent block thereby removing the final remaining usage of checkpoints.

Consequently, the `--nocheckpoints` command-line option and separate
`findcheckpoints` utility have been removed.

### Improved Network Protocol Message Responsiveness (`getheaders`/`getcfilterv2`)

All protocol message requests for headers (`getheaders`) and version 2 compact
filters (`getcfilterv2`) will now receive empty responses when there is not any
available data or the peer is otherwise unwilling to serve the data for a
variety of reasons.

For example, a peer might be unwilling to serve data because they are still
performing the initial sync or temporarily no longer consider themselves synced
with the network due to recently coming back online after being unable to
communicate with the network for a long time.

This change helps improve network robustness by preventing peers from appearing
unresponsive or stalled in such cases.

### Header Proof Commitment Hash Storage

The individual commitment hashes covered by the commitment root field of the
header of each block are now stored in the database for fast access.  This
provides better scaling for generating and serving inclusion proofs as more
commitments are added to the header proof in future upgrades.

### Address Index Removal (`--addrindex`, `--dropaddrindex`)

The previously deprecated optional address index that could be enabled via
`--addrindex` and removed via `--dropaddrindex` is no longer available.  All of
the information previously provided from the address index, and much more, is
available via [dcrdata](https://github.com/leedeternal/dcrdata/).

### Several CLI Options Deprecated

The following CLI options no longer have any effect and are now deprecated:

* `--norelaypriority`
* `--limitfreerelay`
* `--blockminsize`
* `--blockprioritysize`

They will be removed in a future release.

### RPC Server Changes

The RPC server version as of this release is 8.0.0.

#### Total Coin Supply Output Correction (`getcoinsupply`)

The total coin supply reported by `getcoinsupply` will now correctly include the
coins generated as a part of the block reward for the decentralized treasury as
intended.

As a result, the amount reported will now be higher than it was previously.  It
is important to note that this issue was only an RPC display issue and did not
affect consensus in any way.

#### More Stable Global Communication over WebSockets

WebSocket connections now have longer timeouts and remain connected through
transient network timeouts.  This significantly improves the stability of
high-latency connections such as those communicating across multiple continents.

#### Winning Ticket Notifications when Unsynced Mining on Test Networks (`winningtickets`)

Clients that subscribe to receive `winningtickets` notifications via WebSockets
with `notifywinningtickets` will now also receive the notifications on test
networks prior to being fully synced when the `--allowunsyncedmining` CLI option
is provided.

See the following for API details:

* [notifywinningtickets JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#notifywinningtickets)
* [winningtickets JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#winningtickets)

#### Transaction Fee Priority Fields on Mempool RPC Deprecated (`getrawmempool`)

Due to the removal of the policy related to low fee transaction priority, the
`startingpriority` and `currentpriority` fields of the results of the verbose
output of the `getrawmempool` RPC are now deprecated.  They will always be set
to 0 and are scheduled to be removed in a future version.

See the
[getrawmempool JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getrawmempool)
for API details.

#### Removal of Raw Transaction Search RPC (`searchrawtransactions`)

The deprecated `searchrawtransactions` RPC, which could previously be used to
obtain all transactions that either credit or debit a given address via RPC is
no longer available.

Callers that wish to access details related to addresses are encouraged to use
[dcrdata](https://github.com/leedeternal/dcrdata/) instead.

#### Removal of Address Index Status Field on Info RPC (`getinfo`)

The `addrindex` field of the `getinfo` RPC is no longer available.

See the
[getinfo JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getinfo)
for API details.

#### Removal of Missed and Expired Ticket RPCs

Now that missed and expired tickets are automatically revoked by the consensus
rules, all RPCs related to querying and requesting notifications for missed and
expired tickets are no longer available.

In particular, the following deprecated RPCs are no longer available:

* `missedtickets`
* `rebroadcastmissed`
* `existsmissedtickets`
* `existsexpiredtickets`
* `notifyspentandmissedtickets`

#### Updates to Work RPC (`getwork`)

The `getwork` RPC will now return an error message immediately if block template
generation is temporarily unable to generate a template indicating the reason.
Previously, the RPC would block until a new template was eventually generated
which could potentially be an exceedingly long time.

Additionally, cancelling a `getwork` invocation before the work has been fully
generated will now cancel the underlying request which allows the RPC server to
immediately service other queued work requests.

See the
[getwork JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getwork)
for API details.

## Changelog

This release consists of 439 git commits from 18 contributors which total to 408
files changed, 25840 additional lines of code, and 22871 deleted lines of code.

All commits since the last release may be viewed on GitHub
[here](https://github.com/leedeternal/dcrd/compare/release-v1.7.7...release-v1.8.0).

### Protocol and network:

- server: Force PoW upgrade to v9 ([leedeternal/dcrd#2874](https://github.com/leedeternal/dcrd/pull/2874))
- blockchain: Optimize old block ver upgrade checks ([leedeternal/dcrd#2912](https://github.com/leedeternal/dcrd/pull/2912))
- server: Fix syncNotified race ([leedeternal/dcrd#2932](https://github.com/leedeternal/dcrd/pull/2932))
- multi: Rework old fork rejection logic ([leedeternal/dcrd#2945](https://github.com/leedeternal/dcrd/pull/2945))
- blockchain: Implement header proof storage ([leedeternal/dcrd#2938](https://github.com/leedeternal/dcrd/pull/2938))
- chaincfg: Remove planetdecred seeders ([leedeternal/dcrd#2974](https://github.com/leedeternal/dcrd/pull/2974))
- netsync: Improve sync height tracking ([leedeternal/dcrd#2978](https://github.com/leedeternal/dcrd/pull/2978))
- blockchain: Enforce testnet difficulty throttling ([leedeternal/dcrd#2978](https://github.com/leedeternal/dcrd/pull/2978))
- chaincfg: Deprecate min diff reduction params ([leedeternal/dcrd#2978](https://github.com/leedeternal/dcrd/pull/2978))
- blockchain: Don't add treasurybase utxos ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Remove unspendable utxo set entries ([leedeternal/dcrd#2996](https://github.com/leedeternal/dcrd/pull/2996))
- server: Update peer attempt timestamp before dialing ([leedeternal/dcrd#3014](https://github.com/leedeternal/dcrd/pull/3014))
- server: Bump supported wire pver for reject removal ([leedeternal/dcrd#3017](https://github.com/leedeternal/dcrd/pull/3017))
- peer: Use latest pver by default ([leedeternal/dcrd#3019](https://github.com/leedeternal/dcrd/pull/3019))
- server: Always respond to getheaders ([leedeternal/dcrd#3030](https://github.com/leedeternal/dcrd/pull/3030))
- server: Always serve known getcfilterv2 filters ([leedeternal/dcrd#3035](https://github.com/leedeternal/dcrd/pull/3035))
- chaincfg: Enforce globally unique vote IDs ([leedeternal/dcrd#3057](https://github.com/leedeternal/dcrd/pull/3057))
- multi: Add forced deployment result network param ([leedeternal/dcrd#3060](https://github.com/leedeternal/dcrd/pull/3060))
- peer: Correct known inventory check ([leedeternal/dcrd#3074](https://github.com/leedeternal/dcrd/pull/3074))
- netsync: Re-request data sooner after peer disconnect ([leedeternal/dcrd#3067](https://github.com/leedeternal/dcrd/pull/3067))
- chaincfg: Introduce BLAKE3 PoW agenda ([leedeternal/dcrd#3089](https://github.com/leedeternal/dcrd/pull/3089))
- chaincfg: Introduce subsidy split change r2 agenda ([leedeternal/dcrd#3090](https://github.com/leedeternal/dcrd/pull/3090))
- multi: Implement DCP0012 subsidy consensus vote ([leedeternal/dcrd#3092](https://github.com/leedeternal/dcrd/pull/3092))
- multi: Separate block hash and proof of work hash ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- chaincfg: Add params for DCP0011 blake3 diff calc ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- multi: Implement DCP0011 PoW hash consensus vote ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- chaincfg: Update assume valid for release ([leedeternal/dcrd#3122](https://github.com/leedeternal/dcrd/pull/3122))
- chaincfg: Update min known chain work for release ([leedeternal/dcrd#3123](https://github.com/leedeternal/dcrd/pull/3123))
- blockchain: Reject old block vers for pow vote ([leedeternal/dcrd#3135](https://github.com/leedeternal/dcrd/pull/3135))
- server: Force PoW upgrade to v10 ([leedeternal/dcrd#3137](https://github.com/leedeternal/dcrd/pull/3137))

### Transaction relay (memory pool):

- mempool: Invert reorg transaction handling ([leedeternal/dcrd#2956](https://github.com/leedeternal/dcrd/pull/2956))
- mempool: Explicitly reject standalone treasurybase ([leedeternal/dcrd#2963](https://github.com/leedeternal/dcrd/pull/2963))
- mempool: Do not accept low-fee/free transactions ([leedeternal/dcrd#2964](https://github.com/leedeternal/dcrd/pull/2964))
- mempool: Remove unused verbose tx curprio field ([leedeternal/dcrd#3003](https://github.com/leedeternal/dcrd/pull/3003))
- mempool: Remove unused tx desc starting prio field ([leedeternal/dcrd#3003](https://github.com/leedeternal/dcrd/pull/3003))

### Mining:

- mining: Use test net block ver on simnet ([leedeternal/dcrd#2868](https://github.com/leedeternal/dcrd/pull/2868))
- mining: Wait for initial sync to generate template ([leedeternal/dcrd#2897](https://github.com/leedeternal/dcrd/pull/2897))
- cpuminer: Rework speed stat tracking ([leedeternal/dcrd#2977](https://github.com/leedeternal/dcrd/pull/2977))
- cpuminer: Significantly optimize mining workers ([leedeternal/dcrd#2977](https://github.com/leedeternal/dcrd/pull/2977))
- mining: Remove high prio/free tx mining code ([leedeternal/dcrd#3003](https://github.com/leedeternal/dcrd/pull/3003))
- mining: Remove testnet min diff reduction support ([leedeternal/dcrd#3109](https://github.com/leedeternal/dcrd/pull/3109))
- mining: Update to latest block vers for pow vote ([leedeternal/dcrd#3136](https://github.com/leedeternal/dcrd/pull/3136))

### RPC:

- rpcserver: Update websocket ping timeout handling ([leedeternal/dcrd#2865](https://github.com/leedeternal/dcrd/pull/2865))
- rpcserver: Fix websocket auth failure ([leedeternal/dcrd#2877](https://github.com/leedeternal/dcrd/pull/2877))
- rpcserver: Remove missedtickets RPC ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- rpcserver: Remove rebroadcastmissed RPC ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- rpcserver: Remove existsmissedtickets RPC ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- rpcserver: Remove existsexpiredtickets RPC ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- rpcserver: Remove notifyspentandmissedtickets RPC ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- rpcserver: Remove searchrawtransactions ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- rpcserver: Remove unused AddrIndexer interface ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- rpcserver: Remove getinfo addrindex field ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- rpcserver: Return template errors from getwork RPC ([leedeternal/dcrd#2978](https://github.com/leedeternal/dcrd/pull/2978))
- server: Send winning tickets when unsynced mining ([leedeternal/dcrd#2978](https://github.com/leedeternal/dcrd/pull/2978))
- rpcserver: Return 0 for deprecated priority fields ([leedeternal/dcrd#3003](https://github.com/leedeternal/dcrd/pull/3003))
- rpcserver: Support getwork cancellation ([leedeternal/dcrd#3027](https://github.com/leedeternal/dcrd/pull/3027))
- rpcserver: Remove unused method refs from limited ([leedeternal/dcrd#3033](https://github.com/leedeternal/dcrd/pull/3033))
- rpcserver: Decouple RPC agenda info status strings ([leedeternal/dcrd#3071](https://github.com/leedeternal/dcrd/pull/3071))
- blockchain: Correct total subsidy calculation ([leedeternal/dcrd#3112](https://github.com/leedeternal/dcrd/pull/3112))

### dcrd command-line flags and configuration:

- server/indexers: Remove address index support ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- config: Deprecate norelaypriority CLI option ([leedeternal/dcrd#2964](https://github.com/leedeternal/dcrd/pull/2964))
- config: Deprecate limitfreerelay CLI option ([leedeternal/dcrd#2964](https://github.com/leedeternal/dcrd/pull/2964))
- config: Deprecate min block size CLI option ([leedeternal/dcrd#3002](https://github.com/leedeternal/dcrd/pull/3002))
- config: Deprecate block prio size CLI option ([leedeternal/dcrd#3003](https://github.com/leedeternal/dcrd/pull/3003))
- config: Minor description consistency cleanup ([leedeternal/dcrd#3041](https://github.com/leedeternal/dcrd/pull/3041))

### dcrd server runtime interface changes:

- dcrd: Support SIGTERM on Win and all unix variants ([leedeternal/dcrd#2958](https://github.com/leedeternal/dcrd/pull/2958))
- main: Remove no longer needed max cores config ([leedeternal/dcrd#3016](https://github.com/leedeternal/dcrd/pull/3016))
- main: Tweak runtime GC params for Go 1.19 ([leedeternal/dcrd#3016](https://github.com/leedeternal/dcrd/pull/3016))
- server: Add bound addresses IPC events ([leedeternal/dcrd#3020](https://github.com/leedeternal/dcrd/pull/3020))

### findcheckpoint utility changes:

- findcheckpoints: Remove utility ([leedeternal/dcrd#2945](https://github.com/leedeternal/dcrd/pull/2945))

### Documentation:

- docs: Add release notes for v1.7.0 ([leedeternal/dcrd#2858](https://github.com/leedeternal/dcrd/pull/2858))
- docs: Add release notes for v1.7.1 ([leedeternal/dcrd#2881](https://github.com/leedeternal/dcrd/pull/2881))
- docs: Update Min Recommended Disc Space ([leedeternal/dcrd#2906](https://github.com/leedeternal/dcrd/pull/2906))
- docs: Update doc.go with latest arguments ([leedeternal/dcrd#2924](https://github.com/leedeternal/dcrd/pull/2924))
- docs: add backport documentation ([leedeternal/dcrd#2934](https://github.com/leedeternal/dcrd/pull/2934))
- primitives: Update README.md for subsidy calcs ([leedeternal/dcrd#2933](https://github.com/leedeternal/dcrd/pull/2933))
- docs: Remove missedtickets JSON-RPC API ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- docs: Remove rebroadcastmissed JSON-RPC API ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- docs: Remove existsmissedtickets JSON-RPC API ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- docs: Remove existsexpiredtickets JSON-RPC API ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- docs: Remove (notify)spentandmissed JSON-RPC API ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- docs: Add release notes for v1.7.2 ([leedeternal/dcrd#2940](https://github.com/leedeternal/dcrd/pull/2940))
- blockchain: Comment concurrency semantics ([leedeternal/dcrd#2946](https://github.com/leedeternal/dcrd/pull/2946))
- docs: Remove searchrawtransactions JSON-RPC API ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- docs: Remove getinfo addrindex field JSON-RPC API ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- docs: Update indexers readme module path ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- docs: Update indexers readme for removed indexes ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- docs: Update chaingen readme module path ([leedeternal/dcrd#2952](https://github.com/leedeternal/dcrd/pull/2952))
- mempool/docs: Update low-fee/free tx policy removal ([leedeternal/dcrd#2964](https://github.com/leedeternal/dcrd/pull/2964))
- docs: Update dcrutil/v3 references to v4 ([leedeternal/dcrd#2980](https://github.com/leedeternal/dcrd/pull/2980))
- docs: Add release notes for v1.7.4 ([leedeternal/dcrd#2982](https://github.com/leedeternal/dcrd/pull/2982))
- docs: Update README.md to required Go 1.18/1.19 ([leedeternal/dcrd#2981](https://github.com/leedeternal/dcrd/pull/2981))
- rpcserver: Remove unused result help text ([leedeternal/dcrd#3001](https://github.com/leedeternal/dcrd/pull/3001))
- docs: Deprecate JSON-RPC API getrawmempool prio ([leedeternal/dcrd#3003](https://github.com/leedeternal/dcrd/pull/3003))
- docs: Add release notes for v1.7.5 ([leedeternal/dcrd#3009](https://github.com/leedeternal/dcrd/pull/3009))
- rpcserver: Update rescan help to match reality ([leedeternal/dcrd#3032](https://github.com/leedeternal/dcrd/pull/3032))
- docs: Make JSON-RPC rescan docs match reality ([leedeternal/dcrd#3032](https://github.com/leedeternal/dcrd/pull/3032))
- docs: Remove {stop,}notifyreceived JSON-RPC API ([leedeternal/dcrd#3034](https://github.com/leedeternal/dcrd/pull/3034))
- docs: Remove recvtx ntfn JSON-RPC API ([leedeternal/dcrd#3034](https://github.com/leedeternal/dcrd/pull/3034))
- docs: Remove {stop,}notifyspent JSON-RPC API ([leedeternal/dcrd#3034](https://github.com/leedeternal/dcrd/pull/3034))
- docs: Remove redeemingtx ntfn JSON-RPC API ([leedeternal/dcrd#3034](https://github.com/leedeternal/dcrd/pull/3034))
- docs: Update chaincfg examples ([leedeternal/dcrd#3040](https://github.com/leedeternal/dcrd/pull/3040))
- docs: Don't use deprecated ioutil package ([leedeternal/dcrd#3046](https://github.com/leedeternal/dcrd/pull/3046))
- docs: Update README.md to required Go 1.19/1.20 ([leedeternal/dcrd#3052](https://github.com/leedeternal/dcrd/pull/3052))
- docs: Add release notes for v1.7.7 ([leedeternal/dcrd#3088](https://github.com/leedeternal/dcrd/pull/3088))
- docs: Update for rpc/jsonrpc/types/v4 module ([leedeternal/dcrd#3103](https://github.com/leedeternal/dcrd/pull/3103))
- docs: Update simnet env docs for subsidy split r2 ([leedeternal/dcrd#3092](https://github.com/leedeternal/dcrd/pull/3092))
- docs: Update for blockchain/stake v5 module ([leedeternal/dcrd#3131](https://github.com/leedeternal/dcrd/pull/3131))
- docs: Update for gcs v4 module ([leedeternal/dcrd#3132](https://github.com/leedeternal/dcrd/pull/3132))
- docs: Update for blockchain v5 module ([leedeternal/dcrd#3133](https://github.com/leedeternal/dcrd/pull/3133))
- docs: Update for rpcclient v8 module ([leedeternal/dcrd#3134](https://github.com/leedeternal/dcrd/pull/3134))

### Contrib changes:

- contrib: Use bash builtins instead of seq ([leedeternal/dcrd#2867](https://github.com/leedeternal/dcrd/pull/2867))
- docker: Update image to golang:1.18.0-alpine3.15 ([leedeternal/dcrd#2907](https://github.com/leedeternal/dcrd/pull/2907))
- contrib: Add Go multimod workspace setup script ([leedeternal/dcrd#2904](https://github.com/leedeternal/dcrd/pull/2904))
- docker: Update image to golang:1.18.3-alpine3.16 ([leedeternal/dcrd#2960](https://github.com/leedeternal/dcrd/pull/2960))
- docker: Update image to golang:1.19.0-alpine3.16 ([leedeternal/dcrd#2983](https://github.com/leedeternal/dcrd/pull/2983))
- docker: Update image to golang:1.19.1-alpine3.16 ([leedeternal/dcrd#2992](https://github.com/leedeternal/dcrd/pull/2992))
- docker: Update image to golang:1.19.5-alpine3.17 ([leedeternal/dcrd#3043](https://github.com/leedeternal/dcrd/pull/3043))
- contrib: Docker forward entrypoint signals ([leedeternal/dcrd#3044](https://github.com/leedeternal/dcrd/pull/3044))
- contrib: Finish Docker documentation ([leedeternal/dcrd#3045](https://github.com/leedeternal/dcrd/pull/3045))
- docker: Add ability to build versioned docker images ([leedeternal/dcrd#3048](https://github.com/leedeternal/dcrd/pull/3048))
- docker: Update image to golang:1.20.1-alpine3.17 ([leedeternal/dcrd#3063](https://github.com/leedeternal/dcrd/pull/3063))
- docker: Add dcrctl version output ([leedeternal/dcrd#3062](https://github.com/leedeternal/dcrd/pull/3062))

### Developer-related package and module changes:

- blockchain: Change tspend pass log level to debug ([leedeternal/dcrd#2862](https://github.com/leedeternal/dcrd/pull/2862))
- stdscript: Reject multisig neg thresholds ([leedeternal/dcrd#2859](https://github.com/leedeternal/dcrd/pull/2859))
- stdaddr: Limit v0 addr decode to max possible size ([leedeternal/dcrd#2860](https://github.com/leedeternal/dcrd/pull/2860))
- hdkeychain: Limit decode to max possible size ([leedeternal/dcrd#2860](https://github.com/leedeternal/dcrd/pull/2860))
- dcrutil: Limit WIF decode to max possible size ([leedeternal/dcrd#2860](https://github.com/leedeternal/dcrd/pull/2860))
- txscript: Support min int64 script num encoding ([leedeternal/dcrd#2863](https://github.com/leedeternal/dcrd/pull/2863))
- edwards: More strict pubkey parsing ([leedeternal/dcrd#2869](https://github.com/leedeternal/dcrd/pull/2869))
- server: sync peer handling sends/receives ([leedeternal/dcrd#2864](https://github.com/leedeternal/dcrd/pull/2864))
- indexers: Cleanup hash and string handling ([leedeternal/dcrd#2873](https://github.com/leedeternal/dcrd/pull/2873))
- ffldb: Add dbErr to error description ([leedeternal/dcrd#2876](https://github.com/leedeternal/dcrd/pull/2876))
- blockchain: Add ldbErr to error description ([leedeternal/dcrd#2876](https://github.com/leedeternal/dcrd/pull/2876))
- secp256k1: Support go generate w/o removing file ([leedeternal/dcrd#2885](https://github.com/leedeternal/dcrd/pull/2885))
- secp256k1: Optimize precomp value storage ([leedeternal/dcrd#2885](https://github.com/leedeternal/dcrd/pull/2885))
- chain: Add currentDeploymentVersion helper ([leedeternal/dcrd#2878](https://github.com/leedeternal/dcrd/pull/2878))
- chain: Add nextDeploymentVersion helper ([leedeternal/dcrd#2878](https://github.com/leedeternal/dcrd/pull/2878))
- chain: Add deployment version to db ([leedeternal/dcrd#2878](https://github.com/leedeternal/dcrd/pull/2878))
- chain: Track deployment version ([leedeternal/dcrd#2878](https://github.com/leedeternal/dcrd/pull/2878))
- chain: Add newDeploymentsStartTime helper ([leedeternal/dcrd#2878](https://github.com/leedeternal/dcrd/pull/2878))
- chain: Revalidate blocks for new deployments ([leedeternal/dcrd#2878](https://github.com/leedeternal/dcrd/pull/2878))
- secp256k1: Decouple precomputation generation ([leedeternal/dcrd#2886](https://github.com/leedeternal/dcrd/pull/2886))
- secp256k1: Rework k splitting tests ([leedeternal/dcrd#2888](https://github.com/leedeternal/dcrd/pull/2888))
- secp256k1: Generate all endomorphism params ([leedeternal/dcrd#2888](https://github.com/leedeternal/dcrd/pull/2888))
- secp256k1: Optimize k splitting with mod n scalar ([leedeternal/dcrd#2888](https://github.com/leedeternal/dcrd/pull/2888))
- txscript: Reduce checkmultisig allocs ([leedeternal/dcrd#2890](https://github.com/leedeternal/dcrd/pull/2890))
- secp256k1: Reduce scalar base mult copies ([leedeternal/dcrd#2898](https://github.com/leedeternal/dcrd/pull/2898))
- indexers: fix indexer wait for sync ([leedeternal/dcrd#2871](https://github.com/leedeternal/dcrd/pull/2871))
- secp256k1/ecdsa: Accept nonce in internal signing ([leedeternal/dcrd#2908](https://github.com/leedeternal/dcrd/pull/2908))
- secp256k1/ecdsa: Consistent sig recovery errors ([leedeternal/dcrd#2914](https://github.com/leedeternal/dcrd/pull/2914))
- chaincfg: Deprecate subsidy split params ([leedeternal/dcrd#2916](https://github.com/leedeternal/dcrd/pull/2916))
- internal/mining: createCoinbaseTx never returns an error ([leedeternal/dcrd#2917](https://github.com/leedeternal/dcrd/pull/2917))
- blockchain: Remove unused params ([leedeternal/dcrd#2918](https://github.com/leedeternal/dcrd/pull/2918))
- stake: Use a single copy instead of a for loop ([leedeternal/dcrd#2919](https://github.com/leedeternal/dcrd/pull/2919))
- primitives: Add subsidy calcs ([leedeternal/dcrd#2920](https://github.com/leedeternal/dcrd/pull/2920))
- primitives: Add subsidy calc benchmarks ([leedeternal/dcrd#2920](https://github.com/leedeternal/dcrd/pull/2920))
- rpcserver: cleanup queueHandler process ([leedeternal/dcrd#2929](https://github.com/leedeternal/dcrd/pull/2929))
- rpcclient: Remove missedtickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- jsonrpc/types: Remove deprecated missedtickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- blockchain: Remove unused MissedTickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- jsonrpc/types: Remove rebroadcastmissed ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- rpcclient: Remove existsmissedtickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- blockchain: Remove unused CheckMissedTickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- jsonrpc/types: Remove existsmissedtickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- rpcclient: Remove existsexpiredtickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- blockchain: Remove unused CheckExpiredTickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- jsonrpc/types: Remove existsexpiredtickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- rpcclient: Remove notifyspentandmissedtickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- blockchain: Remove unused NTSpentAndMissedTickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- jsonrpc/types: Remove notifyspentandmissedtickets ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- internal/mempool: remove unused isTreasuryEnabled param ([leedeternal/dcrd#2917](https://github.com/leedeternal/dcrd/pull/2917))
- multi: Remove auto revoc flag from CheckSSRtx ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- multi: Remove auto revoc flag from IsSSRtx ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- multi: Remove treasury flag from CheckSSGenVotes ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- multi: Remove treasury flag from CheckSSGen ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- multi: Remove treasury flag from IsSSGen ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- multi: Remove treasury flag from several funcs ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- stake: Tx ver as agenda proxy in DetermineTxType ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- multi: Remove agenda flags from DetermineTxType ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- multi: Remove agenda flags from several funcs ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- server: Remove unnecessary vote ntfn check ([leedeternal/dcrd#2942](https://github.com/leedeternal/dcrd/pull/2942))
- blockchain: Export block index flush ([leedeternal/dcrd#2943](https://github.com/leedeternal/dcrd/pull/2943))
- blockchain: Consolidate best header access ([leedeternal/dcrd#2943](https://github.com/leedeternal/dcrd/pull/2943))
- blockchain: Optimize stake node pruning ([leedeternal/dcrd#2943](https://github.com/leedeternal/dcrd/pull/2943))
- blockchain: Remove unused errors ([leedeternal/dcrd#2947](https://github.com/leedeternal/dcrd/pull/2947))
- multi: Consolidate header proof logic ([leedeternal/dcrd#2937](https://github.com/leedeternal/dcrd/pull/2937))
- blockchain: Fix revocation fee limit bug ([leedeternal/dcrd#2948](https://github.com/leedeternal/dcrd/pull/2948))
- rpc/jsonrpc/types: Add agenda status constants ([leedeternal/dcrd#2939](https://github.com/leedeternal/dcrd/pull/2939))
- rpcserver: Decouple agenda info status strings ([leedeternal/dcrd#2939](https://github.com/leedeternal/dcrd/pull/2939))
- blockchain: Avoid repeated blks in 2 weeks calc ([leedeternal/dcrd#2945](https://github.com/leedeternal/dcrd/pull/2945))
- chaincfg: Deprecate LatestCheckpointHeight ([leedeternal/dcrd#2945](https://github.com/leedeternal/dcrd/pull/2945))
- chaincfg: Deprecate Checkpoints and remove entries ([leedeternal/dcrd#2945](https://github.com/leedeternal/dcrd/pull/2945))
- rpcclient: Remove searchrawtransactions ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- rpc/jsonrpc/types: Remove searchrawtransactions ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- rpc/jsonrpc/types: Remove getinfo addrindex field ([leedeternal/dcrd#2930](https://github.com/leedeternal/dcrd/pull/2930))
- standalone: Add transaction sanity check ([leedeternal/dcrd#2949](https://github.com/leedeternal/dcrd/pull/2949))
- blockchain: Use standalone check tx sanity ([leedeternal/dcrd#2950](https://github.com/leedeternal/dcrd/pull/2950))
- main: Only use server peer accessors ([leedeternal/dcrd#2953](https://github.com/leedeternal/dcrd/pull/2953))
- blockchain: Remove unused difficulty error ([leedeternal/dcrd#2951](https://github.com/leedeternal/dcrd/pull/2951))
- blockchain: Rename error for no treasury payout ([leedeternal/dcrd#2951](https://github.com/leedeternal/dcrd/pull/2951))
- fullblocktests: Decouple from blockchain ([leedeternal/dcrd#2951](https://github.com/leedeternal/dcrd/pull/2951))
- blockchain: Add filter hash to hdr cmt data struct ([leedeternal/dcrd#2938](https://github.com/leedeternal/dcrd/pull/2938))
- blockchain: Avoid db for filters of unknown blocks ([leedeternal/dcrd#2938](https://github.com/leedeternal/dcrd/pull/2938))
- blockchain: Move package to internal ([leedeternal/dcrd#2952](https://github.com/leedeternal/dcrd/pull/2952))
- mempool: Remove agendas from removeOrphan ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from limitNumOrphans ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from addOrphan ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from maybeAddOrphan ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove removeOrphanDoubleSpends agendas ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from removeTransaction ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agenda from addTransaction ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agenda from pruneStakeTx ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from pruneExpiredTx ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from RemoveOrphan ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from RemoveOrphansByTag ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from RemoveTransaction ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- mempool: Remove agendas from RemoveDoubleSpends ([leedeternal/dcrd#2954](https://github.com/leedeternal/dcrd/pull/2954))
- blockchain: Use new uint256 for work sums ([leedeternal/dcrd#2957](https://github.com/leedeternal/dcrd/pull/2957))
- blockchain: Return uint256 from chain work method ([leedeternal/dcrd#2959](https://github.com/leedeternal/dcrd/pull/2959))
- multi: remove spend pruner ([leedeternal/dcrd#2961](https://github.com/leedeternal/dcrd/pull/2961))
- mempool: Remove unused insufficient priority error ([leedeternal/dcrd#2964](https://github.com/leedeternal/dcrd/pull/2964))
- mempool: Remove maybeAcceptTransaction rate limit ([leedeternal/dcrd#2964](https://github.com/leedeternal/dcrd/pull/2964))
- mempool: Remove MaybeAcceptTransaction rate limit ([leedeternal/dcrd#2964](https://github.com/leedeternal/dcrd/pull/2964))
- mempool: Remove ProcessTransaction rate limit ([leedeternal/dcrd#2964](https://github.com/leedeternal/dcrd/pull/2964))
- blockchain: Remove leftover treasury debug logging ([leedeternal/dcrd#2966](https://github.com/leedeternal/dcrd/pull/2966))
- rpcserver: Avoid error in handleRebroadcastWinners ([leedeternal/dcrd#2968](https://github.com/leedeternal/dcrd/pull/2968))
- secp256k1: Expose IsZeroBit on mod n scalar type ([leedeternal/dcrd#2971](https://github.com/leedeternal/dcrd/pull/2971))
- secp256k1: Implement direct key generation ([leedeternal/dcrd#2971](https://github.com/leedeternal/dcrd/pull/2971))
- secp256k1: Store constant on stack instead of heap ([leedeternal/dcrd#2972](https://github.com/leedeternal/dcrd/pull/2972))
- blockchain: Consistency pass for next req dif calc ([leedeternal/dcrd#2978](https://github.com/leedeternal/dcrd/pull/2978))
- mining: Copy regular txns for alternate templates ([leedeternal/dcrd#2978](https://github.com/leedeternal/dcrd/pull/2978))
- mempool: Delete unreachable code in tests ([leedeternal/dcrd#2987](https://github.com/leedeternal/dcrd/pull/2987))
- indexers: Remove unused PrevScripts from idx ntfn ([leedeternal/dcrd#2989](https://github.com/leedeternal/dcrd/pull/2989))
- indexers: Remove unused indexNeedsInputs ([leedeternal/dcrd#2989](https://github.com/leedeternal/dcrd/pull/2989))
- indexers: Remove unused NeedsInputser ([leedeternal/dcrd#2989](https://github.com/leedeternal/dcrd/pull/2989))
- blockchain: Remove unused PrevScripts from ntfns ([leedeternal/dcrd#2989](https://github.com/leedeternal/dcrd/pull/2989))
- blockchain: Remove unused prev scripts snapshots ([leedeternal/dcrd#2989](https://github.com/leedeternal/dcrd/pull/2989))
- indexers: Remove unused ChainQueryer.PrevScripts ([leedeternal/dcrd#2989](https://github.com/leedeternal/dcrd/pull/2989))
- blockchain: Remove unused stxosToScriptSource ([leedeternal/dcrd#2989](https://github.com/leedeternal/dcrd/pull/2989))
- indexers: Remove unused PrevScripter interface ([leedeternal/dcrd#2989](https://github.com/leedeternal/dcrd/pull/2989))
- dcrutil: Correct off-by-1 index check ([leedeternal/dcrd#2991](https://github.com/leedeternal/dcrd/pull/2991))
- blockchain: Avoid unneeded view script deep copies ([leedeternal/dcrd#2993](https://github.com/leedeternal/dcrd/pull/2993))
- mining: Don't fetch inputs for coinbase ([leedeternal/dcrd#2994](https://github.com/leedeternal/dcrd/pull/2994))
- blockchain: Remove legacy spend pruner bucket ([leedeternal/dcrd#2998](https://github.com/leedeternal/dcrd/pull/2998))
- blockchain: Remove unused internal spendpruner pkg ([leedeternal/dcrd#2998](https://github.com/leedeternal/dcrd/pull/2998))
- blockchain: Remove unused disable verify method ([leedeternal/dcrd#2999](https://github.com/leedeternal/dcrd/pull/2999))
- blockchain: Remove disconnected spend journal ([leedeternal/dcrd#2997](https://github.com/leedeternal/dcrd/pull/2997))
- blockchain: PrevScripter interface for script val ([leedeternal/dcrd#3000](https://github.com/leedeternal/dcrd/pull/3000))
- blockchain: Misc consistency cleanup pass ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Pre-allocate in-flight utxoview tx map ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Remove exported utxo cache SpendEntry ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Remove exported utxo cache AddEntry ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Remove unused utxo cache add entry err ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Remove tip param from utxo cache init ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Fix rare unclean utxo cache recovery ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Don't fetch trsy{base,spend} inputs ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Separate utxo cache vs view state ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Improve utxo cache spend robustness ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Split regular/stake view tx connect ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Bypass utxo cache for zero conf spends ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Update utxodb info during upgrade ([leedeternal/dcrd#2996](https://github.com/leedeternal/dcrd/pull/2996))
- rpc/jsonrpc/types: Deprecate mempool prio fields ([leedeternal/dcrd#3003](https://github.com/leedeternal/dcrd/pull/3003))
- txscript: remove obsolete requireMinimal comment ([leedeternal/dcrd#3004](https://github.com/leedeternal/dcrd/pull/3004))
- blockchain: Remove incorrect upgrade version check ([leedeternal/dcrd#3010](https://github.com/leedeternal/dcrd/pull/3010))
- server: Use the server context when connecting to peers ([leedeternal/dcrd#3011](https://github.com/leedeternal/dcrd/pull/3011))
- server: Make sure the peer address exists in the manager ([leedeternal/dcrd#3013](https://github.com/leedeternal/dcrd/pull/3013))
- mempool: Store transaction descs in pools ([leedeternal/dcrd#3015](https://github.com/leedeternal/dcrd/pull/3015))
- rpcserver: Use atomic for ws client disconnect ([leedeternal/dcrd#3024](https://github.com/leedeternal/dcrd/pull/3024))
- rpcserver: Cast JSON-RPC req method once in parse ([leedeternal/dcrd#3025](https://github.com/leedeternal/dcrd/pull/3025))
- rpcserver: Consistent block connected ntfn skip ([leedeternal/dcrd#3025](https://github.com/leedeternal/dcrd/pull/3025))
- rpcserver: Convert ws client lifecycle to context ([leedeternal/dcrd#3025](https://github.com/leedeternal/dcrd/pull/3025))
- rpcserver: Pass request context to handlers ([leedeternal/dcrd#3026](https://github.com/leedeternal/dcrd/pull/3026))
- rpcserver: Detect disconnect of ws clients ([leedeternal/dcrd#3031](https://github.com/leedeternal/dcrd/pull/3031))
- mempool: Accept expired prune height ([leedeternal/dcrd#3042](https://github.com/leedeternal/dcrd/pull/3042))
- addrmgr: break after selecting random address ([leedeternal/dcrd#3047](https://github.com/leedeternal/dcrd/pull/3047))
- addrmgr: set min value and optimize address chance ([leedeternal/dcrd#3047](https://github.com/leedeternal/dcrd/pull/3047))
- multi: Use atomic types in unexported modules ([leedeternal/dcrd#3053](https://github.com/leedeternal/dcrd/pull/3053))
- connmgr: Access conn req id via accessor ([leedeternal/dcrd#3055](https://github.com/leedeternal/dcrd/pull/3055))
- connmgr: Synchronize existing conn req ID assign ([leedeternal/dcrd#3055](https://github.com/leedeternal/dcrd/pull/3055))
- chaincfg: Rework init time deployment val logic ([leedeternal/dcrd#3056](https://github.com/leedeternal/dcrd/pull/3056))
- blockchain: Simplify threshold state determination ([leedeternal/dcrd#3059](https://github.com/leedeternal/dcrd/pull/3059))
- multi: Remove unused last state depl ver param ([leedeternal/dcrd#3059](https://github.com/leedeternal/dcrd/pull/3059))
- multi: Remove unused next state depl ver param ([leedeternal/dcrd#3059](https://github.com/leedeternal/dcrd/pull/3059))
- blockchain: Remove unused lookup depl ver method ([leedeternal/dcrd#3059](https://github.com/leedeternal/dcrd/pull/3059))
- blockchain: Remove unused db update tracking ([leedeternal/dcrd#3065](https://github.com/leedeternal/dcrd/pull/3065))
- blockchain: Validate deployment chain params ([leedeternal/dcrd#3068](https://github.com/leedeternal/dcrd/pull/3068))
- blockchain: Remove deployment checker abstraction ([leedeternal/dcrd#3069](https://github.com/leedeternal/dcrd/pull/3069))
- blockchain: Use iota for threshold states ([leedeternal/dcrd#3072](https://github.com/leedeternal/dcrd/pull/3072))
- blockchain: Reject params with mask approval bit ([leedeternal/dcrd#3073](https://github.com/leedeternal/dcrd/pull/3073))
- blockchain: Reject params with shared mask bits ([leedeternal/dcrd#3077](https://github.com/leedeternal/dcrd/pull/3077))
- blockchain: Reject params with blank choice id ([leedeternal/dcrd#3079](https://github.com/leedeternal/dcrd/pull/3079))
- blockchain: Make zero val threshold tuple invalid ([leedeternal/dcrd#3080](https://github.com/leedeternal/dcrd/pull/3080))
- secp256k1: Add GeneratePrivateKeyFromRand function ([leedeternal/dcrd#3096](https://github.com/leedeternal/dcrd/pull/3096))
- secp256k1: Require concerete rand for privkey gen ([leedeternal/dcrd#3097](https://github.com/leedeternal/dcrd/pull/3097))
- secp256k1: Update PrivKeyFromBytes comment ([leedeternal/dcrd#3098](https://github.com/leedeternal/dcrd/pull/3098))
- dcrutil: Remove unused block assertion ([leedeternal/dcrd#3106](https://github.com/leedeternal/dcrd/pull/3106))
- peer: Minor summary debug log cleanup ([leedeternal/dcrd#3108](https://github.com/leedeternal/dcrd/pull/3108))
- standalone: Add modified subsidy split calcs ([leedeternal/dcrd#3092](https://github.com/leedeternal/dcrd/pull/3092))
- blockchain: Remove unused next threshold state err ([leedeternal/dcrd#3110](https://github.com/leedeternal/dcrd/pull/3110))
- blockchain: Remove unused last changed state err ([leedeternal/dcrd#3110](https://github.com/leedeternal/dcrd/pull/3110))
- blockchain: Remove unused deployment state err ([leedeternal/dcrd#3110](https://github.com/leedeternal/dcrd/pull/3110))
- blockchain: Remove unused max block size err ([leedeternal/dcrd#3110](https://github.com/leedeternal/dcrd/pull/3110))
- blockchain: Remove unused stake diff v1 err ([leedeternal/dcrd#3110](https://github.com/leedeternal/dcrd/pull/3110))
- blockchain: Remove unused calc next stake diff err ([leedeternal/dcrd#3110](https://github.com/leedeternal/dcrd/pull/3110))
- blockchain: Remove deprecated subscribers method ([leedeternal/dcrd#3113](https://github.com/leedeternal/dcrd/pull/3113))
- multi: Remove unused tip generation error ([leedeternal/dcrd#3112](https://github.com/leedeternal/dcrd/pull/3112))
- blockchain: Correct total subsidy database entry ([leedeternal/dcrd#3112](https://github.com/leedeternal/dcrd/pull/3112))
- wire: Add PowHashV2 using blake3 ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- rpcserver: Consolidate work data serialization ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- standalone: Add separate proof of work hash check ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- chaincfg: Use consts for pow limit bits ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- standalone: Add ASERT difficulty calculation ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))

### Developer-related module management:

- multi: Update module requirements to go1.13 ([leedeternal/dcrd#2891](https://github.com/leedeternal/dcrd/pull/2891))
- main: Update to use new module versions ([leedeternal/dcrd#2892](https://github.com/leedeternal/dcrd/pull/2892))
- blockchain: Start v5 module dev cycle ([leedeternal/dcrd#2903](https://github.com/leedeternal/dcrd/pull/2903))
- multi: Support module graph prune and lazy load ([leedeternal/dcrd#2905](https://github.com/leedeternal/dcrd/pull/2905))
- rpc/jsonrpc/types: Start v4 module dev cycle ([leedeternal/dcrd#2910](https://github.com/leedeternal/dcrd/pull/2910))
- rpcclient: Start v8 module dev cycle ([leedeternal/dcrd#2909](https://github.com/leedeternal/dcrd/pull/2909))
- rpcserver: Bump version to 8.0.0 ([leedeternal/dcrd#2911](https://github.com/leedeternal/dcrd/pull/2911))
- gcs: Start v4 module dev cycle ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- blockchain/stake: Start v5 module dev cycle ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- fullblocktests: Update readme module path ([leedeternal/dcrd#2951](https://github.com/leedeternal/dcrd/pull/2951))
- main: Update to use latest sys module ([leedeternal/dcrd#3049](https://github.com/leedeternal/dcrd/pull/3049))
- main: Update to use latest term module ([leedeternal/dcrd#3050](https://github.com/leedeternal/dcrd/pull/3050))
- apbf: Prepare v1.0.1 ([leedeternal/dcrd#3061](https://github.com/leedeternal/dcrd/pull/3061))
- chaincfg/chainhash: Prepare v1.0.4 ([leedeternal/dcrd#3094](https://github.com/leedeternal/dcrd/pull/3094))
- secp256k1: Prepare v4.2.0 ([leedeternal/dcrd#3101](https://github.com/leedeternal/dcrd/pull/3101))
- dcrjson: Prepare v4.0.1 ([leedeternal/dcrd#3102](https://github.com/leedeternal/dcrd/pull/3102))
- rpc/jsonrpc/types: Prepare v4.0.0 ([leedeternal/dcrd#3103](https://github.com/leedeternal/dcrd/pull/3103))
- wire: Prepare v1.6.0 ([leedeternal/dcrd#3119](https://github.com/leedeternal/dcrd/pull/3119))
- blockchain/standalone: Prepare v2.2.0 ([leedeternal/dcrd#3120](https://github.com/leedeternal/dcrd/pull/3120))
- addrmgr: Prepare v2.0.2 ([leedeternal/dcrd#3121](https://github.com/leedeternal/dcrd/pull/3121))
- chaincfg: Prepare v3.2.0 ([leedeternal/dcrd#3125](https://github.com/leedeternal/dcrd/pull/3125))
- connmgr: Prepare v3.1.1 ([leedeternal/dcrd#3124](https://github.com/leedeternal/dcrd/pull/3124))
- txscript: Prepare v4.1.0 ([leedeternal/dcrd#3126](https://github.com/leedeternal/dcrd/pull/3126))
- hdkeychain: Prepare v3.1.1 ([leedeternal/dcrd#3127](https://github.com/leedeternal/dcrd/pull/3127))
- peer: Prepare v3.0.2 ([leedeternal/dcrd#3128](https://github.com/leedeternal/dcrd/pull/3128))
- dcrutil: Prepare v4.0.1 ([leedeternal/dcrd#3129](https://github.com/leedeternal/dcrd/pull/3129))
- database: Prepare v3.0.1 ([leedeternal/dcrd#3130](https://github.com/leedeternal/dcrd/pull/3130))
- blockchain/stake: Prepare v5.0.0 ([leedeternal/dcrd#3131](https://github.com/leedeternal/dcrd/pull/3131))
- gcs: Prepare v4.0.0 ([leedeternal/dcrd#3132](https://github.com/leedeternal/dcrd/pull/3132))
- blockchain: Prepare v5.0.0 ([leedeternal/dcrd#3133](https://github.com/leedeternal/dcrd/pull/3133))
- rpcclient: Prepare v8.0.0 ([leedeternal/dcrd#3134](https://github.com/leedeternal/dcrd/pull/3134))
- main: Update to use all new module versions ([leedeternal/dcrd#3138](https://github.com/leedeternal/dcrd/pull/3138))
- main: Remove module replacements ([leedeternal/dcrd#3139](https://github.com/leedeternal/dcrd/pull/3139))

### Testing and Quality Assurance:

- dcrutil: Rework WIF tests ([leedeternal/dcrd#2860](https://github.com/leedeternal/dcrd/pull/2860))
- build: update golangci-lint to v1.44.1 ([leedeternal/dcrd#2882](https://github.com/leedeternal/dcrd/pull/2882))
- build: Set token permissions for go.yml ([leedeternal/dcrd#2896](https://github.com/leedeternal/dcrd/pull/2896))
- secp256k1: Benchmark consistency pass ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Consolidate Jacobian group chk in tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Consolidate affine group check in tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup and move affine addition tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup and move affine double tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup and move key generation tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup and move affine scalar mul tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup affine scalar base mult tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup and move base mult rand tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup private key tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup Jacobian addition tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Cleanup Jacobian double tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Add test gen for random mod n scalars ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Add Jacobian scalar base mult tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- secp256k1: Rework Jacobian rand scalar mult tests ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- build: Replace deprecated terminal dep ([leedeternal/dcrd#2894](https://github.com/leedeternal/dcrd/pull/2894))
- rpctest: Pass and use context versus channel ([leedeternal/dcrd#2895](https://github.com/leedeternal/dcrd/pull/2895))
- secp256k1: Add benchmark for splitK ([leedeternal/dcrd#2888](https://github.com/leedeternal/dcrd/pull/2888))
- build: Use recommended golangci-lint installer ([leedeternal/dcrd#2899](https://github.com/leedeternal/dcrd/pull/2899))
- build: Update to latest action versions ([leedeternal/dcrd#2900](https://github.com/leedeternal/dcrd/pull/2900))
- blockchain: Use TempDir to create temp test dirs ([leedeternal/dcrd#2902](https://github.com/leedeternal/dcrd/pull/2902))
- secp256k1/ecdsa: Correct test comment ([leedeternal/dcrd#2908](https://github.com/leedeternal/dcrd/pull/2908))
- secp256k1/ecdsa: Add sign and verify tests ([leedeternal/dcrd#2908](https://github.com/leedeternal/dcrd/pull/2908))
- secp256k1/ecdsa: Add rand sign and verify tests ([leedeternal/dcrd#2908](https://github.com/leedeternal/dcrd/pull/2908))
- secp256k1/ecdsa: Add compact signature tests ([leedeternal/dcrd#2915](https://github.com/leedeternal/dcrd/pull/2915))
- secp256k1/ecdsa: Rework rand compact sig tests ([leedeternal/dcrd#2915](https://github.com/leedeternal/dcrd/pull/2915))
- ecdsa: Fix test that randomly picks a component ([leedeternal/dcrd#2919](https://github.com/leedeternal/dcrd/pull/2919))
- chaingen: Update for deprecated subsidy params ([leedeternal/dcrd#2928](https://github.com/leedeternal/dcrd/pull/2928))
- stake: Set test vote transactions as version 1 ([leedeternal/dcrd#2922](https://github.com/leedeternal/dcrd/pull/2922))
- mempool: Use valid tx fees in test harness ([leedeternal/dcrd#2962](https://github.com/leedeternal/dcrd/pull/2962))
- secp256k1: Add benchmark for private key gen ([leedeternal/dcrd#2971](https://github.com/leedeternal/dcrd/pull/2971))
- build: Update to latest action versions ([leedeternal/dcrd#2981](https://github.com/leedeternal/dcrd/pull/2981))
- build: Update golangci-lint to v1.48.0 ([leedeternal/dcrd#2981](https://github.com/leedeternal/dcrd/pull/2981))
- build: Test against Go 1.19 ([leedeternal/dcrd#2981](https://github.com/leedeternal/dcrd/pull/2981))
- blockchain: Make longer running tests parallel ([leedeternal/dcrd#2988](https://github.com/leedeternal/dcrd/pull/2988))
- blockchain: Allow tests to override cache flushing ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Improve utxo cache initialize tests ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Consolidate utxo cache test entries ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Rework utxo cache spend entry tests ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Rework utxo cache commit tests ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- blockchain: Rework utxo cache add entry tests ([leedeternal/dcrd#2995](https://github.com/leedeternal/dcrd/pull/2995))
- build: Update to golangci-lint v1.50.0 ([leedeternal/dcrd#3011](https://github.com/leedeternal/dcrd/pull/3011))
- rpctest:  Use context provided by the user ([leedeternal/dcrd#3012](https://github.com/leedeternal/dcrd/pull/3012))
- build: Enable run_tests.sh to work with go.work ([leedeternal/dcrd#3021](https://github.com/leedeternal/dcrd/pull/3021))
- build: Only invoke tests once ([leedeternal/dcrd#3023](https://github.com/leedeternal/dcrd/pull/3023))
- build: Rename root pkg path vars ([leedeternal/dcrd#3022](https://github.com/leedeternal/dcrd/pull/3022))
- multi: Move rpctest-based tests to own package ([leedeternal/dcrd#3028](https://github.com/leedeternal/dcrd/pull/3028))
- rpctests: Switch to dcrtest/dcrdtest package ([leedeternal/dcrd#3028](https://github.com/leedeternal/dcrd/pull/3028))
- rpctest: Remove package ([leedeternal/dcrd#3028](https://github.com/leedeternal/dcrd/pull/3028))
- rpcserver: Add limited methods exist test ([leedeternal/dcrd#3033](https://github.com/leedeternal/dcrd/pull/3033))
- rpctests: Build constraint for util too ([leedeternal/dcrd#3037](https://github.com/leedeternal/dcrd/pull/3037))
- hdkeychain: Use errors for test compare ([leedeternal/dcrd#3038](https://github.com/leedeternal/dcrd/pull/3038))
- build: Update to latest action versions ([leedeternal/dcrd#3052](https://github.com/leedeternal/dcrd/pull/3052))
- build: Update golangci-lint to v1.51.1 ([leedeternal/dcrd#3052](https://github.com/leedeternal/dcrd/pull/3052))
- build: Test against Go 1.20 ([leedeternal/dcrd#3052](https://github.com/leedeternal/dcrd/pull/3052))
- chaincfg: Rework deployment validation tests ([leedeternal/dcrd#3056](https://github.com/leedeternal/dcrd/pull/3056))
- blockchain: Rework vote tests ([leedeternal/dcrd#3075](https://github.com/leedeternal/dcrd/pull/3075))
- blockchain: Convert several test helpers ([leedeternal/dcrd#3076](https://github.com/leedeternal/dcrd/pull/3076))
- blockchain: Use local mock votes in tests ([leedeternal/dcrd#3076](https://github.com/leedeternal/dcrd/pull/3076))
- blockchain: Reassign non-overlapping test params ([leedeternal/dcrd#3077](https://github.com/leedeternal/dcrd/pull/3077))
- build: move golangci flags to its own config file ([leedeternal/dcrd#3081](https://github.com/leedeternal/dcrd/pull/3081))
- secp256k1: Add GeneratePrivateKeyFromRand tests ([leedeternal/dcrd#3100](https://github.com/leedeternal/dcrd/pull/3100))
- rpcserver: Use solved mock orphan block in tests ([leedeternal/dcrd#3104](https://github.com/leedeternal/dcrd/pull/3104))
- rpcserver: Consolidate mock mining addr ([leedeternal/dcrd#3105](https://github.com/leedeternal/dcrd/pull/3105))
- blockchain: Agenda test consistency pass ([leedeternal/dcrd#3107](https://github.com/leedeternal/dcrd/pull/3107))
- build: golangci-lint v1.53.1 ([leedeternal/dcrd#3117](https://github.com/leedeternal/dcrd/pull/3117))
- chaingen: Move mining solve to generate state ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- chaingen: Add blake3 support ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- chaingen: Add ASERT difficulty algorithm support ([leedeternal/dcrd#3115](https://github.com/leedeternal/dcrd/pull/3115))
- rpctest: Enable treasury integration test ([leedeternal/dcrd#3118](https://github.com/leedeternal/dcrd/pull/3118))

### Misc:

- release: Bump for 1.8 release cycle ([leedeternal/dcrd#2854](https://github.com/leedeternal/dcrd/pull/2854))
- secp256k1: Correct several comments ([leedeternal/dcrd#2887](https://github.com/leedeternal/dcrd/pull/2887))
- main: Update .gitignore for Go1.18 ([leedeternal/dcrd#2893](https://github.com/leedeternal/dcrd/pull/2893))
- multi: Update Go versions in README.md and .github/workflows/go.yml ([leedeternal/dcrd#2906](https://github.com/leedeternal/dcrd/pull/2906))
- multi: Fix a few typos ([leedeternal/dcrd#2923](https://github.com/leedeternal/dcrd/pull/2923))
- blockchain: Address some linter complaints ([leedeternal/dcrd#2927](https://github.com/leedeternal/dcrd/pull/2927))
- dcrjson: Address some linter complaints ([leedeternal/dcrd#2927](https://github.com/leedeternal/dcrd/pull/2927))
- connmgr: Address some linter complaints ([leedeternal/dcrd#2927](https://github.com/leedeternal/dcrd/pull/2927))
- blockchain/stake: Address some linter complaints ([leedeternal/dcrd#2927](https://github.com/leedeternal/dcrd/pull/2927))
- multi: Ensure newline at end of file ([leedeternal/dcrd#2941](https://github.com/leedeternal/dcrd/pull/2941))
- blockchain: Correct a few error comment typos ([leedeternal/dcrd#2951](https://github.com/leedeternal/dcrd/pull/2951))
- blockchain: Correct some db cfilterv2 comments ([leedeternal/dcrd#2938](https://github.com/leedeternal/dcrd/pull/2938))
- blockchain: Address some linter complaints ([leedeternal/dcrd#2965](https://github.com/leedeternal/dcrd/pull/2965))
- rpcserver: Address unused param linter complaints ([leedeternal/dcrd#2970](https://github.com/leedeternal/dcrd/pull/2970))
- multi: Go 1.19 doc comment formatting ([leedeternal/dcrd#2976](https://github.com/leedeternal/dcrd/pull/2976))
- schnorr: Go 1.19 doc comment formatting ([leedeternal/dcrd#2981](https://github.com/leedeternal/dcrd/pull/2981))
- main: Remove old style build constraints ([leedeternal/dcrd#3036](https://github.com/leedeternal/dcrd/pull/3036))
- secp256k1: Fix typo in a doc comment ([leedeternal/dcrd#3091](https://github.com/leedeternal/dcrd/pull/3091))
- release: Bump for 1.8.0 ([leedeternal/dcrd#3140](https://github.com/leedeternal/dcrd/pull/3140))

### Code Contributors (alphabetical order):

- Abirdcfly
- Dave Collins
- David Hill
- Donald Adu-Poku
- Eng Zer Jun
- Jamie Holdstock
- JoeGruff
- Jonathan Chappelow
- Josh Rickmar
- Julian Y
- Matheus Degiovani
- Ryan Staudt
- Sef Boukenken
- arjundashrath
- matthawkins90
- norwnd
- peterzen
- 刘昆
