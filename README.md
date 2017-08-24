<pre>
  BIP: ????
  Layer: Applications
  Title: ...
  Author: Simone Bronzini <simone.bronzini@chainside.net>
          ...
  comments-URI: ...
  Status: Proposed
  Type: Standards Track
  Created: 2017-08-24
  License: GNU-All-Permissive
</pre>

==Abstract==

This BIP defines a logical istructure for hierarhical deterministic multisig
wallets based on the algorithm described in BIP-0032 (BIP32 from now on) and
purpose scheme described in BIP-0043 (BIP43 from now on). This BIP is a
particular application of BIP43.

==Motivation==

The structure described in this document allows different parties to agree on
a standard way to independently create multisig P2SH addresses between them.
A similar structure has been proposed in the past for such need (BIP-0045)
but that structure has some disadvantages that this BIP aims at solving.

In fact, when used in a multi-account fashion, BIP45 structure presents
the following limits:
* The same keys are reused when multisig addresses are created by different
parties for different accounts, resulting in a loss of privacy. This happens
because BIP45 does not allow to separate different accounts.
* Tracking different accounts that create multisig addresses in this way
can be particularly complex

The other option would be using different master keys for every multisig accounts
but this would mean having to backup many master seeds, hence non fully
exploiting the potentiality of having an HD structure (i.e. having to backup
only one secret for many purposes/accounts)

==Specification==

The proposed structure is as follows:
<pre>
m / purpose' / coin_type' / account' / cosigner_index / change / address_index
</pre>

Apostrophe in the path indicates that hardened derivation described in BIP32 is used.

Each level has special meaning described in the following chapters.

===Purpose===

Purpose is a constant set to [whatever number this BIP will be assigned]',
following the BIP43 recommendation.

<pre>
m / ????' / *
</pre>

It indicates that the subtree of this node is used according to this specification.

Hardened derivation is used at this level.

===Coin type===

Due to the generic structure of this proposal, the keys generated following this BIP,
can be used on different cryptocurrencies.

As described in BIP44, this level specifies the cryptocurrency for which the keys
of this subtree are used. The list of already allocated coin types is in the chapter
"Registered coin types" below.

Hardened derivation is used at this level.

===Account===

This level splits the key space into independent multisig accounts, so the wallet never
mixes the coins across different accounts.
For instance, if user A wants to have a 2-of-2 multisig account with user B and also a 2-of-3
multisig account with users B and C, he/she will create two different subtrees on this level
for the two accounts. This is the key the parties should exchange among them to allow other
create shared addresses, as described in the "Address generation" section.

Software should prevent a creation of an account if a previous account does not have a
transaction history (meaning none of its addresses have been used before).

Software needs to discover all used accounts after importing the seed from an external
source and the account roots of the cosigners for that account.
Such an algorithm is described in "Address discovery" chapter.

Hardened derivation is used at this level.

===Cosigner index===

As described in BIP45, this is the index of the party creating a P2SH multisig address.
The indices can be determined independently by lexicographically sorting the account public
keys of each cosigner. Each cosigner creates addresses on it's own branch, even though
they have independent extended master public key, as explained in the "Address generation"
section.

Note that the master public key is not shared amongst the cosigners. Only the hardened
account public key is shared, and this is what is used to derive child extended public keys.

Software should only use indices corresponding to each of the N cosigners sequentially.
For example, for a 2-of-3 HDPM wallet, having the following purpose public keys:

Software needs to discover all used indices when importing the seed from an external source.
Such algorithm is described in the "Address discovery" chapter.

===Change===

Constant 0 is used for external chain and constant 1 for internal chain (also known as
change addresses). External chain is used for addresses that are meant to be visible outside
of the wallet (e.g. for receiving payments). Internal chain is used for addresses which
are not meant to be visible outside of the wallet and is used for return transaction change.

Having a change subtree also helps when more than 20 non-change addresses are generated
but have not received transactions yet, this will always allow to be able to generate new
addresses to receive change, without exceeding the gap limit. This will be further
clarified in the "Address discovery" section.

===Address index===

This is the level where actual addresses are generated. Addresses are numbered from index
0 in sequentially increasing manner.

===Address generation===

As described in BIP45, on account setup, each party has their own extended master
keypair, derives a new account for the multisig wallet from the account' level and shares
the extended account' public key with the others, which is stored encrypted. Each party
can generate any of the other's derived public keys, but only his own private keys.

When generating an address, each party can independently generate the N needed public
keys. They do this by deriving the public key in each of the different trees, but using
the same path. They can then generate the multisig script (by lexicographically sorting 
the public keys) and the corresponding P2SH address. In this way, each path corresponds
to an address, but the public keys for that address come from different trees.

====Receive address case====

Each cosigner generates addresses only on his own branch. One of the N
cosigners wants to receive a payment, and the others are offline. He
knows the last used index in his own branch, because only he generates
addresses there. Thus, he can generate the public keys for all of the
others using the next index, and calculate the needed script for the address.

Example: 
Cosigner 2 has a multisig account with cosigners 1 and 3. He/she has account number X
on cosigner 1's tree, account number Y on his/her own tree and account number Z on
cosigner 3's tree. Cosigner 2 wants to receive a payment to the shared wallet. His last
used index on his own branch is 4. Then, the paths for the next receive
address are:

<pre>
m/????'/BTC'/X'/0/5
</pre>
on cosigner 1's tree,
<pre>
m/????'/BTC'/Y'/0/5
</pre>
on his/her own tree, and
<pre>
m/????'/BTC'/Z'/0/5
</pre>
on cosigner 3's tree.

He uses these paths to generate a public key for each cosigner, and from that he gets the new
P2SH address.

====Change address case====

Again, each cosigner generates addresses only on his own branch. One of the
N cosigners wants to create an outgoing payment, for which he'll need a change
address. He generates a new address using the same procedure as above, but
using a separate index to track the used change addresses.

The procedure is the same as described in the preceeding section but index 1 will be used
for the change branch instead of 0.

===Transaction signing===

The signing of transactions can be done exactly as described in BIP45.

===Address discovery===

As similarly described in BIP45, when the master seed and the account roots
of the cosigners are imported from an external source,
the software should start to discover the addresses in the following manner:

# for each cosigner:
## derive the cosigner's node (m / ????' / BTC' / cosigner's account root' / cosigner_index`)
## for both the external and internal chains on this node (m / ????' / BTC' / cosigner's account root / cosigner_index / 0 and m / ????' / BTC' / cosigner's account root / cosigner_index / 0):
### scan addresses of the chain; respect the gap limit described below

Please note that the algorithm uses the transaction history, not address balances,
so even if the address has 0 coins, the program should continue with discovery.
Opposite to BIP44, as described in BIP45, each cosigner branch needs to be checked,
even if the earlier ones don't have transactions.

==Address gap limit==

Address gap limit is currently set to 20. If the software hits 20 unused 
addresses (no transactions associated with that address) in a row, it expects
there are no used addresses beyond this point and stops searching the address chain.

Wallet software should warn when user is trying to exceed the gap limit on
an external chain by generating a new address.

==Registered coin types==

...

==Reference==

* [[bip-0032.mediawiki|BIP32 - Hierarchical Deterministic Wallets]]
* [[bip-0043.mediawiki|BIP43 - Purpose Field for Deterministic Wallets]]
* [[bip-0044.mediawiki|BIP32 - Multi-Account Hierarchy for Deterministic Wallets]]
* [[bip-0045.mediawiki|BIP43 - Structure for Deterministic P2SH Multisignature Wallets]]
