# Cosmos HD Key Derivation

Cosmos blockchains allow users to utilize Hierarchical deterministic key generation (HD keys) for
deriving multiple cryptographic keypairs from a single secret value. This allows the user to use
different keypair for different accounts on one blockchain or create accounts on different
blockchains without having to secure multiple secrets.

The specifications used to do that are [BIP32], [BIP39], [BIP43] and [BIP44]. They originate from
Bitcoin but are also used in many other blockchain ecosystems. BIP32 specifies HD derivation for the
elliptic curve only but alternative aproaches have been developed outside of the Bitcoin ecosystem
to generalize the same idea to other signing algorithms<sup>1</sup>.

## BIP44

An "HD path" is an instruction how to derive a keypair from a root secret. BIPP44 specifies a schema
for such paths as follows:

```
m / 44' / coin_type' / account' / change / address_index
```

where `m` is a constant symbol that defines the beginning of the path, `/` is the separator of path
components, `44` is the the `purpose` value from BIP43, `'` denotes hardened derivation and the
remaining four symbols are variable names for integers. A BIP44 path always has those five
components.

A `coin_type` registry is maintained in [SLIP44] and the pattern for some example blockchains look
like this.

```
Bitcoin:    m / 44' /   0' /       account' / change / address_index
Litecoin:   m / 44' /   2' /       account' / change / address_index
Ethereum:   m / 44' /  60' /             0' /      0 / address_index
Cosmos Hub: m / 44' / 118' /             0' /      0 / address_index
EOS:        m / 44' / 194' / address_index' /      0 / 0
Monero:     m / 44' / 128' / address_index'
Stellar:    m / 44' / 148' / address_index'
```

As you can see for the blockchains that use an account based model instead of UTXOs, only one path
component is valiable (`address_index`) and the differention between account and address does not
help for those cases. For some chains the unused components are filled with zeros. Others follow the
Stellar's model by simplifying the path to 3 components<sup>2</sup>, which results in paths that do
not comply with BIP44 anymore. This shows how BIP44 is used for history reasons but is not a great
fit for account based blockchains.

## The Cosmos Hub path

The Cosmos Hub HD path is `m / 44' / 118' / 0' / 0 / address_index` where `44` is the BIP44
`purpose`, `118` is the `coin_index` [for ATOM][slip44] and
[the last component](https://github.com/cosmos/cosmos-sdk/issues/4278#issuecomment-561238038) is
used for multiple accounts of the same user. This path has been [used in the Cosmos
fundraiser][fundraiser path] with `address_index = 0`.

Thoughout Cosmos's history the difference between the Cosmos ecosystem and the Cosmos Hub is not
cleanly differentiated in many places. The fundraiser code uses the term "Cosmos addresses" when
talking about addresses on the Cosmos Hub, the
["Cosmos" Ledger App is marketed as the ATOM app](https://support.ledger.com/hc/en-us/articles/360013713840-Cosmos-ATOM-)
and a large number of other references in the ecosystem use cosmos when referring to the Cosmos
Hub<sup>3</sup>. The latest revision of [the Cosmos website](https://cosmos.network) manages to
communicate a strict differentiation between the two. It is unclear if this overall lack of
precision is sloppiness or intentional. But in the context of HP paths it leads to a privacy issue.

### Reuse of the Cosmos Hub path in Cosmos

A good number of blockchains were created that reuse the Cosmos Hub path. A quick serach in the
Keplr configuration reveals that at least Kava, Secret Network, Akash, SifChain, CertiK, IRISnet,
Regen, Sentinel, Cyber and Straightedge used or actively use the ATOM coin index 118. Using the same
derivation path means that users that use one secret recovery phrase to interact with multiple
networks will be signing with the same keypair. Their public identity is the same public key. This
is then obfuscated because [bech32][bip173] addresses with different prefixes are used. Those
addresses look different at first glance, but contain the same data.

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
`pkptre7fdkl6gfrzlesjjvhxhlc3r4gm`, which is the bech32 data that is surrounded by the prefix and a
checksum. We can use any bech32 decoder to see both addresses contain the same data.

```js
import { toHex, Bech32 } from "@cosmjs/encoding";

const dataA = Bech32.decode("achain1pkptre7fdkl6gfrzlesjjvhxhlc3r4gmjufvfw").data;
const dataB = Bech32.decode("bitwhatever1pkptre7fdkl6gfrzlesjjvhxhlc3r4gmtwnu3c").data;

console.log(toHex(dataA)); // 0d82b1e7c96dbfa42462fe612932e6bff111d51b
console.log(toHex(dataB)); // 0d82b1e7c96dbfa42462fe612932e6bff111d51b
```

This knowledge can not be used to create equality sets containing addresses on different chains that
belong to the same user. This is a problem if the user is not aware that this linking is possible
and behaves as if the identities were independent.

## Exploring other purposes

## Notes

<sup>1</sup> e.g. [SLIP10],
[Cardano's Ed25519 derivation](https://docs.cardano.org/projects/cardano-wallet/en/latest/About-Address-Derivation.html)
or
[Parity's sr25519 derivation](https://github.com/paritytech/parity-signer/blob/c6b30f5648f8a1abc6481695d747a9ed15c151d5/docs/tutorials/Hierarchical-Deterministic-Key-Derivation.md)

<sup>2</sup> _If it is account-based we follow Stellar's SEP-0005 - paths have only three parts
44'/c'/a'. Unfortunately, lot of exceptions occur due to compatibility reasons._
[Trezor BIP-44 derivation paths](https://github.com/trezor/trezor-firmware/blob/4e005de0/docs/misc/coins-bip44-paths.md)

<sup>3</sup> Cosmos Hub specific resources: [Mintscan](https://www.mintscan.io/cosmos),
[Big Dipper](https://cosmos.bigdipper.live/), [RPC domain](https://rpc.cosmos.network/),
[Keplr config](https://github.com/chainapsis/keplr-extension/blob/v0.8.8/packages/extension/src/config.ts#L62-L67)

[bip32]: https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki
[bip39]: https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki
[bip43]: https://github.com/bitcoin/bips/blob/master/bip-0043.mediawiki
[bip44]: https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki
[bip173]: https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki
[slip10]: https://github.com/satoshilabs/slips/blob/master/slip-0010.md
[slip44]: https://github.com/satoshilabs/slips/blob/master/slip-0044.md
[fundraiser path]: https://github.com/cosmos/fundraiser-cli/blob/2.11.3/golang/main.go#L90
