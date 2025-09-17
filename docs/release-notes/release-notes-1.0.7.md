## dcrd v1.0.7

This release of dcrd primarily contains improvements to the infrastructure and
other quality assurance changes that are bringing us closer to providing full
support for Lightning Network.

A lot of work required for Lightning Network support went into getting the
required code merged into the upstream project, btcd, which now fully supports
it.  These changes also must be synced and integrated with dcrd as well and
therefore many of the changes in this release are related to that process.

## Notable Changes

### Dust check removed from stake transactions

The standard policy for regular transactions is to reject any transactions that
have outputs so small that they cost more to the network than their value.  This
behavior is desirable for regular transactions, however it was also being
applied to vote and revocation transactions which could lead to a situation
where stake pools with low fees could result in votes and revocations having
difficulty being mined.

This check has been changed to only apply to regular transactions now in order
to prevent any issues.  Stake transactions have several other checks that make
this one unnecessary for them.

### New `feefilter` peer-to-peer message

A new optional peer-to-peer message named `feefilter` has been added that allows
peers to inform others about the minimum transaction fee rate they are willing
to accept.  This will enable peers to avoid notifying others about transactions
they will not accept anyways and therefore can result in a significant bandwidth
savings.

### Bloom filter service bit enforcement

Peers that are configured to disable bloom filter support will now disconnect
remote peers that send bloom filter related commands rather than simply ignoring
them.  This allows any light clients that do not observe the service bit to
potentially find another peer that provides the service.  Additionally, remote
peers that have negotiated a high enough protocol version to observe the service
bit and still send bloom filter related commands anyways will now be banned.


## Changelog

All commits since the last release may be viewed on GitHub [here](https://github.com/leedeternal/dcrd/compare/v1.0.5...v1.0.7).

### Protocol and network:
- Allow reorg of block one [leedeternal/dcrd#745](https://github.com/leedeternal/dcrd/pull/745)
- blockchain: use the time source [leedeternal/dcrd#747](https://github.com/leedeternal/dcrd/pull/747)
- peer: Strictly enforce bloom filter service bit [leedeternal/dcrd#768](https://github.com/leedeternal/dcrd/pull/768)
- wire/peer: Implement feefilter p2p message [leedeternal/dcrd#779](https://github.com/leedeternal/dcrd/pull/779)
- chaincfg: update checkpoints for 1.0.7 release  [leedeternal/dcrd#816](https://github.com/leedeternal/dcrd/pull/816)

### Transaction relay (memory pool):
- mempool: Break dependency on chain instance [leedeternal/dcrd#754](https://github.com/leedeternal/dcrd/pull/754)
- mempool: unexport the mutex [leedeternal/dcrd#755](https://github.com/leedeternal/dcrd/pull/755)
- mempool: Add basic test harness infrastructure [leedeternal/dcrd#756](https://github.com/leedeternal/dcrd/pull/756)
- mempool: Improve tx input standard checks [leedeternal/dcrd#758](https://github.com/leedeternal/dcrd/pull/758)
- mempool: Update comments for dust calcs [leedeternal/dcrd#764](https://github.com/leedeternal/dcrd/pull/764)
- mempool: Only perform standard dust checks on regular transactions  [leedeternal/dcrd#806](https://github.com/leedeternal/dcrd/pull/806)

### RPC:
- Fix gettxout includemempool handling [leedeternal/dcrd#738](https://github.com/leedeternal/dcrd/pull/738)
- Improve help text for getmininginfo [leedeternal/dcrd#748](https://github.com/leedeternal/dcrd/pull/748)
- rpcserverhelp: update TicketFeeInfo help [leedeternal/dcrd#801](https://github.com/leedeternal/dcrd/pull/801)
- blockchain: Improve getstakeversions efficiency [leedeternal/dcrd#81](https://github.com/leedeternal/dcrd/pull/813)

### dcrd command-line flags:
- config: introduce new flags to accept/reject non-std transactions [leedeternal/dcrd#757](https://github.com/leedeternal/dcrd/pull/757)
- config: Add --whitelist option [leedeternal/dcrd#352](https://github.com/leedeternal/dcrd/pull/352)
- config: Improve config file handling [leedeternal/dcrd#802](https://github.com/leedeternal/dcrd/pull/802)
- config: Improve blockmaxsize check [leedeternal/dcrd#810](https://github.com/leedeternal/dcrd/pull/810)

### dcrctl:
- Add --walletrpcserver option [leedeternal/dcrd#736](https://github.com/leedeternal/dcrd/pull/736)

### Documentation
- docs: add commit prefix notes  [leedeternal/dcrd#788](https://github.com/leedeternal/dcrd/pull/788)

### Developer-related package changes:
- blockchain: check errors and remove ineffectual assignments [leedeternal/dcrd#689](https://github.com/leedeternal/dcrd/pull/689)
- stake: less casting [leedeternal/dcrd#705](https://github.com/leedeternal/dcrd/pull/705)
- blockchain: chainstate only needs prev block hash [leedeternal/dcrd#706](https://github.com/leedeternal/dcrd/pull/706)
- remove dead code [leedeternal/dcrd#715](https://github.com/leedeternal/dcrd/pull/715)
- Use btclog for determining valid log levels [leedeternal/dcrd#738](https://github.com/leedeternal/dcrd/pull/738)
- indexers: Minimize differences with upstream code [leedeternal/dcrd#742](https://github.com/leedeternal/dcrd/pull/742)
- blockchain: Add median time to state snapshot [leedeternal/dcrd#753](https://github.com/leedeternal/dcrd/pull/753)
- blockmanager: remove unused GetBlockFromHash function [leedeternal/dcrd#761](https://github.com/leedeternal/dcrd/pull/761)
- mining: call CheckConnectBlock directly [leedeternal/dcrd#762](https://github.com/leedeternal/dcrd/pull/762)
- blockchain: add missing error code entries [leedeternal/dcrd#763](https://github.com/leedeternal/dcrd/pull/763)
- blockchain: Sync main chain flag on ProcessBlock [leedeternal/dcrd#767](https://github.com/leedeternal/dcrd/pull/767)
- blockchain: Remove exported CalcPastTimeMedian func [leedeternal/dcrd#770](https://github.com/leedeternal/dcrd/pull/770)
- blockchain: check for error [leedeternal/dcrd#772](https://github.com/leedeternal/dcrd/pull/772)
- multi: Optimize by removing defers [leedeternal/dcrd#782](https://github.com/leedeternal/dcrd/pull/782)
- blockmanager: remove unused logBlockHeight [leedeternal/dcrd#787](https://github.com/leedeternal/dcrd/pull/787)
- dcrutil: Replace DecodeNetworkAddress with DecodeAddress [leedeternal/dcrd#746](https://github.com/leedeternal/dcrd/pull/746)
- txscript: Force extracted addrs to compressed [leedeternal/dcrd#775](https://github.com/leedeternal/dcrd/pull/775)
- wire: Remove legacy transaction decoding [leedeternal/dcrd#794](https://github.com/leedeternal/dcrd/pull/794)
- wire: Remove dead legacy tx decoding code [leedeternal/dcrd#796](https://github.com/leedeternal/dcrd/pull/796)
- mempool/wire: Don't make policy decisions in wire [leedeternal/dcrd#797](https://github.com/leedeternal/dcrd/pull/797)
- dcrjson: Remove unused cmds & types [leedeternal/dcrd#795](https://github.com/leedeternal/dcrd/pull/795)
- dcrjson: move cmd types [leedeternal/dcrd#799](https://github.com/leedeternal/dcrd/pull/799)
- multi: Separate tx serialization type from version [leedeternal/dcrd#798](https://github.com/leedeternal/dcrd/pull/798)
- dcrjson: add Unconfirmed field to dcrjson.GetAccountBalanceResult [leedeternal/dcrd#812](https://github.com/leedeternal/dcrd/pull/812)
- multi: Error descriptions should be lowercase [leedeternal/dcrd#771](https://github.com/leedeternal/dcrd/pull/771)
- blockchain: cast to int64  [leedeternal/dcrd#817](https://github.com/leedeternal/dcrd/pull/817)

### Testing and Quality Assurance:
- rpcserver: Upstream sync to add basic RPC tests [leedeternal/dcrd#750](https://github.com/leedeternal/dcrd/pull/750)
- rpctest: Correct several issues tests and joins [leedeternal/dcrd#751](https://github.com/leedeternal/dcrd/pull/751)
- rpctest: prevent process leak due to panics [leedeternal/dcrd#752](https://github.com/leedeternal/dcrd/pull/752)
- rpctest: Cleanup resources on failed setup [leedeternal/dcrd#759](https://github.com/leedeternal/dcrd/pull/759)
- rpctest: Use ports based on the process id [leedeternal/dcrd#760](https://github.com/leedeternal/dcrd/pull/760)
- rpctest/deps: Update dependencies and API [leedeternal/dcrd#765](https://github.com/leedeternal/dcrd/pull/765)
- rpctest: Gate rpctest-based behind a build tag [leedeternal/dcrd#766](https://github.com/leedeternal/dcrd/pull/766)
- mempool: Add test for max orphan entry eviction [leedeternal/dcrd#769](https://github.com/leedeternal/dcrd/pull/769)
- fullblocktests: Add more consensus tests [leedeternal/dcrd#77](https://github.com/leedeternal/dcrd/pull/773)
- fullblocktests: Sync upstream block validation [leedeternal/dcrd#774](https://github.com/leedeternal/dcrd/pull/774)
- rpctest: fix a harness range bug in syncMempools [leedeternal/dcrd#778](https://github.com/leedeternal/dcrd/pull/778)
- secp256k1: Add regression tests for field.go [leedeternal/dcrd#781](https://github.com/leedeternal/dcrd/pull/781)
- secp256k1: Sync upstream test consolidation [leedeternal/dcrd#783](https://github.com/leedeternal/dcrd/pull/783)
- txscript: Correct p2sh hashes in json test data  [leedeternal/dcrd#785](https://github.com/leedeternal/dcrd/pull/785)
- txscript: Replace CODESEPARATOR json test data [leedeternal/dcrd#786](https://github.com/leedeternal/dcrd/pull/786)
- txscript: Remove multisigdummy from json test data [leedeternal/dcrd#789](https://github.com/leedeternal/dcrd/pull/789)
- txscript: Remove max money from json test data [leedeternal/dcrd#790](https://github.com/leedeternal/dcrd/pull/790)
- txscript: Update signatures in json test data [leedeternal/dcrd#791](https://github.com/leedeternal/dcrd/pull/791)
- txscript: Use native encoding in json test data [leedeternal/dcrd#792](https://github.com/leedeternal/dcrd/pull/792)
- rpctest: Store logs and data in same path [leedeternal/dcrd#780](https://github.com/leedeternal/dcrd/pull/780)
- txscript: Cleanup reference test code  [leedeternal/dcrd#793](https://github.com/leedeternal/dcrd/pull/793)

### Misc:
- Update deps to pull in additional logging changes [leedeternal/dcrd#734](https://github.com/leedeternal/dcrd/pull/734)
- Update markdown files for GFM changes [leedeternal/dcrd#744](https://github.com/leedeternal/dcrd/pull/744)
- blocklogger: Show votes, tickets, & revocations [leedeternal/dcrd#784](https://github.com/leedeternal/dcrd/pull/784)
- blocklogger: Remove STransactions from transactions calculation [leedeternal/dcrd#811](https://github.com/leedeternal/dcrd/pull/811)

### Contributors (alphabetical order):

- Alex Yocomm-Piatt
- Atri Viss
- Chris Martin
- Dave Collins
- David Hill
- Donald Adu-Poku
- Jimmy Song
- John C. Vernaleo
- Jolan Luff
- Josh Rickmar
- Olaoluwa Osuntokun
- Marco Peereboom
