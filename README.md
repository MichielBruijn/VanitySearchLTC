# VanitySearchLTC

VanitySearchLTC is a Litecoin vanity address generator, forked from [JeanLucPons/VanitySearch](https://github.com/JeanLucPons/VanitySearch) and ported to Litecoin. It supports P2PKH (`L...`), P2SH (`M...`) and native SegWit bech32 (`ltc1q...`) addresses.

If you want to generate safe private keys, use the `-s` option to enter your passphrase which will be used for generating a base key as for BIP38 standard (*./VanitySearch -s "My PassPhrase" Lprefix*). You can also use *./VanitySearch -ps "My PassPhrase"* which will add a crypto secure seed to your passphrase.

VanitySearch may not compute a good grid size for your GPU, so try different values using the `-g` option to get the best performance. If you want to use GPUs and CPUs together, you may have best performance by keeping one CPU core for handling GPU(s)/CPU exchanges (use `-t` to set the number of CPU threads).

# Features

<ul>
  <li>Fixed size arithmetic</li>
  <li>Fast Modular Inversion (Delayed Right Shift 62 bits)</li>
  <li>SecpK1 Fast modular multiplication (2 steps folding 512bits to 256bits using 64 bits digits)</li>
  <li>Use some properties of elliptic curve to generate more keys</li>
  <li>SSE Secure Hash Algorithm SHA256 and RIPEMD160 (CPU)</li>
  <li>Multi-GPU support</li>
  <li>CUDA optimisation via inline PTX assembly</li>
  <li>Seed protected by pbkdf2_hmac_sha512 (BIP38)</li>
  <li>Support P2PKH, P2SH and BECH32 addresses</li>
  <li>Support split-key vanity address</li>
</ul>

# Address formats

| Format | Prefix | Example |
|--------|--------|---------|
| P2PKH (Legacy) | `L` | `LMyAddress...` |
| P2SH (SegWit wrapped) | `M` | `MMyCoin...` |
| Bech32 (Native SegWit) | `ltc1q` | `ltc1q337...` |

> **Note:** Bech32 addresses always start with `ltc1q`. The `q` encodes witness version 0 and is mandatory. Only bech32 characters are allowed after `ltc1q`: `023456789acdefghjklmnpqrstuvwxyz` (no `1`, `b`, `i` or `o`).

# Usage

```
VanitySearch [-check] [-v] [-u] [-b] [-c] [-gpu] [-stop] [-i inputfile]
             [-gpuId gpuId1[,gpuId2,...]] [-g g1x,g1y,[,g2x,g2y,...]]
             [-o outputfile] [-m maxFound] [-ps seed] [-s seed] [-t nbThread]
             [-nosse] [-r rekey] [-check] [-kp] [-sp startPubKey]
             [-rp privkey partialkeyfile] [prefix]

 prefix: prefix to search (Can contains wildcard '?' or '*')
 -v: Print version
 -u: Search uncompressed addresses
 -b: Search both uncompressed or compressed addresses
 -c: Case unsensitive search
 -gpu: Enable gpu calculation
 -stop: Stop when all prefixes are found
 -i inputfile: Get list of prefixes to search from specified file
 -o outputfile: Output results to the specified file
 -gpu gpuId1,gpuId2,...: List of GPU(s) to use, default is 0
 -g g1x,g1y,g2x,g2y, ...: Specify GPU(s) kernel gridsize, default is 8*(MP number),128
 -m: Specify maximun number of prefixes found by each kernel call
 -s seed: Specify a seed for the base key, default is random
 -ps seed: Specify a seed concatened with a crypto secure random seed
 -t threadNumber: Specify number of CPU thread, default is number of core
 -nosse: Disable SSE hash function
 -l: List cuda enabled devices
 -check: Check CPU and GPU kernel vs CPU
 -cp privKey: Compute public key (privKey in hex format)
 -kp: Generate key pair
 -rp privkey partialkeyfile: Reconstruct final private key(s) from partial key(s) info.
 -sp startPubKey: Start the search with a pubKey (for private key splitting)
 -r rekey: Rekey interval in MegaKey, default is disabled
```

## Examples

Search for a bech32 (ltc1q) vanity address:
```
$ ./VanitySearch -gpu -stop ltc1q337
VanitySearch v1.19
Difficulty: 1048576
Search: ltc1q337 [Compressed]
...
Pub Addr: ltc1q337...
Priv (WIF): p2wpkh:T...
Priv (HEX): 0x...
```

Search for a legacy (L) vanity address:
```
$ ./VanitySearch -gpu -stop Ltest
VanitySearch v1.19
Difficulty: 195766468
Search: Ltest [Compressed]
...
Pub Addr: Ltest...
Priv (WIF): p2pkh:T...
Priv (HEX): 0x...
```

Search for a P2SH (M) vanity address:
```
$ ./VanitySearch -gpu -stop MMyCoin
VanitySearch v1.19
Difficulty: 15318045009
Search: MMyCoin [Compressed]
...
Pub Addr: MMyCoin...
Priv (WIF): p2wpkh-p2sh:T...
Priv (HEX): 0x...
```

# Generate a vanity address for a third party using split-key

It is possible to generate a vanity address for a third party in a safe manner using split-key.\
For instance, Alice wants a nice prefix but does not have GPU power. Bob has the requested GPU power but cannot know the private key of Alice, so Alice uses a split-key.

## Step 1

Alice generates a key pair on her computer then sends the generated public key and the wanted prefix to Bob. It can be done by email — nothing is secret. Alice must keep her private key safe and never expose it.
```
./VanitySearch -s "AliceSeed" -kp
Priv : T...
Pub  : 03FC71AE1E88F143E8B05326FC9A83F4DAB93EA88FFEACD37465ED843FCC75AA81
```
Note: The key pair is a standard SecpK1 key pair and can be generated with third party software.

## Step 2

Bob runs VanitySearch using Alice's public key and the wanted prefix.
```
./VanitySearch -sp 03FC71AE1E88F143E8B05326FC9A83F4DAB93EA88FFEACD37465ED843FCC75AA81 -gpu -stop -o keyinfo.txt ltc1qalice
```
It generates a keyinfo.txt file containing the partial private key.
```
PubAddress: ltc1qalice...
PartialPriv: T...
```
Bob sends this file to Alice. The partial private key does not allow anyone to guess Alice's final private key.

## Step 3

Alice reconstructs the final private key using her private key (from step 1) and the keyinfo.txt from Bob.

```
./VanitySearch -rp T... keyinfo.txt

Pub Addr: ltc1qalice...
Priv (WIF): p2wpkh:T...
Priv (HEX): 0x...
```

## How it works

The `-sp` (start public key) adds the specified starting public key (Q) to the starting keys of each thread. When searching with `-sp`, you search for addr(k<sub>part</sub>.G+Q) instead of addr(k.G). The requester reconstructs the final private key as k<sub>part</sub>+k<sub>secret</sub> (mod n) using the `-rp` option. The searcher cannot guess the final private key because he only knows Q, not k<sub>secret</sub>.

Note: This explanation is simplified and does not account for symmetry and endomorphism optimizations.

# Compilation

## Linux (recommended)

Requirements:
- CUDA SDK 12.x
- g++-12 or newer

```sh
sudo apt install libssl-dev libpcre3-dev g++-12
```

Build with GPU support (replace `89` with your GPU's compute capability):
```sh
make all gpu=1 CCAP=89
```

Build CPU-only (no CUDA):
```sh
make all
```

Common compute capabilities:

| GPU generation | CCAP |
|----------------|------|
| RTX 30xx / Ada (30xx Laptop) | 86 / 89 |
| RTX 40xx | 89 |
| RTX 20xx | 75 |
| GTX 10xx | 61 |

Run:
```sh
./VanitySearch -gpu -stop ltc1q337
```

## Windows

Install CUDA SDK and open `VanitySearch.sln` in Visual C++ 2019 or later.\
In Build → Configuration Manager, select the *Release* configuration.\
Build and run.

# License

VanitySearchLTC is licensed under GPLv3.\
Original VanitySearch by Jean-Luc PONS — [github.com/JeanLucPons/VanitySearch](https://github.com/JeanLucPons/VanitySearch)
