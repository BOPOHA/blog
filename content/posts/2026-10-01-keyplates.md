---
layout: post
title: "KeyPlates: Vanity Stellar Addresses, Found on a GPU and Delivered Without a Key Handover"
date: 2026-10-01
tags: [keyplates, stellar, xlm, cuda, metal, gpu, ed25519, vanity-address]
---

A Stellar address looks like this:
`GD4DNGO25NUOX5TCYBSNRGJ3JEYHCB2GAA2FC4AKADFQQYSKEYPLATES`. It is 56 characters
of base32. Nobody remembers it, and everybody checks only the first and the last
four characters. Scammers know this very well.

Now look at the end of that address again: `…KEYPLATES`. That is a vanity
address. A word in an address is easier to recognize and easier to check. It is
also a nice thing to own, like a good car number plate. Hence the name:
[KeyPlates](https://keyplates.com).

## How a vanity address is found

You cannot choose an address. It is derived from a random private key. To get a
word into it, a program generates keys, one after another, and keeps the key
whose address contains the word. Each extra character makes the search 32 times
longer:

| Characters | Keys to try (average) | One NVIDIA L4 GPU |
| --- | ---: | ---: |
| 7 | 3.4 × 10^10 | 6 minutes |
| 8 | 1.1 × 10^12 | 3 hours |
| 9 | 3.5 × 10^13 | 4 days |
| 10 | 1.1 × 10^15 | 130 days |

One detail of Stellar is important here. The character right after `G` can only
be `A`, `B`, `C`, or `D`. A word that starts with one of these letters can sit
right after the `G` (`GDREAMIEST…`). Any other word starts one character later.
There are 10 kinds of people: people who know this, and people who type
`GZEBRA` on the order page and learn it there.

## The engine

The search engine started as my research project on Ed25519 key generation on
GPUs. It has three backends:

- **CUDA** on an NVIDIA L4: about 100 million keys per second;
- **Metal** on my M1 Max laptop: about 25–28 million keys per second;
- **Go on the CPU**: slow, exact, and the reference for all tests.

The interesting part is the matching. The engine does not search for one word.
It searches for about 27,000 words at the same time, at the start and at the end
of the address. A hashed lookup on the GPU makes the speed independent of the
number of words: I measured it as flat up to 100,000 patterns. Each match goes
back to the host, and the host verifies it again on the CPU before it stores
anything. The GPU is fast. The CPU is the auditor.

## The trust problem

Whoever finds a vanity address knows its private key. If a seller sends you that
key, the seller can keep a copy and empty the account later. You must trust the
seller forever. I do not like "trust me forever" as a business model.

KeyPlates never sends a key. Stellar accounts can have several signers with
weights, and delivery uses that. It is one transaction:

```text
op 1  CreateAccount   the vanity address, 2 XLM starting balance
op 2  SetOptions      on the vanity account:
                      add the buyer's key as a signer, weight 1
                      set the vanity key's own weight to 0
                      thresholds 1 / 1 / 1
```

After this transaction, the key that we found has weight 0. It can sign nothing.
The buyer's own wallet key is the only signer. Both operations are in one
transaction, so there is no moment when the old key still controls the account.

You do not have to believe me. Paste the address on the
[verify page](https://keyplates.com/verify/), or in any Stellar explorer, and
look at the signers. The independent checker on
[splitkey.me](https://splitkey.me/verify/) shows the same.

## How to buy

1. Pick an address in the [catalogue](https://keyplates.com/).
2. Pay in XLM or USDC from your own wallet. Lobstr, Varial (formerly
   Freighter), xBull, Albedo, HOT Wallet, and Ledger connect directly. Other
   wallets can scan a Stellar payment QR code.
3. The key that pays becomes the only signer. Delivery is automatic.

Pay from your own wallet, not from an exchange. An exchange account has no key
of yours that can become the signer.

For a word of your own, the [custom order page](https://keyplates.com/custom/)
shows the price and the delivery time while you type.

## Some engineering notes

- **The keys are always encrypted.** The GPU host has only the public recipient
  key of the vault (age, X25519). It can seal the keys that it finds, but it
  cannot open them. The laptop holds the identity, encrypted with a passphrase.
- **The payment watcher is a boring program, on purpose.** It reads the treasury
  payments from Horizon. It delivers, refunds per clear rules, and holds anything
  unclear for a human. Boring programs do not lose money.
- **Address poisoning is real.** After a payment, bots send tiny amounts from
  addresses that start and end like yours. They hope that you copy their address
  from your history. The watcher ignores this dust. You must ignore it too: copy
  addresses from your wallet, never from the transaction history.

## Read more

- What a Stellar vanity address is, and how to buy one safely:
  [keyplates.com/learn/stellar-vanity-address](https://keyplates.com/learn/stellar-vanity-address/)
- Delivery by rotation, in detail: [splitkey.me/stellar](https://splitkey.me/stellar/)
- Questions and answers: [keyplates.com/faq](https://keyplates.com/faq/)

If you buy an address with your name in it, tell me. I will be very glad, and
I will check its signers on the verify page, of course.
