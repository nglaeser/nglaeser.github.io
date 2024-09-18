---
layout: single
title:  "Key distribution on blockchains: The case for registration-based encryption"
header: 
  teaser: 
categories: 
  - Papers
tags:
  - rbe
  - xpost
  - a16z crypto
excerpt: false
share: false
---

*Cross-posted from the [a16z crypto blog](https://a16zcrypto.com/posts/article/registration-based-encryption/). There is also a [summary Twitter thread](https://x.com/cryptonoemi/status/1823719451692556704).*

Linking cryptographic keys to identities has been an issue since [the introduction of the technology](https://en.wikipedia.org/wiki/Public-key_cryptography#History). The challenge is providing and maintaining a publicly available and consistent mapping between identities and public keys (to which those identities have the private key). Without such a mapping, messages intended to be secret can go awry — sometimes with disastrous outcomes. This same challenge exists in web3.

Three possible solutions currently exist. The two classic approaches are a *public key directory* and *identity-based encryption*. The third is *registration-based encryption*, which until recently was entirely theoretical. The three approaches offer different tradeoffs among anonymity, interactivity, and efficiency, which might seem clear at first, but the blockchain setting introduces new possibilities and constraints. The goal of this post, then, is to explore this design space and compare these approaches when deployed on a blockchain. When users need transparency and anonymity onchain, a practical RBE scheme is the clear winner — overcoming identity-based encryption's strong trust assumption, but at a cost.

# The three approaches in brief

The classic approach to linking cryptographic keys to identities is a public key infrastructure (PKI), with a **public key directory** at its heart. This approach requires a potential sender to interact with a trusted third party (the maintainer of the directory, or certificate authority) to send messages. Additionally, in the web2 setting, maintaining this directory can be costly, tedious, and [error prone](https://sslmate.com/resources/certificate_authority_failures), and users run the risk of [abuse by the certificate authority](https://www.bleepingcomputer.com/news/security/russia-creates-its-own-tls-certificate-authority-to-bypass-sanctions/). 

Cryptographers have suggested alternatives to circumvent the problems with PKIs. In 1984, [Adi Shamir suggested](https://link.springer.com/content/pdf/10.1007/3-540-39568-7_5.pdf) **identity-based encryption** (IBE), in which a party's identifier — phone number, email, ENS domain name — serves as the public key. This eliminates the need to maintain a public key directory but comes at the cost of introducing a trusted third party (the key generator). Dan Boneh and Matthew Franklin gave the [first practical IBE construction](https://eprint.iacr.org/2001/090) in 2001, but IBE has not received widespread adoption except in closed ecosystems such as corporate or [government deployments](https://www.ncsc.gov.uk/guidance/mikey-sakke-frequently-asked-questions), perhaps due to the strong trust assumption. (As we'll see below, this assumption can be partially mitigated by relying on a trusted quorum of parties instead, which a blockchain can easily facilitate.)

A third option, **registration-based encryption** (RBE), was [proposed](https://eprint.iacr.org/2018/919) in 2018. RBE maintains the same general architecture as IBE but replaces the trusted key generator with a *transparent* "key curator" that only performs computations on public data (specifically, it accumulates public keys into a sort of publicly available "digest"). The transparency of this key curator makes the blockchain setting — where a smart contract can fill the role of the curator — a natural fit for RBE. RBE was theoretical until 2022, when my co-authors and I introduced the [first practically efficient RBE construction](https://eprint.iacr.org/2022/1505).

# Evaluating the tradeoffs

This design space is complex, and comparing these schemes requires taking many dimensions into consideration. To keep things simpler, I'll make two assumptions:

1. Users don't update or revoke their keys.
1. The smart contract doesn't use any off-chain data availability service (DAS) or blob data.

I'll discuss at the end how each of these additional considerations could affect our three potential solutions.

![tradeoffs]()

# Public Key Directory

**Summary**: Anyone can add an (id, pk) entry to an on-chain table (the directory) by calling the smart contract, provided the ID hasn't been claimed yet.

![pkd]()

A decentralized PKI is essentially a smart contract that maintains a directory of identities and their corresponding public keys. For example, the [Ethereum Name Service (ENS) Registry](https://docs.ens.domains/) maintains a mapping of domain names ("identities") and their corresponding metadata, including the addresses they resolve to (from whose transactions a public key can be derived). A decentralized PKI would provide even simpler functionality: the list maintained by the contract would store only the public key corresponding to each ID. 

To register, a user generates a key-pair (or uses a previously generated key-pair) and sends its ID and public key to the contract (perhaps along with some payment). The contract checks that the ID hasn't been claimed yet, and, if not, it adds the ID and the public key to the directory. At this point, anyone can encrypt a message to a registered user by asking the contract for the public key corresponding to an ID. (If the sender has encrypted something to this user before, it doesn't have to ask the contract again.) With the public key, the sender can encrypt its message as usual and send the ciphertext to the recipient, who can use the corresponding secret key to recover the message.

Let's take a look at the properties of this method. 

| Positives                  | Negatives             
|:---------------------------|:---------------------------------
|                            | Not succinct
| (Somewhat) interactive encryption |
| Non-interactive decryption | 
| Transparent                | 
|                            | Not sender-anonymous
| Arbitrary IDs              | 

On the negative side of the ledger: 
- **Not succinct.** The full key directory needs to be stored on-chain so it is always available to everyone (remember, for now we're assuming no DAS). For the ~900K [unique domain names registered in ENS at the time of this writing](https://dune.com/queries/2101527/3458714), this means nearly 90MB of persistent storage. Registering parties need to pay for the storage their entry takes up, about 65K gas (currently roughly $1 — see the performance evaluation below).
- **Somewhat interactive encryption.** The sender needs to read the chain to retrieve a user's public key, but only if it's the first time the sender is encrypting a message to that particular user. (Remember, we're assuming users don't update or revoke their keys.)
- **Not sender-anonymous.** When you retrieve someone's public key, you link yourself to the recipient, indicating that you have some kind of relationship with them. Imagine for instance that you retrieve the public key for WikiLeaks: This could create a suspicion that you are leaking classified documents. This issue could be mitigated with "cover traffic" (retrieve a big batch of keys, most of which you don't intend to use) or similarly by running a full node, or with private information retrieval (PIR).

On the positive side:
- **Non-interactive decryption.** Users decrypt messages with their locally stored secret key, so it requires no interaction.
- **Transparent.** The list of users and keys is completely public, so anyone can check whether or not they were registered correctly.
- **Arbitrary ID set.** The domain of the IDs is not limited *a priori* by the construction; in theory, the ID can be an [arbitrary string](https://opensea.io/assets/ethereum/0x57f1887a8bf19b14fc0df6fd9b2acc9af147ea85/47930729130881763180624923971074264951089763194840018679308770178224101634854) (up to the constraints imposed by the specific contract's implementation and storage). 

**When might you want to use a public key directory?** A decentralized public key directory is easy to set up and manage, so it's a good baseline choice. If storage costs or privacy are a concern, however, one of the other options below may be a better choice.

# (Threshold) Identity-Based Encryption (IBE)

**Summary**: A user's identity is their public key. A trusted third party or parties is needed to issue keys and hold a master secret (trapdoor) for the lifetime of the system.

![ibe]()

In IBE, a user doesn't generate their own key-pair but instead registers with a trusted key generator. The key generator has a "master" key-pair (msk, mpk in the figure). Given a user's ID, the key generator uses the master secret key msk and the ID to compute a secret key for the user. That secret key has to be communicated to the user over a secure channel (which can be established with a [key exchange protocol](https://en.wikipedia.org/wiki/Key_exchange)). 

IBE optimizes the sender's experience: It downloads the key generator's mpk once, after which it can encrypt a message to any ID by using the ID itself. Decryption is simple as well: a registered user can use the secret key issued to it by the key generator to decrypt the ciphertext.

The key generator's master secret key is more persistent than the ["toxic waste" generated by the trusted setup ceremonies](https://zkproof.org/2021/06/30/setup-ceremonies/) used for some SNARKs. The key generator needs it to issue any new secret keys, so the key generator can't erase it after setup the way the participants in SNARK ceremonies do. It's also a dangerous piece of information. Anyone with the msk can decrypt *all* messages sent to *any* user in the system. This means there is a constant risk of exfiltration with catastrophic consequences. 

Even if the msk is kept safe, every user who registers in the system needs to trust the key generator not to read its messages, since the key generator can keep a copy of the user's secret key or re-compute it from msk at any time. It might even be possible for the key generator to issue a faulty or "constrained" secret key to the user that decrypts all messages except some prohibited ones as determined by the key generator.

This trust can instead be distributed among a quorum of key generators, so that a user needs to trust only a threshold number of them. In that case a small number of malicious key generators can't recover secret keys or compute bad keys, and an attacker would have to break into multiple systems to steal the full master secret. 

| Positives                  | Negatives             
|:---------------------------|:---------------------------------
| Constant/minimal on-chain storage | 
| Non-interactive encryption |
| Non-interactive decryption |
|                            | Strong trust assumption
| Sender-anonymous           |
| Arbitrary IDs

If users are OK with this trust assumption, (threshold) IBE comes with a lot of benefits:
- **Constant/minimal on-chain storage.** Only the master public key (a single group element) needs to be stored on-chain. This is much less than the storage required by an on-chain public key directory. For the Boneh-Franklin IBE over the BN254 curve, this means adding 64 bytes of persistent on-chain storage once during the setup phase, requiring the service to spend only 40K gas.
- **Non-interactive encryption.** A sender only needs the master public key (which it downloads once at the start) and the recipient's ID to encrypt a message. This is in contrast to the public key directory, where it would need to retrieve a key for every new user it wants to communicate with.
- **Non-interactive decryption.** Again, users use their locally-stored secret keys to decrypt messages.
- **Sender-anonymous.** The sender doesn't need to retrieve any per-recipient information from the chain. By comparison, in the case of a public-key registry, the sender has to interact with the contract in a way that depends on the recipient.
- **Arbitrary ID set.** As in the public-key registry, the IDs can be arbitrary strings.

But!
- **Strong trust assumption.** Unlike the public-key registry, where users generate their own keys, IBE requires a trusted party or quorum of parties with a master secret (trapdoor) to issue keys. The master secret must be kept around for the whole lifetime of the system and can compromise all the registered users' messages if it is ever leaked or misused.

**When might you want to use (threshold) IBE?** If a trusted third party or parties is available and users are willing to trust them, IBE offers a much more space-efficient (and therefore cheaper) option compared to an on-chain key registry.

# Registration-Based Encryption (RBE)

Summary: Similar to IBE, a user's identity serves as their public key, but the trusted third party/quorum is replaced with a transparent accumulator (called the "key curator").

![rbe]()

In this section, I'll discuss the efficient RBE construction from [my paper](https://eprint.iacr.org/2022/1505), since unlike the (to my knowledge) [only other practical construction](https://eprint.iacr.org/2023/404), it is currently deployable on a blockchain (being pairing-based instead of lattice-based). Whenever I say "RBE" I mean this particular construction, although some statements might be true of RBE in general.

In RBE, a user generates its own key-pair and computes some "update values" (a in the figure) derived from the secret key and the common reference string (CRS). Although the presence of a CRS means that the setup isn't completely untrusted, the CRS is a [powers-of-tau](https://a16zcrypto.com/posts/article/on-chain-trusted-setup-ceremony/) construction, which can be [computed collaboratively on-chain](https://eprint.iacr.org/2022/1592) and is secure as long as there was a single honest contribution.

The smart contract is set up for an expected number of users *N*, grouped into buckets. When a user registers in the system, it sends its ID, public key, and update values to the smart contract. The smart contract maintains a set of public parameters pp (distinct from the CRS), which can be thought of as a succinct "digest" of the public keys of all the parties registered in the system. When it receives a registration request, it performs a check on the update values and multiplies the public key into the appropriate bucket of pp. 

Registered users also need to maintain some "auxiliary information" locally, which they use to help with decryption and which is appended to anytime a new user registers in their same bucket. They can update this themselves by monitoring the blockchain for registrations in their bucket, or the smart contract can help by maintaining a list of the update values it received from the most recent registrations (say, within the last week), which the users can pull periodically (at least once a week).

Senders in this system download the CRS once and occasionally download the most recent version of the public parameters. For the public parameters, all that matters from the sender's point of view is that they include the intended recipient's public key; it doesn't have to be the most up-to-date version. The sender can then use the CRS and the public parameters, along with the recipient ID, to encrypt a message to the recipient.

Upon receiving a message, the user checks its locally stored auxiliary information for a value passing some check. (If it finds none, it means it needs to fetch an update from the contract.) Using this value and its secret key, the user can decrypt the ciphertext.

Clearly, this scheme is more complex than the two others. But it requires less on-chain storage than the public-key directory while avoiding the strong trust assumption of IBE.
- **Succinct parameters.** The size of parameters to be stored on-chain is sublinear in the number of users. This is much smaller than the total storage required for a public-key directory (linear in the number of users), but still not constant and therefore worse compared to IBE.
- **Somewhat interactive encryption.** To send a message, a sender needs a copy of the public parameters which contains the intended recipient. This means it needs to update the parameters at some point after an intended recipient registers, but not necessarily for *every* intended recipient that registers, since one update might include multiple recipients' keys. Overall, message-sending is more interactive than IBE but less interactive than a directory.
- **Somewhat interactive decryption.** Similar to the encryption case, the recipient needs a copy of the auxiliary information which "matches" the version of public parameters used for the encryption. More specifically, both the public parameter and aux buckets are updated whenever a new user registers in that bucket, and the value *a* capable of decrypting a ciphertext is the one corresponding to the pp version used to encrypt. Like the public parameter updates, a user can decide to retrieve aux updates only periodically, except when decryption fails. Unlike the public parameter updates, retrieving aux updates more often doesn't inherently leak information.
- **Sender-anonymous.** As in the directory case, the sender can encrypt a message completely on its own (provided it has up-to-date parameters) without querying for recipient-specific information. The information read from the chain, when necessary, is independent of the intended recipient. (To reduce communication, the sender could request only a particular pp bucket, leaking partial information.)
- **Transparent.** Although the system needs to be set up using a (potentially distributed and/or externally administered) trusted setup ceremony outputting a punctured CRS, it does not rely on any trusted party or quorum once the setup has been completed: although it relies on a coordinating third party (the contract), it is completely transparent and anybody can be a coordinator or check they are behaving honestly by confirming its state transitions (that's why it can be implemented as a smart contract). Furthermore, users can ask for a *succinct* (non-)membership proof to check that they (or anyone else) are registered/not registered in the system. This is in contrast to the IBE case, where it is difficult for the trusted party/parties to actually prove that they didn't secretly reveal a decryption key (to themselves by making a secret copy or to someone else). The public-key directory, on the other hand, is fully transparent.
- **Restricted ID set.** I've described a "basic" version of the RBE construction. To transparently determine which bucket an ID falls into, the IDs have to have a public and deterministic ordering. Phone numbers can simply be put in order, but ordering arbitrary strings is unwieldy if not impossible since the number of buckets could be extremely large or unbounded. This could be mitigated by offering a separate contract which computes such a mapping or by adopting the cuckoo-hashing approach proposed in [this follow-up work](https://eprint.iacr.org/2023/1389).

With extensions:
- **Recipient-anonymous.** The scheme can be extended so ciphertexts don't reveal the recipient's identity.

When might you want to use RBE? RBE uses less on-chain storage than an on-chain registry while simultaneously avoiding the strong trust assumptions required by IBE. Compared to a registry, RBE also offers stronger privacy guarantees to the sender. However, since RBE only just became a viable option for key management, we're not sure yet what scenarios would most benefit from this combination of properties. Please [let me know](https://nglaeser.github.io/) if you have any ideas.

# Performance comparison

I estimated the costs (as of July 30, 2024) of deploying each of these three approaches on-chain in [this Colab notebook](https://colab.research.google.com/drive/1mEDzJqw3BCHzi-151li4Pa_4wP53VM2P?usp=sharing). The results for a system capacity of ~900K users (the number of [unique domain names registered in ENS at the time of this writing](https://dune.com/queries/2101527/3458714)) are shown in the table below.

|           | Setup cost &#13; (one-time) | Registration cost (per user)
|:----------|---------------------:|------------------------:
| **PKD**   | 0 gas (\$0.00)       | 64740 gas (\$0.96)
| **IBE**   | 41024 gas (\$0.61)   | 40 gas (\$0.01)
| **RBE**   | 110278328 gas (\$3312.35) | 185600 gas (\$6.05)

PKI has no setup cost and a low registration cost per user, while IBE has a small setup cost and practically free registration per user. RBE has a higher setup cost and also an unexpectedly high registration cost, despite the low on-chain storage required.

The bulk of the registration cost (assuming computation is done on an L2) comes from the fact that parties must update a piece of the public parameters in persistent storage, which adds up to a high registration cost.

Although RBE is more expensive than the alternatives, this analysis shows that it can be feasibly deployed on the Ethereum mainnet today — without the potentially risky trust assumptions of either PKI or IBE. And that's before optimizations like deploying multiple, smaller (and therefore cheaper) instances or using blob data. Given that RBE has advantages over the public key directory and IBE in the form of stronger anonymity and decreased trust assumptions, it could be attractive to those who value privacy and trustless setups.

# Appendix: Additional considerations

## Dealing with key updates/revocations

In the case of a public key directory, handling key updates and revocations is simple: to revoke a key, a party asks the contract to erase it from the directory. To update a key, the entry (id, pk) is overwritten with a new key to (id, pk').

For revocation in IBE, Boneh and Franklin suggested that users periodically update their keys (e.g., every week), and that senders include the current time period in the identity string when encrypting (e.g., "Bob <current week>"). Because of the non-interactive nature of encryption, senders have no way of knowing when a user revokes its key, so some period updates are inherent. Boldyreva, Goyal, and Kumar gave a [more efficient revocation mechanism](http://www.cs.cmu.edu/~goyal/ribe-proc.pdf) based on ["Fuzzy" IBE](https://eprint.iacr.org/2004/086) which requires two keys for decryption: an "identity" key and an additional "time" key. This way, only the time key must be updated regularly. The users' time keys are accumulated in a binary tree, and a user can use any node along the path (in the basic case, only the root is necessary and it's the only part that is published by the key generator). Key revocation is achieved by no longer publishing updates for a particular user (deleting nodes along that user's path from the tree), in which case the size of the update increases but is never more than logarithmic in the number of users.

Our efficient RBE construction didn't consider updates and revocations, a [follow-up work](https://eprint.iacr.org/2023/1389) gives a compiler that can "upgrade" our scheme to add these functionalities. 

## Moving data off-chain with DAS

Using a data availability solution (DAS) to move the on-chain storage off-chain would only affect the calculus for the public-key directory and RBE, reducing both to the same amount of on-chain storage. RBE would retain the advantage of being more private for the sender, since it still avoids leaking the recipient identity via access patterns. IBE already had minimal on-chain storage and would not benefit from DAS.