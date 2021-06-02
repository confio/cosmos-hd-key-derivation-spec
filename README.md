# Cosmos HD key derivation

Cosmos blockchains support hierarchical deterministic key generation (HD keys) for
deriving multiple cryptographic keypairs from a single secret value. This allows the user
to use different keypairs for different accounts on one blockchain and create accounts on
multiple blockchains without having to manage multiple secrets.

The technology to do that is specified in [BIP32], [BIP39], [BIP43] and [BIP44]. Those
specs originate from Bitcoin but are also used in many other blockchain ecosystems. BIP32
specifies HD derivation for the elliptic curve secp256k1 only but alternative aproaches
have been developed outside of the Bitcoin ecosystem to generalize the same idea to other
signing algorithms<sup>1</sup>.

**Table of contents**

<!--ts-->

- [Cosmos HD key derivation](#cosmos-hd-key-derivation)
  - [BIP44](#bip44)
  - [The Cosmos Hub path](#the-cosmos-hub-path)
    - [Reuse of the Cosmos Hub path in Cosmos](#reuse-of-the-cosmos-hub-path-in-cosmos)
  - [The Cosmos Ledger app](#the-cosmos-ledger-app)
  - [Goals](#goals)
  - [The Cosmos purpose](#the-cosmos-purpose)
  - [The chain index](#the-chain-index)
  - [Chain specific path structure](#chain-specific-path-structure)
    - [Cosmos simple HD path](#cosmos-simple-hd-path)
  - [Implementation](#implementation)
  - [Migration](#migration)
  - [Other signing algorithms](#other-signing-algorithms)
  - [Notes](#notes)

<!-- Added by: simon, at: Mi  2 Jun 2021 15:24:01 CEST -->

<!--te-->

## BIP44

An "HD path" is an instruction how to derive a keypair from a root secret. BIPP44
specifies a schema for such paths as follows:

```
m / 44' / coin_type' / account' / change / address_index
```

where `m` is a constant symbol a the beginning of the path, `/` is the separator of path
components, `44` is the the `purpose` value from BIP43, `'` denotes hardened derivation
and the remaining four symbols are variable names for integers. A BIP44 path always has
those five components.

A `coin_type` registry is maintained in [SLIP44] and the pattern for some example
blockchains look like this.

```
Bitcoin:    m / 44' /   0' /       account' / change / address_index
Litecoin:   m / 44' /   2' /       account' / change / address_index
Ethereum:   m / 44' /  60' /             0' /      0 / address_index
Cosmos Hub: m / 44' / 118' /             0' /      0 / address_index
EOS:        m / 44' / 194' / address_index' /      0 / 0
Monero:     m / 44' / 128' / address_index'
Stellar:    m / 44' / 148' / address_index'
```

As you can see for the blockchains that use an account based model instead of UTXOs, only
one path component is valiable (`address_index`) and the differention between account and
address does not help for those cases. For some chains the unused components are filled
with zeros. Others follow the Stellar's model by simplifying the path to 3
components<sup>2</sup>, which results in paths that do not comply with BIP44 anymore. This
shows how BIP44 is used for history reasons but is not a great fit for account based
blockchains. This concern is expressed in [EIP600] as follows: _Because Ethereum is based
on account balances rather than UTXO, the hierarchy defined by BIP44 is poorly suited._

## The Cosmos Hub path

The Cosmos Hub HD path is `m / 44' / 118' / 0' / 0 / address_index` where `44` is the
BIP44 `purpose`, `118` is the `coin_index` [for ATOM][slip44] and
[the last component](https://github.com/cosmos/cosmos-sdk/issues/4278#issuecomment-561238038)
is used for multiple accounts of the same user. This path has been [used in the Cosmos
fundraiser][fundraiser path] with `address_index = 0`.

Thoughout Cosmos's history the difference between the Cosmos ecosystem and the Cosmos Hub
is not cleanly differentiated in many places. The fundraiser code uses the term "Cosmos
addresses" when talking about addresses on the Cosmos Hub, the
["Cosmos" Ledger App is marketed as the ATOM app](https://support.ledger.com/hc/en-us/articles/360013713840-Cosmos-ATOM-)
and a large number of other references in the ecosystem use "cosmos" when referring to the
Cosmos Hub<sup>3</sup>. The latest revision of
[the Cosmos website](https://cosmos.network) manages to communicate a strict
differentiation between the two. It is unclear if this overall lack of precision is
sloppiness or intentional but in the context of HP paths it leads to a privacy issue.

### Reuse of the Cosmos Hub path in Cosmos

A good number of blockchains were created that reuse the Cosmos Hub path. A quick search
in the [Keplr configuration] reveals that at least Kava, Secret Network, Akash, SifChain,
CertiK, IRISnet, Regen, Sentinel, Cyber and Straightedge used or actively use the ATOM
coin index 118. Using the same derivation path means that users that use one secret
recovery phrase to interact with multiple networks will be signing with the same keypair.
Their public identity is the same public key. This is then obfuscated because
[bech32][bip173] addresses with different prefixes are used. Those addresses look
different at first glance, but contain the same data.

```js
import { pubkeyToAddress } from "@cosmjs/amino";

const pubkey = {
  type: "tendermint/PubKeySecp256k1",
  value: "A08EGB7ro1ORuFhjOnZcSgwYlpe0DSFjVNUIkNNQxwKQ",
};

const addressChainA = pubkeyToAddress(pubkey, "achain");
const addressChainB = pubkeyToAddress(pubkey, "bitwhatever");
console.log(addressChainA); // achain1pkptre7fdkl6gfrzlesjjvhxhlc3r4gmjufvfw
console.log(addressChainB); // bitwhatever1pkptre7fdkl6gfrzlesjjvhxhlc3r4gmtwnu3c
```

An untrained eye will not easily see that both addresses share a common middle part
`pkptre7fdkl6gfrzlesjjvhxhlc3r4gm`, which is the bech32 data that is surrounded by the
prefix and a checksum. We can use any bech32 decoder to see both addresses contain the
same data.

```js
import { toHex, Bech32 } from "@cosmjs/encoding";

const dataA = Bech32.decode("achain1pkptre7fdkl6gfrzlesjjvhxhlc3r4gmjufvfw").data;
const dataB = Bech32.decode("bitwhatever1pkptre7fdkl6gfrzlesjjvhxhlc3r4gmtwnu3c").data;

console.log(toHex(dataA)); // 0d82b1e7c96dbfa42462fe612932e6bff111d51b
console.log(toHex(dataB)); // 0d82b1e7c96dbfa42462fe612932e6bff111d51b
```

This knowledge can now be used to create equality sets containing addresses on different
chains that belong to the same user. This is a problem if the user is not aware that such
a linking is possible and behaves as if the identities were independent.

Reusing the Cosmos Hub path also has advantages. It allows chains to use existing client
key management tooling made for the Cosmos Hub. This includes the CLI application coming
with the Cosmos SDK, client libraries like e.g. CosmJS or the Cosmos Ledger app.

## The Cosmos Ledger app

This Cosmos app for the [Ledger hardware wallet][ledger] is in a bit of an identity
crisis. One the one hand it is marketed as the app for the ATOM token. On the other hand
it is called "Cosmos", not "Cosmos Hub". The app reqires the use of a HD path starting
with `m / 44' / 118'`, i.e. the ATOM coin index.

A few Cosmos projects decided to create their own Ledger apps, such as
[Binance][ledgerapp-binance], [Terra][ledgerapp-terra] or [Starname][ledgerapp-starname].
This allows them to fully customize the app but comes with significant maintenance cost.

It has been expressed by various Cosmos community members that the goal is to be able to
use one Ledger app for all Cosmos blockchains. This Cosmos SDK team
[is](https://github.com/cosmos/cosmos-sdk/issues/6078)
[working](https://github.com/cosmos/cosmos-sdk/issues/6513)
[hard](https://github.com/cosmos/cosmos-sdk/issues/9320) to make transaction signing for
arbitrary message types sufficiently self-describing to allow this to happen. This is in
line with the vision to spawn
[a million blockchains](https://www.youtube.com/watch?v=DWVPTYOrUUo) in the Cosmos
ecosystem, which requires blockchain projects to be able to focus on their chain specific
product instead of having to reinvent basic infrastructure components.

While the Ledger app is a prominent and important example, most of the above holds for any
key management and signing solution for Cosmos.

## Goals

Until now we went through the status quo, describing various shortcomings that we have
today due to historic reasons. A modern HD path schema for Cosmos' multi-chain world
should:

1. Ensure users do not accidentally use the same public key on multiple blockchains.
2. Avoid unnecessary path components originating from Bitcoin.
3. Allow the use of a single Ledger app for many blockchains without having to update the
   app for new chains.

The rest of this document will describe a solution that achieves those goals.

## The Cosmos purpose

As described above, using the 5 component path of BIP44 is not suitable. Luckily we can
choose a new path format and remain fully BIP32 and BIP43 compliant. BIP43 defines a HD
path as `m / purpose' / *` with an integer purpose. The asterisk denotes an arbitrary sub
path.

Few different `purpose` values are used in the wild and there is no registry listing them.
The ones found at the time of writing are:

| BIP43 `purpose` | Use case                | Reference                                                                                          |
| --------------- | ----------------------- | -------------------------------------------------------------------------------------------------- |
| 0               | BIP32 default account   | _Note that `m/0'/*` is already taken by BIP32 (default account), which preceded this BIP._ [BIP43] |
| 13              | SLIP13                  | [SLIP13]                                                                                           |
| 43              | Potentially reserved    | https://github.com/bitcoin/bips/pull/523, [EIP600]                                                 |
| 44              | BIP44                   | [BIP44]                                                                                            |
| 45              | BIP45                   | [BIP45]                                                                                            |
| 48              | Graphene-based networks | [SLIP48]                                                                                           |
| 49              | BIP49                   | [BIP49]                                                                                            |
| 80              | BIP80                   | [BIP80]                                                                                            |
| 84              | BIP84                   | [BIP84]                                                                                            |
| 1852            | HD Sequential wallets   | [Cardano: About Address Derivation][cardano-derivation]                                            |
| 10001–19999     | Potentially reserved    | https://github.com/bitcoin/bips/pull/523                                                           |
| 4673607         | Ledger GPG/SSH          | [openpgp-card-app-deterministic-key-derivation]                                                    |

Looking at those numbers we see a lot of direct mappings between the document names
`BIP-XXXX` or `SLIP-XXXX` and the purpose `XXXX`. In order to avoid unnecessary conflics
we stay aways from the four decimals decimals as well as the range reserved for SLIPs and
look for a number greater than or equal 20000. In particular, every value `int(ASCII(s))`
with a 3 or 4 character string `s` would work. So let's just use
`int(ASCII("sky")) = 7564153` as the entry point to the cosmos.

## The chain index

Cosmos paths then have the form `m / 7564153' / *` and we want to ensure that different
chains don't step on each others' toes. So the second path component becomes a chain
index: `m / 7564153' / chain_index' / *`.

This chain index is very similar to the `coin_type` from BIP44/SLIP44 but uses a different
name to make clear that there can be any number of coins in a given chain. Other than the
chain vs. coin confusion, the registry in SLIP44 works pretty well to coordinate the
assignment of indices across the blockchain ecosystem. The same is envisioned for Cosmos
as well. The 2^31 possible values allow registering approximately 2 billion chains.

Chain index 0 is reserved for testing purposes and chain index 1 is reserved for the
Cosmos Hub because Cosmos Hub investors pay the bill for this work. All other indices
should be coordinated in a dedicated place.

## Chain specific path structure

The Cosmos purpose and the chain index together serve as a namespace. Once a chain
registered its chain index, the whole subtree `*` in `m / 7564153' / chain_index' / *` can
be chosen freely. This allows very advanced path setup like the ones described in e.g.
[EIP1581] and [EIP1775]. However, to keep simple things simple, we propose the following
simple HD path format.

### Cosmos simple HD path

In account based blockchains the most widely used path format can be reduces to
`(chain, a)` where chain identifies the blockchain and `a` is a 0-based account index.
Above we saw different implementations of that like `m/44'/118'/0'/0/a` or
`m/44'/148'/a'`. If you have something as simple as that, wallets can easily perform
account discovery, similar to what is described in BIP44. We construct a simple HD path
compliant to the Cosmos purpose and chain index as the following 4 component path:

```
m / 7564153' / chain_index' / 1' / a
```

where `a` is a 0-based account index. We use 4 components instead of 3 to allow chains to
use the simple HD path format as well as other formats in parallel. `1'` was picked
arbitrarily. The 4th component is non-hardened to allow public derivation as described in
BIP32.

## Implementation

TODO: do

## Migration

TODO: do

## Other signing algorithms

Supporting HD derivation for algorithms other than BIP32 (secp256k1 only) is out of scope
for this document. However, here are some starting points for future work:

- SLIP10 is a generalization of BIP32 for other elliptic curve signing algorithms,
  particularly Ed25519. It does not support non-hardened derivations and is not commonly
  used in Cosmos. The Comsos purpose and the chain index could be used with SLIP10
  directly. Cosmos simple HD path however does not work unmodified because it requires
  non-hardened derivation.
- [Cardano documents][cardano-derivation] two different derivation algorithms: "ed25519@V1
  (buggy)" and "ed25519@V2" which seem to use the same path component format as BIP32. To
  what degree this is usable in Cosmos is unknown to the author.
- Parity uses a
  [HD Key Derivation](https://substrate.dev/docs/en/knowledgebase/integrate/subkey#hd-key-derivation)
  scheme with `/` and `//` as separators and string path components that works for
  sr25519.

## Notes

<sup>1</sup> e.g. [SLIP10],
[Cardano's Ed25519 derivation](https://docs.cardano.org/projects/cardano-wallet/en/latest/About-Address-Derivation.html)
or
[Parity's sr25519 derivation](https://github.com/paritytech/parity-signer/blob/c6b30f5648f8a1abc6481695d747a9ed15c151d5/docs/tutorials/Hierarchical-Deterministic-Key-Derivation.md)

<sup>2</sup> _If it is account-based we follow Stellar's SEP-0005 - paths have only three
parts 44'/c'/a'. Unfortunately, lot of exceptions occur due to compatibility reasons._
[Trezor BIP-44 derivation paths](https://github.com/trezor/trezor-firmware/blob/4e005de0/docs/misc/coins-bip44-paths.md)

<sup>3</sup> Cosmos Hub specific resources: [Mintscan](https://www.mintscan.io/cosmos),
[Big Dipper](https://cosmos.bigdipper.live/), [RPC domain](https://rpc.cosmos.network/),
[Keplr config](https://github.com/chainapsis/keplr-extension/blob/v0.8.8/packages/extension/src/config.ts#L62-L67)

<!-- End of document. Links below are not rendered. -->

[bip32]: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
[bip39]: https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki
[bip43]: https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki
[bip44]: https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
[bip45]: https://github.com/bitcoin/bips/blob/master/bip-0045.mediawiki
[bip49]: https://github.com/bitcoin/bips/blob/master/bip-0049.mediawiki
[bip80]: https://github.com/bitcoin/bips/blob/master/bip-0080.mediawiki
[bip84]: https://github.com/bitcoin/bips/blob/master/bip-0084.mediawiki
[bip173]: https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
[eip600]: https://eips.ethereum.org/EIPS/eip-600
[eip1581]: https://eips.ethereum.org/EIPS/eip-1581
[eip1775]: https://eips.ethereum.org/EIPS/eip-1775
[slip10]: https://github.com/satoshilabs/slips/blob/master/slip-0010.md
[slip13]: https://github.com/satoshilabs/slips/blob/master/slip-0013.md
[slip44]: https://github.com/satoshilabs/slips/blob/master/slip-0044.md
[slip48]: https://github.com/satoshilabs/slips/blob/master/slip-0048.md
[fundraiser path]: https://github.com/cosmos/fundraiser-cli/blob/2.11.3/golang/main.go#L90
[openpgp-card-app-deterministic-key-derivation]:
  https://github.com/LedgerHQ/openpgp-card-app/blob/64662c181f4c906288564cbfadc2db53df4534b0/doc/developper/gpgcard3.0-addon.rst#deterministic-key-derivation
[cardano-derivation]:
  https://docs.cardano.org/projects/cardano-wallet/en/latest/About-Address-Derivation.html
[keplr configuration]:
  https://github.com/chainapsis/keplr-extension/blob/v0.8.8/packages/extension/src/config.ts
[ledger]: https://www.ledger.com/
[ledgerapp-binance]:
  https://support.ledger.com/hc/en-us/articles/360021894733-Binance-Chain-BNB-
[ledgerapp-terra]: https://support.ledger.com/hc/en-us/articles/360017698979-Terra-LUNA-
[ledgerapp-starname]:
  https://support.ledger.com/hc/en-us/articles/360016254900-Starname-IOV-
