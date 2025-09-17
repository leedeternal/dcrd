# dcrd v1.5.0

This release of dcrd introduces a large number of updates.  Some of the key highlights are:

* A new consensus vote agenda which allows the stakeholders to decide whether or not to activate support for block header commitments
* More efficient block filters
* Significant improvements to the mining infrastructure including asynchronous work notifications
* Major performance enhancements for transaction script validation
* Automatic external IP address discovery
* Support for IPv6 over Tor
* Various updates to the RPC server such as:
  * A new method to query information about the network
  * A method to retrieve the new version 2 block filters
  * More calls available to limited access users
* Infrastructure improvements
* Quality assurance changes

For those unfamiliar with the voting process in Decred, all code in order to support block header commitments is already included in this release, however its enforcement will remain dormant until the stakeholders vote to activate it.

For reference, block header commitments were originally proposed and approved for initial implementation via the following Politeia proposal:
- [Block Header Commitments Consensus Change](https://proposals.decred.org/proposals/0a1ff846ec271184ea4e3a921a3ccd8d478f69948b984445ee1852f272d54c58)


The following Decred Change Proposal (DCP) describes the proposed changes in detail and provides a full technical specification:
- [DCP0005](https://github.com/leedeternal/dcps/blob/master/dcp-0005/dcp-0005.mediawiki)

**It is important for everyone to upgrade their software to this latest release even if you don't intend to vote in favor of the agenda.**

## Downgrade Warning

The database format in v1.5.0 is not compatible with previous versions of the software.  This only affects downgrades as users upgrading from previous versions will see a one time database migration.

Once this migration has been completed, it will no longer be possible to downgrade to a previous version of the software without having to delete the database and redownload the chain.

## Notable Changes

### Block Header Commitments Vote

A new vote with the id `headercommitments` is now available as of this release.  After upgrading, stakeholders may set their preferences through their wallet or Voting Service Provider's (VSP) website.

The primary goal of this change is to increase the security and efficiency of lightweight clients, such as Decrediton in its lightweight mode and the dcrandroid/dcrios mobile wallets, as well as add infrastructure that paves the
way for several future scalability enhancements.

A high level overview aimed at a general audience including a cost benefit analysis can be found in the  [Politeia proposal](https://proposals.decred.org/proposals/0a1ff846ec271184ea4e3a921a3ccd8d478f69948b984445ee1852f272d54c58).

In addition, a much more in-depth treatment can be found in the [motivation section of DCP0005](https://github.com/leedeternal/dcps/blob/master/dcp-0005/dcp-0005.mediawiki#motivation).

### Version 2 Block Filters

The block filters used by lightweight clients, such as SPV (Simplified Payment Verification) wallets, have been updated to improve their efficiency, ergonomics, and include additional information such as the full ticket
commitment script.  The new block filters are version 2.  The older version 1 filters are now deprecated and scheduled to be removed in the next release, so consumers should update to the new filters as soon as possible.

An overview of block filters can be found in the [block filters section of DCP0005](https://github.com/leedeternal/dcps/blob/master/dcp-0005/dcp-0005.mediawiki#block-filters).

Also, the specific contents and technical specification of the new version 2 block filters is available in the
[version 2 block filters section of DCP0005](https://github.com/leedeternal/dcps/blob/master/dcp-0005/dcp-0005.mediawiki#version-2-block-filters).

Finally, there is a one time database update to build and store the new filters for all existing historical blocks which will likely take a while to complete (typically around 8 to 10 minutes on HDDs and 4 to 5 minutes on SSDs).

### Mining Infrastructure Overhaul

The mining infrastructure for building block templates and delivering the work to miners has been significantly overhauled to improve several aspects as follows:

* Support asynchronous background template generation with intelligent vote propagation handling
* Improved handling of chain reorganizations necessary when the current tip is unable to obtain enough votes
* Current state synchronization
* Near elimination of stale templates when new blocks and votes are received
* Subscriptions for streaming template updates

The standard [getwork RPC](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getwork) that PoW miners currently use to perform the mining process has been updated to make use of this new infrastructure, so existing PoW miners will seamlessly get the vast majority of benefits without requiring any updates.

However, in addition, a new [notifywork RPC](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#notifywork) is now available that allows miners to register for work to be delivered
asynchronously as it becomes available via a WebSockets [work notification](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#work).  These notifications include the same information that `getwork` provides along with an additional `reason` parameter which allows the miners to make better decisions about when they should instruct workers to discard the current template immediately or should be allowed to finish their current round before being provided with the new template.

Miners are highly encouraged to update their software to make use of the new asynchronous notification infrastructure since it is more robust, efficient, and faster than polling `getwork` to manually determine the aforementioned conditions.

The following is a non-exhaustive overview that highlights the major benefits of the changes for both cases:

- Requests for updated templates during the normal mining process in between tip   changes will now be nearly instant instead of potentially taking several seconds to build the new template on the spot
- When the chain tip changes, requesting a template will now attempt to wait until either all votes have been received or a timeout occurs prior to handing out a template which is beneficial for PoW miners, PoS miners, and the network as a whole
- PoW miners are much less likely to end up with template with less than the max number of votes which means they are less likely to receive a reduced subsidy
- PoW miners will be much less likely to receive stale templates during chain tip changes due to vote propagation
- PoS voters whose votes end up arriving to the miner slightly slower than the minimum number required are much less likely to have their votes excluded despite having voted simply due to propagation delay

PoW miners who choose to update their software, pool or otherwise, to make use of the asynchronous work notifications will receive additional benefits such as:

- Ability to start mining a new block sooner due to receiving updated work as soon as it becomes available
- Immediate notification with new work that includes any votes that arrive late
- Periodic notifications with new work that include new transactions only when there have actually been new transaction
- Simplified interface code due to removal of the need for polling and manually checking the work bytes for special cases such as the number of votes

**NOTE: Miners that are not rolling the timestamp field as they mine should ensure their software is upgraded to roll the timestamp to the latest timestamp each time they hand work out to a miner.  This helps ensure the block timestamps are as accurate as possible.**

### Transaction Script Validation Optimizations

Transaction script validation has been almost completely rewritten to significantly improve its speed and reduce the number of memory allocations. While this has many more benefits than enumerated here, probably the most
important ones for most stakeholders are:

- Votes can be cast more quickly which helps reduce the number of missed votes
- Blocks are able to propagate more quickly throughout the network, which in turn further improves votes times
- The initial sync process is around 20-25% faster

### Automatic External IP Address Discovery

In order for nodes to fully participate in the peer-to-peer network, they must be publicly accessible and made discoverable by advertising their external IP address.  This is typically made slightly more complicated since most users run their nodes on networks behind Network Address Translation (NAT).

Previously, in addition to configuring the network firewall and/or router to allow inbound connections to port 9108 and forwarding the port to the internal IP address running dcrd, it was also required to manually set the public external IP address via the `--externalip` CLI option.

This release will now make use of other nodes on the network in a decentralized fashion to automatically discover the external IP address, so it is no longer necessary to manually set CLI option for the vast majority of users.

### Tor IPv6 Support

It is now possible to resolve and connect to IPv6 peers over Tor in addition to the existing IPv4 support.

### RPC Server Changes

#### New Version 2 Block Filter Query RPC (`getcfilterv2`)

A new RPC named `getcfilterv2` is now available which can be used to retrieve the version 2 [block filter](https://github.com/leedeternal/dcps/blob/master/dcp-0005/dcp-0005.mediawiki#Block_Filters)
for a given block along with its associated inclusion proof.  See the [getcfilterv2 JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getcfilterv2)
for API details.

#### New Network Information Query RPC (`getnetworkinfo`)

A new RPC named `getnetworkinfo` is now available which can be used to query information related to the peer-to-peer network such as the protocol version, the local time offset, the number of current connections, the supported network protocols, the current transaction relay fee, and the external IP addresses for
the local interfaces.  See the [getnetworkinfo JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getnetworkinfo) for API details.

#### Updates to Chain State Query RPC (`getblockchaininfo`)

The `difficulty` field of the `getblockchaininfo` RPC is now deprecated in favor of a new field named `difficultyratio` which matches the result returned by the `getdifficulty` RPC.

See the [getblockchaininfo JSON-RPC API Documentation](https://github.com/leedeternal/dcrd/blob/master/docs/json_rpc_api.mediawiki#getblockchaininfo) for API details.

#### New Optional Version Parameter on Script Decode RPC (`decodescript`)

The `decodescript` RPC now accepts an additional optional parameter to specify the script version.  The only currently supported script version in Decred is version 0 which means decoding scripts with versions other than 0 will be seen as non standard.

#### Removal of Deprecated Block Template RPC (`getblocktemplate`)

The previously deprecated `getblocktemplate` RPC is no longer available.  All known miners are already using the preferred `getwork` RPC since Decred's block header supports more than enough nonce space to keep mining hardware busy without needing to resort to building custom templates with less efficient extra nonce coinbase workarounds.

#### Additional RPCs Available To Limited Access Users

The following RPCs that were previously unavailable to the limited access RPC user are now available to it:

- `estimatefee`
- `estimatesmartfee`
- `estimatestakediff`
- `existsaddress`
- `existsaddresses`
- `existsexpiredtickets`
- `existsliveticket`
- `existslivetickets`
- `existsmempoltxs`
- `existsmissedtickets`
- `getblocksubsidy`
- `getcfilter`
- `getcoinsupply`
- `getheaders`
- `getstakedifficulty`
- `getstakeversioninfo`
- `getstakeversions`
- `getvoteinfo`
- `livetickets`
- `missedtickets`
- `rebroadcastmissed`
- `rebroadcastwinners`
- `ticketfeeinfo`
- `ticketsforaddress`
- `ticketvwap`
- `txfeeinfo`

### Single Mining State Request

The peer-to-peer protocol message to request the current mining state (`getminings`) is used when peers first connect to retrieve all known votes for the current tip block.  This is only useful when the peer first connects because all future votes will be relayed once the connection has been established.  Consequently, nodes will now only respond to a single mining state request.  Subsequent requests are ignored.

### Developer Go Modules

A full suite of versioned Go modules (essentially code libraries) are now available for use by applications written in Go that wish to create robust software with reproducible, verifiable, and verified builds.

These modules are used to build dcrd itself and are therefore well maintained, tested, documented, and relatively efficient.

## Changelog

This release consists of 600 commits from 17 contributors which total to 537 files changed, 41494 additional lines of code, and 29215 deleted lines of code.

All commits since the last release may be viewed on GitHub [here](https://github.com/leedeternal/dcrd/compare/release-v1.4.0...release-v1.5.0).

### Protocol and network:

- chaincfg: Add checkpoints for 1.5.0 release ([leedeternal/dcrd#1924](https://github.com/leedeternal/dcrd/pull/1924))
- chaincfg: Introduce agenda for header cmtmts vote ([leedeternal/dcrd#1904](https://github.com/leedeternal/dcrd/pull/1904))
- multi: Implement combined merkle root and vote ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))
- blockchain: Implement v2 block filter storage ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))
- gcs/blockcf2: Implement v2 block filter creation ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))
- wire: Implement getcfilterv2/cfilterv2 messages ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))
- peer: Implement getcfilterv2/cfilterv2 listeners ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))
- server: Implement getcfilterv2 ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))
- multi: Implement header commitments and vote ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))
- server: Remove instead of disconnect node ([leedeternal/dcrd#1644](https://github.com/leedeternal/dcrd/pull/1644))
- server: limit getminingstate requests ([leedeternal/dcrd#1678](https://github.com/leedeternal/dcrd/pull/1678))
- peer: Prevent last block height going backwards ([leedeternal/dcrd#1747](https://github.com/leedeternal/dcrd/pull/1747))
- connmgr: Add ability to remove pending connections ([leedeternal/dcrd#1724](https://github.com/leedeternal/dcrd/pull/1724))
- connmgr: Add cancellation of pending requests ([leedeternal/dcrd#1724](https://github.com/leedeternal/dcrd/pull/1724))
- connmgr: Check for canceled connection before connect ([leedeternal/dcrd#1724](https://github.com/leedeternal/dcrd/pull/1724))
- multi: add automatic network address discovery ([leedeternal/dcrd#1522](https://github.com/leedeternal/dcrd/pull/1522))
- connmgr: add TorLookupIPContext, deprecate TorLookupIP ([leedeternal/dcrd#1849](https://github.com/leedeternal/dcrd/pull/1849))
- connmgr: support resolving ipv6 hosts over Tor ([leedeternal/dcrd#1908](https://github.com/leedeternal/dcrd/pull/1908))

### Transaction relay (memory pool):

- mempool: Reject same block vote double spends ([leedeternal/dcrd#1597](https://github.com/leedeternal/dcrd/pull/1597))
- mempool: Limit max vote double spends exactly ([leedeternal/dcrd#1596](https://github.com/leedeternal/dcrd/pull/1596))
- mempool: Optimize pool double spend check ([leedeternal/dcrd#1561](https://github.com/leedeternal/dcrd/pull/1561))
- txscript: Tighten standardness pubkey checks ([leedeternal/dcrd#1649](https://github.com/leedeternal/dcrd/pull/1649))
- mempool: drop container/list for simple FIFO ([leedeternal/dcrd#1681](https://github.com/leedeternal/dcrd/pull/1681))
- mempool: remove unused error return value ([leedeternal/dcrd#1785](https://github.com/leedeternal/dcrd/pull/1785))
- mempool: Add ErrorCode to returned TxRuleErrors ([leedeternal/dcrd#1901](https://github.com/leedeternal/dcrd/pull/1901))

### Mining:

- mining: Optimize get the block's votes tx ([leedeternal/dcrd#1563](https://github.com/leedeternal/dcrd/pull/1563))
- multi: add BgBlkTmplGenerator ([leedeternal/dcrd#1424](https://github.com/leedeternal/dcrd/pull/1424))
- mining: Remove unnecessary notify goroutine ([leedeternal/dcrd#1708](https://github.com/leedeternal/dcrd/pull/1708))
- mining: Improve template key handling ([leedeternal/dcrd#1709](https://github.com/leedeternal/dcrd/pull/1709))
- mining:  fix scheduled template regen ([leedeternal/dcrd#1717](https://github.com/leedeternal/dcrd/pull/1717))
- miner: Improve background generator lifecycle ([leedeternal/dcrd#1715](https://github.com/leedeternal/dcrd/pull/1715))
- cpuminer: No speed monitor on discrete mining ([leedeternal/dcrd#1716](https://github.com/leedeternal/dcrd/pull/1716))
- mining: Run vote ntfn in a separate goroutine ([leedeternal/dcrd#1718](https://github.com/leedeternal/dcrd/pull/1718))
- mining: Overhaul background template generator ([leedeternal/dcrd#1748](https://github.com/leedeternal/dcrd/pull/1748))
- mining: Remove unused error return value ([leedeternal/dcrd#1859](https://github.com/leedeternal/dcrd/pull/1859))
- cpuminer: Fix off-by-one issues in nonce handling ([leedeternal/dcrd#1865](https://github.com/leedeternal/dcrd/pull/1865))
- mining: Remove dead code ([leedeternal/dcrd#1882](https://github.com/leedeternal/dcrd/pull/1882))
- mining: Remove unused extra nonce update code ([leedeternal/dcrd#1883](https://github.com/leedeternal/dcrd/pull/1883))
- mining: Minor cleanup of aggressive mining path ([leedeternal/dcrd#1888](https://github.com/leedeternal/dcrd/pull/1888))
- mining: Remove unused error codes ([leedeternal/dcrd#1889](https://github.com/leedeternal/dcrd/pull/1889))
- mining: fix data race ([leedeternal/dcrd#1894](https://github.com/leedeternal/dcrd/pull/1894))
- mining: fix data race ([leedeternal/dcrd#1896](https://github.com/leedeternal/dcrd/pull/1896))
- cpuminer: fix race ([leedeternal/dcrd#1899](https://github.com/leedeternal/dcrd/pull/1899))
- cpuminer: Improve speed stat tracking ([leedeternal/dcrd#1921](https://github.com/leedeternal/dcrd/pull/1921))
- rpcserver/mining: Use bg tpl generator for getwork ([leedeternal/dcrd#1922](https://github.com/leedeternal/dcrd/pull/1922))
- mining: Export TemplateUpdateReason ([leedeternal/dcrd#1923](https://github.com/leedeternal/dcrd/pull/1923))
- multi: Add tpl update reason to work ntfns ([leedeternal/dcrd#1923](https://github.com/leedeternal/dcrd/pull/1923))
- mining: Store block templates given by notifywork ([leedeternal/dcrd#1949](https://github.com/leedeternal/dcrd/pull/1949))

### RPC:

- dcrjson: add cointype to WalletInfoResult ([leedeternal/dcrd#1606](https://github.com/leedeternal/dcrd/pull/1606))
- rpcclient: Introduce v2 module using wallet types ([leedeternal/dcrd#1608](https://github.com/leedeternal/dcrd/pull/1608))
- rpcserver: Update for dcrjson/v2 ([leedeternal/dcrd#1612](https://github.com/leedeternal/dcrd/pull/1612))
- rpcclient: Add EstimateSmartFee ([leedeternal/dcrd#1641](https://github.com/leedeternal/dcrd/pull/1641))
- rpcserver: remove unused quit chan ([leedeternal/dcrd#1629](https://github.com/leedeternal/dcrd/pull/1629))
- rpcserver: Undeprecate getwork ([leedeternal/dcrd#1635](https://github.com/leedeternal/dcrd/pull/1635))
- rpcserver: Add difficultyratio to getblockchaininfo ([leedeternal/dcrd#1630](https://github.com/leedeternal/dcrd/pull/1630))
- multi:  add version arg to decodescript rpc ([leedeternal/dcrd#1731](https://github.com/leedeternal/dcrd/pull/1731))
- dcrjson: Remove API breaking change ([leedeternal/dcrd#1778](https://github.com/leedeternal/dcrd/pull/1778))
- rpcclient: Add GetMasterPubkey ([leedeternal/dcrd#1777](https://github.com/leedeternal/dcrd/pull/1777))
- multi: add getnetworkinfo rpc ([leedeternal/dcrd#1536](https://github.com/leedeternal/dcrd/pull/1536))
- rpcserver: Better error message ([leedeternal/dcrd#1861](https://github.com/leedeternal/dcrd/pull/1861))
- multi: update limited user rpcs ([leedeternal/dcrd#1870](https://github.com/leedeternal/dcrd/pull/1870))
- multi: make rebroadcast winners & missed ws only ([leedeternal/dcrd#1872](https://github.com/leedeternal/dcrd/pull/1872))
- multi: remove getblocktemplate ([leedeternal/dcrd#1736](https://github.com/leedeternal/dcrd/pull/1736))
- rpcserver: Match tx filter on ticket commitments ([leedeternal/dcrd#1881](https://github.com/leedeternal/dcrd/pull/1881))
- rpcserver: don't use activeNetParams ([leedeternal/dcrd#1733](https://github.com/leedeternal/dcrd/pull/1733))
- rpcserver: update rpcAskWallet rpc set ([leedeternal/dcrd#1892](https://github.com/leedeternal/dcrd/pull/1892))
- rpcclient: close the unused response body ([leedeternal/dcrd#1905](https://github.com/leedeternal/dcrd/pull/1905))
- rpcclient: Support getcfilterv2 JSON-RPC ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))
- multi: add notifywork rpc ([leedeternal/dcrd#1410](https://github.com/leedeternal/dcrd/pull/1410))
- rpcserver: Cleanup getvoteinfo RPC ([leedeternal/dcrd#2005](https://github.com/leedeternal/dcrd/pull/2005))

### dcrd command-line flags and configuration:

- config: Remove deprecated getworkkey option ([leedeternal/dcrd#1594](https://github.com/leedeternal/dcrd/pull/1594))

### certgen utility changes:

- certgen: Support Ed25519 cert generation on Go 1.13 ([leedeternal/dcrd#1757](https://github.com/leedeternal/dcrd/pull/1757))

### dcrctl utility changes:

- dcrctl: Make version string consistent ([leedeternal/dcrd#1598](https://github.com/leedeternal/dcrd/pull/1598))
- dcrctl: Update for dcrjson/v2 and wallet types ([leedeternal/dcrd#1609](https://github.com/leedeternal/dcrd/pull/1609))
- sampleconfig: add export dcrctl sample config ([leedeternal/dcrd#2006](https://github.com/leedeternal/dcrd/pull/2006))

### promptsecret utility changes:

- promptsecret: Add -n flag to prompt multiple times ([leedeternal/dcrd#1705](https://github.com/leedeternal/dcrd/pull/1705))

### Documentation:

- docs: Update for secp256k1 v2 module ([leedeternal/dcrd#1919](https://github.com/leedeternal/dcrd/pull/1919))
- docs: document module breaking changes process ([leedeternal/dcrd#1891](https://github.com/leedeternal/dcrd/pull/1891))
- docs: Link to btc whitepaper on decred.org ([leedeternal/dcrd#1885](https://github.com/leedeternal/dcrd/pull/1885))
- docs: Update for mempool v3 module ([leedeternal/dcrd#1835](https://github.com/leedeternal/dcrd/pull/1835))
- docs: Update for peer v2 module ([leedeternal/dcrd#1834](https://github.com/leedeternal/dcrd/pull/1834))
- docs: Update for connmgr v2 module ([leedeternal/dcrd#1833](https://github.com/leedeternal/dcrd/pull/1833))
- docs: Update for mining v2 module ([leedeternal/dcrd#1831](https://github.com/leedeternal/dcrd/pull/1831))
- docs: Update for blockchain v2 module ([leedeternal/dcrd#1823](https://github.com/leedeternal/dcrd/pull/1823))
- docs: Update for rpcclient v4 module ([leedeternal/dcrd#1807](https://github.com/leedeternal/dcrd/pull/1807))
- docs: Update for blockchain/stake v2 module ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- docs: Update for database v2 module ([leedeternal/dcrd#1799](https://github.com/leedeternal/dcrd/pull/1799))
- docs: Update for rpcclient v3 module ([leedeternal/dcrd#1793](https://github.com/leedeternal/dcrd/pull/1793))
- docs: Update for dcrjson/v3 module ([leedeternal/dcrd#1792](https://github.com/leedeternal/dcrd/pull/1792))
- docs: Update for txscript v2 module ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- docs: Update for dcrutil v2 module ([leedeternal/dcrd#1770](https://github.com/leedeternal/dcrd/pull/1770))
- docs: Update for dcrec/edwards v2 module ([leedeternal/dcrd#1765](https://github.com/leedeternal/dcrd/pull/1765))
- docs: Update for chaincfg v2 module ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- docs: Update for hdkeychain v2 module ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- hdkeychain: Correct docs key examples ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- docs: allowHighFees arg has been implemented ([leedeternal/dcrd#1695](https://github.com/leedeternal/dcrd/pull/1695))
- docs: move json rpc docs to mediawiki ([leedeternal/dcrd#1687](https://github.com/leedeternal/dcrd/pull/1687))
- docs: Update for lru module ([leedeternal/dcrd#1683](https://github.com/leedeternal/dcrd/pull/1683))
- docs: fix formatting in json rpc doc ([leedeternal/dcrd#1633](https://github.com/leedeternal/dcrd/pull/1633))
- docs: Update for mempool v2 module ([leedeternal/dcrd#1613](https://github.com/leedeternal/dcrd/pull/1613))
- docs: Update for rpcclient v2 module ([leedeternal/dcrd#1608](https://github.com/leedeternal/dcrd/pull/1608))
- docs: Update for dcrjson v2 module ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- jsonrpc/types: Add README.md and doc.go ([leedeternal/dcrd#1794](https://github.com/leedeternal/dcrd/pull/1794))
- dcrjson: Update README.md ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- dcrec/secp256k1: Update README.md broken link ([leedeternal/dcrd#1631](https://github.com/leedeternal/dcrd/pull/1631))
- bech32: Correct README build badge reference ([leedeternal/dcrd#1689](https://github.com/leedeternal/dcrd/pull/1689))
- hdkeychain: Update README.md ([leedeternal/dcrd#1686](https://github.com/leedeternal/dcrd/pull/1686))
- bech32: Correct README links ([leedeternal/dcrd#1691](https://github.com/leedeternal/dcrd/pull/1691))
- stake: Remove unnecessary language in comment ([leedeternal/dcrd#1752](https://github.com/leedeternal/dcrd/pull/1752))
- multi: Use https links where available ([leedeternal/dcrd#1771](https://github.com/leedeternal/dcrd/pull/1771))
- stake: Make doc.go formatting consistent ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- blockchain: Update doc.go to reflect reality ([leedeternal/dcrd#1823](https://github.com/leedeternal/dcrd/pull/1823))
- multi: update rpc documentation ([leedeternal/dcrd#1867](https://github.com/leedeternal/dcrd/pull/1867))
- dcrec: fix examples links ([leedeternal/dcrd#1914](https://github.com/leedeternal/dcrd/pull/1914))
- gcs: Improve package documentation ([leedeternal/dcrd#1915](https://github.com/leedeternal/dcrd/pull/1915))

### Developer-related package and module changes:

- dcrutil: Return deep copied tx in NewTxDeepTxIns ([leedeternal/dcrd#1545](https://github.com/leedeternal/dcrd/pull/1545))
- mining: Remove superfluous error check ([leedeternal/dcrd#1552](https://github.com/leedeternal/dcrd/pull/1552))
- dcrutil: Block does not cache the header bytes ([leedeternal/dcrd#1571](https://github.com/leedeternal/dcrd/pull/1571))
- blockchain: Remove superfluous GetVoteInfo check ([leedeternal/dcrd#1574](https://github.com/leedeternal/dcrd/pull/1574))
- blockchain: Make consensus votes network agnostic ([leedeternal/dcrd#1590](https://github.com/leedeternal/dcrd/pull/1590))
- blockchain: Optimize skip stakebase input ([leedeternal/dcrd#1565](https://github.com/leedeternal/dcrd/pull/1565))
- txscript: code cleanup ([leedeternal/dcrd#1591](https://github.com/leedeternal/dcrd/pull/1591))
- dcrjson: Move estimate fee test to matching file ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- dcrjson: Move raw stake tx cmds to correct file ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- dcrjson: Move best block result to correct file ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- dcrjson: Move winning tickets ntfn to correct file ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- dcrjson: Move spent tickets ntfn to correct file ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- dcrjson: Move stake diff ntfn to correct file ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- dcrjson: Move new tickets ntfn to correct file ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- txscript: Rename p2sh indicator to isP2SH ([leedeternal/dcrd#1605](https://github.com/leedeternal/dcrd/pull/1605))
- mempool: Remove deprecated min high prio constant ([leedeternal/dcrd#1613](https://github.com/leedeternal/dcrd/pull/1613))
- mempool: Remove tight coupling with dcrjson ([leedeternal/dcrd#1613](https://github.com/leedeternal/dcrd/pull/1613))
- blockmanager: only check if current once handling inv's ([leedeternal/dcrd#1621](https://github.com/leedeternal/dcrd/pull/1621))
- connmngr: Add DialAddr config option ([leedeternal/dcrd#1642](https://github.com/leedeternal/dcrd/pull/1642))
- txscript: Consistent checksigaltverify handling ([leedeternal/dcrd#1647](https://github.com/leedeternal/dcrd/pull/1647))
- multi: preallocate memory ([leedeternal/dcrd#1646](https://github.com/leedeternal/dcrd/pull/1646))
- wire: Fix maximum payload length of MsgAddr ([leedeternal/dcrd#1638](https://github.com/leedeternal/dcrd/pull/1638))
- blockmanager: remove unused requestedEverTxns ([leedeternal/dcrd#1624](https://github.com/leedeternal/dcrd/pull/1624))
- blockmanager: remove useless requestedEverBlocks ([leedeternal/dcrd#1624](https://github.com/leedeternal/dcrd/pull/1624))
- txscript: Introduce constant for max CLTV bytes ([leedeternal/dcrd#1650](https://github.com/leedeternal/dcrd/pull/1650))
- txscript: Introduce constant for max CSV bytes ([leedeternal/dcrd#1651](https://github.com/leedeternal/dcrd/pull/1651))
- chaincfg: Remove unused definition ([leedeternal/dcrd#1661](https://github.com/leedeternal/dcrd/pull/1661))
- chaincfg: Use expected regnet merkle root var ([leedeternal/dcrd#1662](https://github.com/leedeternal/dcrd/pull/1662))
- blockchain: Deprecate BlockOneCoinbasePaysTokens ([leedeternal/dcrd#1657](https://github.com/leedeternal/dcrd/pull/1657))
- blockchain: Explicit script ver in coinbase checks ([leedeternal/dcrd#1658](https://github.com/leedeternal/dcrd/pull/1658))
- chaincfg: Explicit unique net addr prefix ([leedeternal/dcrd#1663](https://github.com/leedeternal/dcrd/pull/1663))
- chaincfg: Introduce params lookup by addr prefix ([leedeternal/dcrd#1664](https://github.com/leedeternal/dcrd/pull/1664))
- dcrutil: Lookup params by addr prefix in chaincfg ([leedeternal/dcrd#1665](https://github.com/leedeternal/dcrd/pull/1665))
- peer: Deprecate dependency on chaincfg ([leedeternal/dcrd#1671](https://github.com/leedeternal/dcrd/pull/1671))
- server: Update for deprecated peer chaincfg ([leedeternal/dcrd#1671](https://github.com/leedeternal/dcrd/pull/1671))
- fees: drop unused chaincfg ([leedeternal/dcrd#1675](https://github.com/leedeternal/dcrd/pull/1675))
- lru: Implement a new module with generic LRU cache ([leedeternal/dcrd#1683](https://github.com/leedeternal/dcrd/pull/1683))
- peer: Use lru cache module for inventory ([leedeternal/dcrd#1683](https://github.com/leedeternal/dcrd/pull/1683))
- peer: Use lru cache module for nonces ([leedeternal/dcrd#1683](https://github.com/leedeternal/dcrd/pull/1683))
- server: Use lru cache module for addresses ([leedeternal/dcrd#1683](https://github.com/leedeternal/dcrd/pull/1683))
- multi: drop init and just set default log ([leedeternal/dcrd#1676](https://github.com/leedeternal/dcrd/pull/1676))
- multi: deprecate DisableLog ([leedeternal/dcrd#1676](https://github.com/leedeternal/dcrd/pull/1676))
- blockchain: Remove unused params from block index ([leedeternal/dcrd#1674](https://github.com/leedeternal/dcrd/pull/1674))
- bech32: Initial Version ([leedeternal/dcrd#1646](https://github.com/leedeternal/dcrd/pull/1646))
- chaincfg: Add extended key accessor funcs ([leedeternal/dcrd#1694](https://github.com/leedeternal/dcrd/pull/1694))
- chaincfg: Rename extended key accessor funcs ([leedeternal/dcrd#1699](https://github.com/leedeternal/dcrd/pull/1699))
- wire: Accurate calculations of maximum length ([leedeternal/dcrd#1672](https://github.com/leedeternal/dcrd/pull/1672))
- wire: Fix MsgCFTypes maximum payload length ([leedeternal/dcrd#1673](https://github.com/leedeternal/dcrd/pull/1673))
- txscript: Deprecate HasP2SHScriptSigStakeOpCodes ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Deprecate IsStakeOutput ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Deprecate GetMultisigMandN ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Introduce zero-alloc script tokenizer ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize script disasm ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Introduce raw script sighash calc func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize CalcSignatureHash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make isSmallInt accept raw opcode ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make asSmallInt accept raw opcode ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make isStakeOpcode accept raw opcode ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize IsPayToScriptHash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize IsMultisigScript ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize IsMultisigSigScript ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize GetSigOpCount ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize isAnyKindOfScriptHash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize IsPushOnlyScript ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize new engine push only script ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Check p2sh push before parsing scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize GetPreciseSigOpCount ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make typeOfScript accept raw script ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript pay-to-script-hash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isScriptHash function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript multisig ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isMultiSig function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript pay-to-pubkey ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isPubkey function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript pay-to-alt-pubkey ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript pay-to-pubkey-hash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isPubkeyHash function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript pay-to-alt-pk-hash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript nulldata detection ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isNullData function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript stakesub detection ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isStakeSubmission function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript stakegen detection ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isStakeGen function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript stakerev detection ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isStakeRevocation function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize typeOfScript stakechange detect ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isSStxChange function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ContainsStakeOpCodes ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractCoinbaseNullData ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Convert CalcScriptInfo ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isPushOnly function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused getSigOpCount function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize CalcMultiSigStats ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize multi sig redeem script func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Convert GetScriptHashFromP2SHScript ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize PushedData ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize IsUnspendable ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make canonicalPush accept raw opcode ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractAtomicSwapDataPushes ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs scripthash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs pubkeyhash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs altpubkeyhash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs pubkey ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs altpubkey ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs multisig ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs stakesub ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs stakegen ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs stakerev ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs stakechange ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAddrs nulldata ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Optimize ExtractPkScriptAltSigType ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused extractOneBytePush func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isPubkeyAlt function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isPubkeyHashAlt function ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused isOneByteMaxDataPush func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: mergeMultiSig function def order cleanup ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Use raw scripts in RawTxInSignature ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Use raw scripts in RawTxInSignatureAlt ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Correct p2pkSignatureScriptAlt comment ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Use raw scripts in SignTxOutput ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Implement efficient opcode data removal ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make isDisabled accept raw opcode ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make alwaysIllegal accept raw opcode ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make isConditional accept raw opcode ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make min push accept raw opcode and data ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Convert to use non-parsed opcode disasm ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Refactor engine to use raw scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused removeOpcodeByData func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Rename removeOpcodeByDataRaw func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused calcSignatureHash func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Rename calcSignatureHashRaw func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused parseScript func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused unparseScript func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused parsedOpcode.bytes func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Remove unused parseScriptTemplate func ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make executeOpcode take opcode and data ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Make op callbacks take opcode and data ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- dcrutil: Fix NewTxDeepTxIns implementation ([leedeternal/dcrd#1685](https://github.com/leedeternal/dcrd/pull/1685))
- stake: drop txscript.DefaultScriptVersion usage ([leedeternal/dcrd#1704](https://github.com/leedeternal/dcrd/pull/1704))
- peer: invSendQueue is a FIFO ([leedeternal/dcrd#1680](https://github.com/leedeternal/dcrd/pull/1680))
- peer: pendingMsgs is a FIFO ([leedeternal/dcrd#1680](https://github.com/leedeternal/dcrd/pull/1680))
- blockchain: drop container/list ([leedeternal/dcrd#1682](https://github.com/leedeternal/dcrd/pull/1682))
- blockmanager: use local var for the request queue ([leedeternal/dcrd#1622](https://github.com/leedeternal/dcrd/pull/1622))
- server: return on outbound peer creation error ([leedeternal/dcrd#1637](https://github.com/leedeternal/dcrd/pull/1637))
- hdkeychain: Remove Address method ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- hdkeychain: Remove SetNet method ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- hdkeychain: Require network on decode extended key ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- hdkeychain: Don't rely on global state ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- hdkeychain: Introduce NetworkParams interface ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- server: Remove unused ScheduleShutdown func ([leedeternal/dcrd#1711](https://github.com/leedeternal/dcrd/pull/1711))
- server: Remove unused dynamicTickDuration func ([leedeternal/dcrd#1711](https://github.com/leedeternal/dcrd/pull/1711))
- main: Convert signal handling to use context ([leedeternal/dcrd#1712](https://github.com/leedeternal/dcrd/pull/1712))
- txscript: Remove checks for impossible conditions ([leedeternal/dcrd#1713](https://github.com/leedeternal/dcrd/pull/1713))
- indexers: Remove unused func ([leedeternal/dcrd#1714](https://github.com/leedeternal/dcrd/pull/1714))
- multi: fix onVoteReceivedHandler shutdown ([leedeternal/dcrd#1721](https://github.com/leedeternal/dcrd/pull/1721))
- wire: Rename extended errors to malformed errors ([leedeternal/dcrd#1742](https://github.com/leedeternal/dcrd/pull/1742))
- rpcwebsocket: convert from list to simple FIFO ([leedeternal/dcrd#1726](https://github.com/leedeternal/dcrd/pull/1726))
- dcrec: implement GenerateKey ([leedeternal/dcrd#1652](https://github.com/leedeternal/dcrd/pull/1652))
- txscript: Remove SigHashOptimization constant ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- txscript: Remove CheckForDuplicateHashes constant ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- txscript: Remove CPUMinerThreads constant ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Move DNSSeed stringer next to type def ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Remove all registration capabilities ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Move mainnet code to mainnet files ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Move testnet3 code to testnet files ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Move simnet code to testnet files ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Move regnet code to regnet files ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Concrete genesis hash in Params struct ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Use scripts in block one token payouts ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Convert global param defs to funcs ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- edwards: remove curve param ([leedeternal/dcrd#1762](https://github.com/leedeternal/dcrd/pull/1762))
- edwards: unexport EncodedBytesToBigIntPoint ([leedeternal/dcrd#1762](https://github.com/leedeternal/dcrd/pull/1762))
- edwards: unexport a slew of funcs ([leedeternal/dcrd#1762](https://github.com/leedeternal/dcrd/pull/1762))
- edwards: add signature IsEqual and Verify methods ([leedeternal/dcrd#1762](https://github.com/leedeternal/dcrd/pull/1762))
- edwards: add Sign method to PrivateKey ([leedeternal/dcrd#1762](https://github.com/leedeternal/dcrd/pull/1762))
- chaincfg: Add addr params accessor funcs ([leedeternal/dcrd#1766](https://github.com/leedeternal/dcrd/pull/1766))
- schnorr: remove curve param ([leedeternal/dcrd#1764](https://github.com/leedeternal/dcrd/pull/1764))
- schnorr: unexport functions ([leedeternal/dcrd#1764](https://github.com/leedeternal/dcrd/pull/1764))
- schnorr: add signature IsEqual and Verify methods ([leedeternal/dcrd#1764](https://github.com/leedeternal/dcrd/pull/1764))
- secp256k1: unexport NAF ([leedeternal/dcrd#1764](https://github.com/leedeternal/dcrd/pull/1764))
- addrmgr: drop container/list ([leedeternal/dcrd#1679](https://github.com/leedeternal/dcrd/pull/1679))
- dcrutil: Remove unused ErrAddressCollision ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcurtil: Remove unused ErrMissingDefaultNet ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Require network on address decode ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Remove IsForNet from Address interface ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Remove DSA from Address interface ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Remove Net from Address interface ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Rename EncodeAddress to Address ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Don't store net ref in addr impls ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Require network on WIF decode ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Accept magic bytes directly in NewWIF ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Introduce AddressParams interface ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- blockchain: Do coinbase nulldata check locally ([leedeternal/dcrd#1770](https://github.com/leedeternal/dcrd/pull/1770))
- blockchain: update CalcBlockSubsidy ([leedeternal/dcrd#1750](https://github.com/leedeternal/dcrd/pull/1750))
- txscript: Use const for sighashall optimization ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Remove DisableLog ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Unexport HasP2SHScriptSigStakeOpCodes ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Remove third GetPreciseSigOpCount param ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Remove IsMultisigScript err return ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Unexport IsStakeOutput ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Remove CalcScriptInfo ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Remove multisig redeem script err return ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Remove GetScriptHashFromP2SHScript ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Remove GetMultisigMandN ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Remove DefaultScriptVersion ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Use secp256k1 types in sig cache ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- multi: decouple BlockManager from server ([leedeternal/dcrd#1728](https://github.com/leedeternal/dcrd/pull/1728))
- database: Introduce BlockSerializer interface ([leedeternal/dcrd#1799](https://github.com/leedeternal/dcrd/pull/1799))
- hdkeychain: Add ChildNum and Depth methods ([leedeternal/dcrd#1800](https://github.com/leedeternal/dcrd/pull/1800))
- chaincfg: Avoid block 1 subsidy codegen explosion ([leedeternal/dcrd#1801](https://github.com/leedeternal/dcrd/pull/1801))
- chaincfg: Add stake params accessor funcs ([leedeternal/dcrd#1802](https://github.com/leedeternal/dcrd/pull/1802))
- stake: Remove DisableLog ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- stake: Remove unused TxSSGenStakeOutputInfo ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- stake: Remove unused TxSSRtxStakeOutputInfo ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- stake: Remove unused SetTxTree ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- stake: Introduce StakeParams interface ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- stake: Accept AddressParams for ticket commit addr ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- gcs: Optimize AddSigScript ([leedeternal/dcrd#1804](https://github.com/leedeternal/dcrd/pull/1804))
- chaincfg: Add subsidy params accessor funcs ([leedeternal/dcrd#1813](https://github.com/leedeternal/dcrd/pull/1813))
- blockchain/standalone: Implement a new module ([leedeternal/dcrd#1808](https://github.com/leedeternal/dcrd/pull/1808))
- blockchain/standalone: Add merkle root calc funcs ([leedeternal/dcrd#1809](https://github.com/leedeternal/dcrd/pull/1809))
- blockchain/standalone: Add subsidy calc funcs ([leedeternal/dcrd#1812](https://github.com/leedeternal/dcrd/pull/1812))
- blockchain/standalone: Add IsCoinBaseTx ([leedeternal/dcrd#1815](https://github.com/leedeternal/dcrd/pull/1815))
- crypto/blake256: Add module with zero alloc funcs ([leedeternal/dcrd#1811](https://github.com/leedeternal/dcrd/pull/1811))
- stake: Check minimum req outputs for votes earlier ([leedeternal/dcrd#1819](https://github.com/leedeternal/dcrd/pull/1819))
- blockchain: Use standalone module for merkle calcs ([leedeternal/dcrd#1816](https://github.com/leedeternal/dcrd/pull/1816))
- blockchain: Use standalone for coinbase checks ([leedeternal/dcrd#1816](https://github.com/leedeternal/dcrd/pull/1816))
- blockchain: Use standalone module subsidy calcs ([leedeternal/dcrd#1816](https://github.com/leedeternal/dcrd/pull/1816))
- blockchain: Use standalone module for work funcs ([leedeternal/dcrd#1816](https://github.com/leedeternal/dcrd/pull/1816))
- blockchain: Remove deprecated code ([leedeternal/dcrd#1823](https://github.com/leedeternal/dcrd/pull/1823))
- blockchain: Accept subsidy cache in config ([leedeternal/dcrd#1823](https://github.com/leedeternal/dcrd/pull/1823))
- mining: Use lastest major version deps ([leedeternal/dcrd#1831](https://github.com/leedeternal/dcrd/pull/1831))
- connmgr: Accept DNS seeds as string slice ([leedeternal/dcrd#1833](https://github.com/leedeternal/dcrd/pull/1833))
- peer: Remove deprecated Config.ChainParams field ([leedeternal/dcrd#1834](https://github.com/leedeternal/dcrd/pull/1834))
- peer: Accept hash slice for block locators ([leedeternal/dcrd#1834](https://github.com/leedeternal/dcrd/pull/1834))
- peer: Use latest major version deps ([leedeternal/dcrd#1834](https://github.com/leedeternal/dcrd/pull/1834))
- mempool: Use latest major version deps ([leedeternal/dcrd#1835](https://github.com/leedeternal/dcrd/pull/1835))
- main: Update to use all new major module versions ([leedeternal/dcrd#1837](https://github.com/leedeternal/dcrd/pull/1837))
- blockchain: Implement stricter bounds checking ([leedeternal/dcrd#1825](https://github.com/leedeternal/dcrd/pull/1825))
- gcs: Start v2 module dev cycle ([leedeternal/dcrd#1843](https://github.com/leedeternal/dcrd/pull/1843))
- gcs: Support empty filters ([leedeternal/dcrd#1844](https://github.com/leedeternal/dcrd/pull/1844))
- gcs: Make error consistent with rest of codebase ([leedeternal/dcrd#1846](https://github.com/leedeternal/dcrd/pull/1846))
- gcs: Add filter version support ([leedeternal/dcrd#1848](https://github.com/leedeternal/dcrd/pull/1848))
- gcs: Correct zero hash filter matches ([leedeternal/dcrd#1857](https://github.com/leedeternal/dcrd/pull/1857))
- gcs: Standardize serialization on a single format ([leedeternal/dcrd#1851](https://github.com/leedeternal/dcrd/pull/1851))
- gcs: Optimize Hash ([leedeternal/dcrd#1853](https://github.com/leedeternal/dcrd/pull/1853))
- gcs: Group V1 filter funcs after filter defs ([leedeternal/dcrd#1854](https://github.com/leedeternal/dcrd/pull/1854))
- gcs: Support independent fp rate and bin size ([leedeternal/dcrd#1854](https://github.com/leedeternal/dcrd/pull/1854))
- blockchain: Refactor best chain state init ([leedeternal/dcrd#1871](https://github.com/leedeternal/dcrd/pull/1871))
- gcs: Implement version 2 filters ([leedeternal/dcrd#1856](https://github.com/leedeternal/dcrd/pull/1856))
- blockchain: Cleanup subsidy cache init order ([leedeternal/dcrd#1873](https://github.com/leedeternal/dcrd/pull/1873))
- multi: use chain ref. from blockmanager config ([leedeternal/dcrd#1879](https://github.com/leedeternal/dcrd/pull/1879))
- multi: remove unused funcs and vars ([leedeternal/dcrd#1880](https://github.com/leedeternal/dcrd/pull/1880))
- gcs: Prevent empty data elements in v2 filters ([leedeternal/dcrd#1911](https://github.com/leedeternal/dcrd/pull/1911))
- crypto: import ripemd160 ([leedeternal/dcrd#1907](https://github.com/leedeternal/dcrd/pull/1907))
- multi: Use secp256k1/v2 module ([leedeternal/dcrd#1919](https://github.com/leedeternal/dcrd/pull/1919))
- multi: Use crypto/ripemd160 module ([leedeternal/dcrd#1918](https://github.com/leedeternal/dcrd/pull/1918))
- multi: Use dcrec/edwards/v2 module ([leedeternal/dcrd#1920](https://github.com/leedeternal/dcrd/pull/1920))
- gcs: Prevent empty data elements fp matches ([leedeternal/dcrd#1940](https://github.com/leedeternal/dcrd/pull/1940))
- main: Update to use all new module versions ([leedeternal/dcrd#1946](https://github.com/leedeternal/dcrd/pull/1946))
- blockchain/standalone: Add inclusion proof funcs ([leedeternal/dcrd#1906](https://github.com/leedeternal/dcrd/pull/1906))

### Developer-related module management:

- build: Require dcrjson v1.2.0 ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- multi: Remove non-root module replacements ([leedeternal/dcrd#1599](https://github.com/leedeternal/dcrd/pull/1599))
- dcrjson: Introduce v2 module without wallet types ([leedeternal/dcrd#1607](https://github.com/leedeternal/dcrd/pull/1607))
- release: Freeze version 1 mempool module ([leedeternal/dcrd#1613](https://github.com/leedeternal/dcrd/pull/1613))
- release: Introduce mempool v2 module ([leedeternal/dcrd#1613](https://github.com/leedeternal/dcrd/pull/1613))
- main: Tidy module to latest ([leedeternal/dcrd#1613](https://github.com/leedeternal/dcrd/pull/1613))
- main: Update for mempool/v2 ([leedeternal/dcrd#1616](https://github.com/leedeternal/dcrd/pull/1616))
- multi: Add go 1.11 directive to all modules ([leedeternal/dcrd#1677](https://github.com/leedeternal/dcrd/pull/1677))
- build: Tidy module sums (go mod tidy) ([leedeternal/dcrd#1692](https://github.com/leedeternal/dcrd/pull/1692))
- release: Freeze version 1 hdkeychain module ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- release: Introduce hdkeychain v2 module ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- release: Freeze version 1 chaincfg module ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Introduce chaincfg v2 module ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- chaincfg: Use dcrec/edwards/v1.0.0 ([leedeternal/dcrd#1758](https://github.com/leedeternal/dcrd/pull/1758))
- dcrutil: Prepare v1.3.0 ([leedeternal/dcrd#1761](https://github.com/leedeternal/dcrd/pull/1761))
- release: freeze version 1 dcrec/edwards module ([leedeternal/dcrd#1762](https://github.com/leedeternal/dcrd/pull/1762))
- edwards: Introduce v2 module ([leedeternal/dcrd#1762](https://github.com/leedeternal/dcrd/pull/1762))
- release: freeze version 1 dcrec/secp256k1 module ([leedeternal/dcrd#1764](https://github.com/leedeternal/dcrd/pull/1764))
- secp256k1: Introduce v2 module ([leedeternal/dcrd#1764](https://github.com/leedeternal/dcrd/pull/1764))
- multi: Update all modules for chaincfg v1.5.1 ([leedeternal/dcrd#1768](https://github.com/leedeternal/dcrd/pull/1768))
- release: Freeze version 1 dcrutil module ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Update to use chaincfg/v2 module ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- release: Introduce dcrutil v2 module ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- database: Use chaincfg/v2 ([leedeternal/dcrd#1772](https://github.com/leedeternal/dcrd/pull/1772))
- txscript: Prepare v1.1.0 ([leedeternal/dcrd#1773](https://github.com/leedeternal/dcrd/pull/1773))
- stake: Prepare v1.2.0 ([leedeternal/dcrd#1775](https://github.com/leedeternal/dcrd/pull/1775))
- release: Freeze version 1 txscript module ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- txscript: Use dcrutil/v2 ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- release: Introduce txscript v2 module ([leedeternal/dcrd#1774](https://github.com/leedeternal/dcrd/pull/1774))
- main: Add requires for new version modules ([leedeternal/dcrd#1776](https://github.com/leedeternal/dcrd/pull/1776))
- dcrjson: Introduce v3 and move types to module ([leedeternal/dcrd#1779](https://github.com/leedeternal/dcrd/pull/1779))
- jsonrpc/types: Prepare 1.0.0 ([leedeternal/dcrd#1787](https://github.com/leedeternal/dcrd/pull/1787))
- main: Use latest JSON-RPC types ([leedeternal/dcrd#1789](https://github.com/leedeternal/dcrd/pull/1789))
- multi: Use decred fork of go-socks ([leedeternal/dcrd#1790](https://github.com/leedeternal/dcrd/pull/1790))
- rpcclient: Prepare v2.1.0 ([leedeternal/dcrd#1791](https://github.com/leedeternal/dcrd/pull/1791))
- release: Freeze version 2 rpcclient module ([leedeternal/dcrd#1793](https://github.com/leedeternal/dcrd/pull/1793))
- rpcclient: Use dcrjson/v3 ([leedeternal/dcrd#1793](https://github.com/leedeternal/dcrd/pull/1793))
- release: Introduce rpcclient v3 module ([leedeternal/dcrd#1793](https://github.com/leedeternal/dcrd/pull/1793))
- main: Use rpcclient/v3 ([leedeternal/dcrd#1795](https://github.com/leedeternal/dcrd/pull/1795))
- hdkeychain: Prepare v2.0.1 ([leedeternal/dcrd#1798](https://github.com/leedeternal/dcrd/pull/1798))
- release: Freeze version 1 database module ([leedeternal/dcrd#1799](https://github.com/leedeternal/dcrd/pull/1799))
- database: Use dcrutil/v2 ([leedeternal/dcrd#1799](https://github.com/leedeternal/dcrd/pull/1799))
- release: Introduce database v2 module ([leedeternal/dcrd#1799](https://github.com/leedeternal/dcrd/pull/1799))
- release: Freeze version 1 blockchain/stake module ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- stake: Use dcrutil/v2 and chaincfg/v2 ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- Use txscript/v2 ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- stake: Use database/v2 ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- release: Introduce blockchain/stake v2 module ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- gcs: Use txscript/v2 ([leedeternal/dcrd#1804](https://github.com/leedeternal/dcrd/pull/1804))
- gcs: Prepare v1.1.0 ([leedeternal/dcrd#1804](https://github.com/leedeternal/dcrd/pull/1804))
- release: Freeze version 3 rpcclient module ([leedeternal/dcrd#1807](https://github.com/leedeternal/dcrd/pull/1807))
- rpcclient: Use dcrutil/v2 and chaincfg/v2 ([leedeternal/dcrd#1807](https://github.com/leedeternal/dcrd/pull/1807))
- release: Introduce rpcclient v4 module ([leedeternal/dcrd#1807](https://github.com/leedeternal/dcrd/pull/1807))
- blockchain/standalone: Prepare v1.0.0 ([leedeternal/dcrd#1817](https://github.com/leedeternal/dcrd/pull/1817))
- multi: Use crypto/blake256 ([leedeternal/dcrd#1818](https://github.com/leedeternal/dcrd/pull/1818))
- main: Consume latest module minors and patches ([leedeternal/dcrd#1822](https://github.com/leedeternal/dcrd/pull/1822))
- blockchain: Prepare v1.2.0 ([leedeternal/dcrd#1820](https://github.com/leedeternal/dcrd/pull/1820))
- mining: Prepare v1.1.1 ([leedeternal/dcrd#1826](https://github.com/leedeternal/dcrd/pull/1826))
- release: Freeze version 1 blockchain module use ([leedeternal/dcrd#1823](https://github.com/leedeternal/dcrd/pull/1823))
- blockchain: Use lastest major version deps ([leedeternal/dcrd#1823](https://github.com/leedeternal/dcrd/pull/1823))
- release: Introduce blockchain v2 module ([leedeternal/dcrd#1823](https://github.com/leedeternal/dcrd/pull/1823))
- connmgr: Prepare v1.1.0 ([leedeternal/dcrd#1828](https://github.com/leedeternal/dcrd/pull/1828))
- peer: Prepare v1.2.0 ([leedeternal/dcrd#1830](https://github.com/leedeternal/dcrd/pull/1830))
- release: Freeze version 1 mining module use ([leedeternal/dcrd#1831](https://github.com/leedeternal/dcrd/pull/1831))
- release: Introduce mining v2 module ([leedeternal/dcrd#1831](https://github.com/leedeternal/dcrd/pull/1831))
- mempool: Prepare v2.1.0 ([leedeternal/dcrd#1832](https://github.com/leedeternal/dcrd/pull/1832))
- release: Freeze version 1 connmgr module use ([leedeternal/dcrd#1833](https://github.com/leedeternal/dcrd/pull/1833))
- release: Introduce connmgr v2 module ([leedeternal/dcrd#1833](https://github.com/leedeternal/dcrd/pull/1833))
- release: Freeze version 1 peer module use ([leedeternal/dcrd#1834](https://github.com/leedeternal/dcrd/pull/1834))
- release: Introduce peer v2 module ([leedeternal/dcrd#1834](https://github.com/leedeternal/dcrd/pull/1834))
- blockchain: Prepare v2.0.1 ([leedeternal/dcrd#1836](https://github.com/leedeternal/dcrd/pull/1836))
- release: Freeze version 2 mempool module use ([leedeternal/dcrd#1835](https://github.com/leedeternal/dcrd/pull/1835))
- release: Introduce mempool v3 module ([leedeternal/dcrd#1835](https://github.com/leedeternal/dcrd/pull/1835))
- go.mod: sync ([leedeternal/dcrd#1913](https://github.com/leedeternal/dcrd/pull/1913))
- secp256k1: Prepare v2.0.0 ([leedeternal/dcrd#1916](https://github.com/leedeternal/dcrd/pull/1916))
- wire: Prepare v1.3.0 ([leedeternal/dcrd#1925](https://github.com/leedeternal/dcrd/pull/1925))
- chaincfg: Prepare v2.3.0 ([leedeternal/dcrd#1926](https://github.com/leedeternal/dcrd/pull/1926))
- dcrjson: Prepare v3.0.1 ([leedeternal/dcrd#1927](https://github.com/leedeternal/dcrd/pull/1927))
- rpc/jsonrpc/types: Prepare v2.0.0 ([leedeternal/dcrd#1928](https://github.com/leedeternal/dcrd/pull/1928))
- dcrutil: Prepare v2.0.1 ([leedeternal/dcrd#1929](https://github.com/leedeternal/dcrd/pull/1929))
- blockchain/standalone: Prepare v1.1.0 ([leedeternal/dcrd#1930](https://github.com/leedeternal/dcrd/pull/1930))
- txscript: Prepare v2.1.0 ([leedeternal/dcrd#1931](https://github.com/leedeternal/dcrd/pull/1931))
- database: Prepare v2.0.1 ([leedeternal/dcrd#1932](https://github.com/leedeternal/dcrd/pull/1932))
- blockchain/stake: Prepare v2.0.2 ([leedeternal/dcrd#1933](https://github.com/leedeternal/dcrd/pull/1933))
- gcs: Prepare v2.0.0 ([leedeternal/dcrd#1934](https://github.com/leedeternal/dcrd/pull/1934))
- blockchain: Prepare v2.1.0 ([leedeternal/dcrd#1935](https://github.com/leedeternal/dcrd/pull/1935))
- addrmgr: Prepare v1.1.0 ([leedeternal/dcrd#1936](https://github.com/leedeternal/dcrd/pull/1936))
- connmgr: Prepare v2.1.0 ([leedeternal/dcrd#1937](https://github.com/leedeternal/dcrd/pull/1937))
- hdkeychain: Prepare v2.1.0 ([leedeternal/dcrd#1938](https://github.com/leedeternal/dcrd/pull/1938))
- peer: Prepare v2.1.0 ([leedeternal/dcrd#1939](https://github.com/leedeternal/dcrd/pull/1939))
- fees: Prepare v2.0.0 ([leedeternal/dcrd#1941](https://github.com/leedeternal/dcrd/pull/1941))
- rpcclient: Prepare v4.1.0 ([leedeternal/dcrd#1943](https://github.com/leedeternal/dcrd/pull/1943))
- mining: Prepare v2.0.1 ([leedeternal/dcrd#1944](https://github.com/leedeternal/dcrd/pull/1944))
- mempool: Prepare v3.1.0 ([leedeternal/dcrd#1945](https://github.com/leedeternal/dcrd/pull/1945))

### Testing and Quality Assurance:

- mempool: Accept test mungers for vote tx ([leedeternal/dcrd#1595](https://github.com/leedeternal/dcrd/pull/1595))
- build: Replace TravisCI with CI via Github actions ([leedeternal/dcrd#1903](https://github.com/leedeternal/dcrd/pull/1903))
- build: Setup github actions for CI ([leedeternal/dcrd#1902](https://github.com/leedeternal/dcrd/pull/1902))
- TravisCI: Recommended install for golangci-lint ([leedeternal/dcrd#1808](https://github.com/leedeternal/dcrd/pull/1808))
- TravisCI: Use more portable module ver stripping ([leedeternal/dcrd#1784](https://github.com/leedeternal/dcrd/pull/1784))
- TravisCI: Test and lint latest version modules ([leedeternal/dcrd#1776](https://github.com/leedeternal/dcrd/pull/1776))
- TravisCI: Disable race detector ([leedeternal/dcrd#1749](https://github.com/leedeternal/dcrd/pull/1749))
- TravisCI: Set ./run_tests.sh executable perms ([leedeternal/dcrd#1648](https://github.com/leedeternal/dcrd/pull/1648))
- travis: bump golangci-lint to v1.18.0 ([leedeternal/dcrd#1890](https://github.com/leedeternal/dcrd/pull/1890))
- travis: Test go1.13 and drop go1.11 ([leedeternal/dcrd#1875](https://github.com/leedeternal/dcrd/pull/1875))
- travis: Allow staged builds with build cache ([leedeternal/dcrd#1797](https://github.com/leedeternal/dcrd/pull/1797))
- travis: drop docker and test directly ([leedeternal/dcrd#1783](https://github.com/leedeternal/dcrd/pull/1783))
- travis: test go1.12 ([leedeternal/dcrd#1627](https://github.com/leedeternal/dcrd/pull/1627))
- travis: Add misspell linter ([leedeternal/dcrd#1618](https://github.com/leedeternal/dcrd/pull/1618))
- travis: run linters in each module ([leedeternal/dcrd#1601](https://github.com/leedeternal/dcrd/pull/1601))
- multi: switch to golangci-lint ([leedeternal/dcrd#1575](https://github.com/leedeternal/dcrd/pull/1575))
- blockchain: Consistent legacy seq lock tests ([leedeternal/dcrd#1580](https://github.com/leedeternal/dcrd/pull/1580))
- blockchain: Add test logic to find deployments ([leedeternal/dcrd#1581](https://github.com/leedeternal/dcrd/pull/1581))
- blockchain: Introduce chaingen test harness ([leedeternal/dcrd#1583](https://github.com/leedeternal/dcrd/pull/1583))
- blockchain: Use harness in force head reorg tests ([leedeternal/dcrd#1584](https://github.com/leedeternal/dcrd/pull/1584))
- blockchain: Use harness in stake version tests ([leedeternal/dcrd#1585](https://github.com/leedeternal/dcrd/pull/1585))
- blockchain: Use harness in checkblktemplate tests ([leedeternal/dcrd#1586](https://github.com/leedeternal/dcrd/pull/1586))
- blockchain: Use harness in threshold state tests ([leedeternal/dcrd#1587](https://github.com/leedeternal/dcrd/pull/1587))
- blockchain: Use harness in legacy seqlock tests ([leedeternal/dcrd#1588](https://github.com/leedeternal/dcrd/pull/1588))
- blockchain: Use harness in fixed seqlock tests ([leedeternal/dcrd#1589](https://github.com/leedeternal/dcrd/pull/1589))
- multi: cleanup linter warnings ([leedeternal/dcrd#1601](https://github.com/leedeternal/dcrd/pull/1601))
- txscript: Add remove signature reference test ([leedeternal/dcrd#1604](https://github.com/leedeternal/dcrd/pull/1604))
- rpctest: Update for rpccclient/v2 and dcrjson/v2 ([leedeternal/dcrd#1610](https://github.com/leedeternal/dcrd/pull/1610))
- wire: Add tests for MsgCFTypes ([leedeternal/dcrd#1619](https://github.com/leedeternal/dcrd/pull/1619))
- chaincfg: Move a test to chainhash package ([leedeternal/dcrd#1632](https://github.com/leedeternal/dcrd/pull/1632))
- rpctest: Add RemoveNode ([leedeternal/dcrd#1643](https://github.com/leedeternal/dcrd/pull/1643))
- rpctest: Add NodesConnected ([leedeternal/dcrd#1643](https://github.com/leedeternal/dcrd/pull/1643))
- dcrutil: Reduce global refs in addr unit tests ([leedeternal/dcrd#1666](https://github.com/leedeternal/dcrd/pull/1666))
- dcrutil: Consolidate tests into package ([leedeternal/dcrd#1669](https://github.com/leedeternal/dcrd/pull/1669))
- peer: Consolidate tests into package ([leedeternal/dcrd#1670](https://github.com/leedeternal/dcrd/pull/1670))
- wire: Add tests for BlockHeader (From)Bytes ([leedeternal/dcrd#1600](https://github.com/leedeternal/dcrd/pull/1600))
- wire: Add tests for MsgGetCFilter ([leedeternal/dcrd#1628](https://github.com/leedeternal/dcrd/pull/1628))
- dcrutil: Add tests for NewTxDeep ([leedeternal/dcrd#1684](https://github.com/leedeternal/dcrd/pull/1684))
- rpctest: Introduce VotingWallet ([leedeternal/dcrd#1668](https://github.com/leedeternal/dcrd/pull/1668))
- txscript: Add stake tx remove opcode tests ([leedeternal/dcrd#1210](https://github.com/leedeternal/dcrd/pull/1210))
- txscript: Move init func in benchmarks to top ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for script parsing ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for DisasmString ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Convert sighash calc tests ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for IsPayToScriptHash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmarks for IsMutlsigScript ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmarks for IsMutlsigSigScript ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for GetSigOpCount ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add tests for stake-tagged script hash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for isAnyKindOfScriptHash ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for IsPushOnlyScript ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for GetPreciseSigOpCount ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for GetScriptClass ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for pay-to-pubkey scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add bench for pay-to-alt-pubkey scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add bench for pay-to-pubkey-hash scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add bench for pay-to-alt-pubkey-hash scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add bench for null scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add bench for stake submission scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add bench for stake generation scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add bench for stake revocation scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add bench for stake change scripts ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for ContainsStakeOpCodes ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for ExtractCoinbaseNullData ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add CalcMultiSigStats benchmark ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add multisig redeem script extract bench ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for PushedData ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add benchmark for IsUnspendable ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add tests for atomic swap extraction ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add ExtractAtomicSwapDataPushes benches ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add ExtractPkScriptAddrs benchmarks ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- txscript: Add ExtractPkScriptAltSigType benchmark ([leedeternal/dcrd#1656](https://github.com/leedeternal/dcrd/pull/1656))
- wire: Add tests for MsgGetCFTypes ([leedeternal/dcrd#1703](https://github.com/leedeternal/dcrd/pull/1703))
- blockchain: Allow named blocks in chaingen harness ([leedeternal/dcrd#1701](https://github.com/leedeternal/dcrd/pull/1701))
- txscript: Cleanup opcode removal by data tests ([leedeternal/dcrd#1702](https://github.com/leedeternal/dcrd/pull/1702))
- hdkeychain: Correct benchmark extended key ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- hdkeychain: Consolidate tests into package ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- hdkeychain: Use locally-scoped netparams in tests ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- hdkeychain: Use mock net params in tests ([leedeternal/dcrd#1696](https://github.com/leedeternal/dcrd/pull/1696))
- wire: Add tests for MsgGetCFHeaders ([leedeternal/dcrd#1720](https://github.com/leedeternal/dcrd/pull/1720))
- wire: Add tests for MsgCFHeaders ([leedeternal/dcrd#1732](https://github.com/leedeternal/dcrd/pull/1732))
- main/rpctest: Update for hdkeychain/v2 ([leedeternal/dcrd#1707](https://github.com/leedeternal/dcrd/pull/1707))
- rpctest: Allow custom miner on voting wallet ([leedeternal/dcrd#1751](https://github.com/leedeternal/dcrd/pull/1751))
- wire: Add tests for MsgCFilter ([leedeternal/dcrd#1741](https://github.com/leedeternal/dcrd/pull/1741))
- chaincfg; Add tests for required unique fields ([leedeternal/dcrd#1698](https://github.com/leedeternal/dcrd/pull/1698))
- fullblocktests: Add coinbase nulldata tests ([leedeternal/dcrd#1769](https://github.com/leedeternal/dcrd/pull/1769))
- dcrutil: Make docs example testable and correct it ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- dcrutil: Use mock addr params in tests ([leedeternal/dcrd#1767](https://github.com/leedeternal/dcrd/pull/1767))
- wire: assert MaxMessagePayload limit in tests ([leedeternal/dcrd#1755](https://github.com/leedeternal/dcrd/pull/1755))
- docker: use go 1.12 ([leedeternal/dcrd#1782](https://github.com/leedeternal/dcrd/pull/1782))
- docker: update alpine and include notes ([leedeternal/dcrd#1786](https://github.com/leedeternal/dcrd/pull/1786))
- hdkeychain: Correct a few comment typos ([leedeternal/dcrd#1796](https://github.com/leedeternal/dcrd/pull/1796))
- database: Use unique test db names for v2 module ([leedeternal/dcrd#1806](https://github.com/leedeternal/dcrd/pull/1806))
- main: Add database/v2 override for tests ([leedeternal/dcrd#1806](https://github.com/leedeternal/dcrd/pull/1806))
- gcs: Add benchmark for AddSigScript ([leedeternal/dcrd#1804](https://github.com/leedeternal/dcrd/pull/1804))
- txscript: Fix typo in script test data ([leedeternal/dcrd#1821](https://github.com/leedeternal/dcrd/pull/1821))
- database: Separate dbs for concurrent db tests ([leedeternal/dcrd#1824](https://github.com/leedeternal/dcrd/pull/1824))
- gcs: Overhaul tests and benchmarks ([leedeternal/dcrd#1845](https://github.com/leedeternal/dcrd/pull/1845))
- rpctest: Remove leftover debug print ([leedeternal/dcrd#1862](https://github.com/leedeternal/dcrd/pull/1862))
- txscript: Fix duplicate test name ([leedeternal/dcrd#1863](https://github.com/leedeternal/dcrd/pull/1863))
- gcs: Add benchmark for filter hashing ([leedeternal/dcrd#1853](https://github.com/leedeternal/dcrd/pull/1853))
- gcs: Add tests for bit reader/writer ([leedeternal/dcrd#1855](https://github.com/leedeternal/dcrd/pull/1855))
- peer: Ensure listener tests sync with messages ([leedeternal/dcrd#1874](https://github.com/leedeternal/dcrd/pull/1874))
- rpctest: remove always-nil error ([leedeternal/dcrd#1913](https://github.com/leedeternal/dcrd/pull/1913))
- rpctest: use errgroup to catch errors from go routines ([leedeternal/dcrd#1913](https://github.com/leedeternal/dcrd/pull/1913))

### Misc:

- release: Bump for 1.5 release cycle ([leedeternal/dcrd#1546](https://github.com/leedeternal/dcrd/pull/1546))
- mempool: Fix typo in fetchInputUtxos comment ([leedeternal/dcrd#1562](https://github.com/leedeternal/dcrd/pull/1562))
- blockchain: Fix typos found by misspell ([leedeternal/dcrd#1617](https://github.com/leedeternal/dcrd/pull/1617))
- dcrutil: Fix typos found by misspell ([leedeternal/dcrd#1617](https://github.com/leedeternal/dcrd/pull/1617))
- main: Write memprofile on shutdown ([leedeternal/dcrd#1655](https://github.com/leedeternal/dcrd/pull/1655))
- config: Parse network interfaces ([leedeternal/dcrd#1514](https://github.com/leedeternal/dcrd/pull/1514))
- config: Cleanup and simplify network info parsing ([leedeternal/dcrd#1706](https://github.com/leedeternal/dcrd/pull/1706))
- main: Rework windows service sod notification ([leedeternal/dcrd#1710](https://github.com/leedeternal/dcrd/pull/1710))
- multi: fix recent govet findings ([leedeternal/dcrd#1727](https://github.com/leedeternal/dcrd/pull/1727))
- rpcserver: Fix misspelling ([leedeternal/dcrd#1763](https://github.com/leedeternal/dcrd/pull/1763))
- chaincfg: Run gofmt -s ([leedeternal/dcrd#1776](https://github.com/leedeternal/dcrd/pull/1776))
- jsonrpc/types: Update copyright years ([leedeternal/dcrd#1794](https://github.com/leedeternal/dcrd/pull/1794))
- stake: Correct comment typo on Hash256PRNG ([leedeternal/dcrd#1803](https://github.com/leedeternal/dcrd/pull/1803))
- multi: Correct typos ([leedeternal/dcrd#1839](https://github.com/leedeternal/dcrd/pull/1839))
- wire: Fix a few messageError string typos ([leedeternal/dcrd#1840](https://github.com/leedeternal/dcrd/pull/1840))
- miningerror: Remove duplicate copyright ([leedeternal/dcrd#1860](https://github.com/leedeternal/dcrd/pull/1860))
- multi: Correct typos ([leedeternal/dcrd#1864](https://github.com/leedeternal/dcrd/pull/1864))

### Code Contributors (alphabetical order):

- Aaron Campbell
- Conner Fromknecht
- Dave Collins
- David Hill
- Donald Adu-Poku
- Hamid
- J Fixby
- Jamie Holdstock
- JoeGruffins
- Jonathan Chappelow
- Josh Rickmar
- Matheus Degiovani
- Nicola Larosa
- Olaoluwa Osuntokun
- Roei Erez
- Sarlor
- Victor Oliveira
