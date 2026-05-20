# Self-Hostable BBID Architecture

Let

```
[BBID]
```

denote a self-hostable identity/projection system with architecture

```
[BBID]
=============

(
  G;
  O_KDF;
  C_Wasm;
  Π
)
```

where:

```
G
```

is a graph database,

```
O_KDF
```

is a key-derivation oracle,

```
C_Wasm
```

is a browser-local WebAssembly client, and

```
Π
```

is the projection pipeline.

---

## 1. Graph State

Let each visitor be represented by a node

```
v_i ∈ V(G)
```

with public metadata

```
meta(v_i)
==================

(
  δ_i;
  s_i;
  r_i;
  a_i
)
```

where:

```
δ_i = blinding factor
```

```
s_i = user-specific salt
```

```
r_i = version parameter
```

```
a_i = hash anchor
```

Thus the graph stores

```
G = (V,E)
```

with

```
V = {v_1,...,v_n}
```

and

```
∀ v_i ∈ V,      meta(v_i)
⊆
{public, non-secret metadata}
```

No master secret is stored in the graph.

---

## 2. Master System Secret

Let

```
M_sys ∈ {0,1}^λ
```

be the master system secret.

Self-hosting requires

```
M_sys
```

to be protected by the operator's infrastructure, e.g.

```
M_sys
∈
{encrypted environment variable, TEE, HSM, sealed container secret}
```

The security condition is:

```
M_sys ∉ G
```

and

```
M_sys ∉ C_Wasm
```

---

## 3. Key-Derivation Oracle

Define the oracle

```
O_KDF
:
(v_i, H_i)
↦
z_i
```

where

```
H_i
```

is the client-observed signal vector comprising:

```
H_i
===

(H_i^hw,;
H_i^behav)
```

with

```
H_i^hw
===

hardware/browser/device features

(e.g., canvas fingerprint, screen geometry, CPU)
```

and

```
H_i^behav
===

behavioral features

(e.g., keystroke dynamics, mouse kinematics, scroll entropy)
```

This separation enables explicit mismatch detection to distinguish:
- Hardware-only mismatch (different device, same user)
- Behavioral-only mismatch (same device, different user)
- Combined mismatch (different device, different user)

The oracle first queries

```
G
```

for the visitor metadata:

```
(δ_i,s_i,r_i,a_i)
←
G[v_i]
```

Then it derives a seed using HKDF-SHA256:

```
z_i
===

HKDF_SHA256
(
  IKM=M_sys,
  salt=s_i,
  info
=============

  enc
  (
    v_i,δ_i,r_i,a_i,H_i
  )
)
```

Thus:

```
z_i
===

O_KDF
(
  M_sys,s_i,δ_i,r_i,a_i,H_i
)
```

The oracle returns

```
z_i
```

to the client, but never returns

```
M_sys
```

or any intermediate secret material.

---

## 4. Client-Side Expansion

The WebAssembly client receives

```
z_i
```

and expands it locally using AES-CTR-DRBG:

```
R_i
===

AES-CTR-DRBG(z_i)
```

where

```
R_i = (r_{i,1},r_{i,2},...,r_{i,k})
```

is a deterministic pseudorandom stream.

The client then constructs projection hyperplanes

```
H_i
=============

{h_{i,1},...,h_{i,m}}
```

from

```
R_i
```

via

```
H_i
=============

Φ(R_i)
```

where

```
Φ
```

is the deterministic projection-construction function.

---

## 5. Projection and BBID Generation

Let

```
x_i
```

be the client-side feature vector derived from browser/device/user-interaction signals.

The BBID projection is

```
b_i
===

Π(x_i;H_i)
```

or explicitly:

```
b_i
===

Π
(
  x_i;
  Φ
  (
    AES-CTR-DRBG
    (
      HKDF_SHA256
      (
        M_sys,
        s_i,
        enc(v_i,δ_i,r_i,a_i,H_i)
      )
    )
  )
)
```

The browser then zeroizes local secret material:

```
z_i ← 0
```

```
R_i ← 0
```

```
H_i ← 0
```

after use.

---

## 6. Self-Hosting Condition

BBID is self-hostable iff there exists an operator-controlled environment

```
E
```

such that

```
E
===========

(
  G;
  O_KDF;
  C_Wasm
)
```

and the following conditions hold:

```
G stores only public metadata
```

```
O_KDF securely stores M_sys
```

```
C_Wasm performs expansion, projection, and zeroization locally
```

```
M_sys
∉
G
∪
C_Wasm
```

Therefore:

```
{
  BBID is self-hostable
}
```

provided the operator supplies:

```
{
  G
  +
  O_KDF
  +
  C_Wasm
  +
  SecureStorage(M_sys)
}
```

---

## 7. Deployment-Tier Interpretation

Let

```
H_i
```

be the hardware/browser signal vector.

For open-web deployments:

```
H_i
∼
semi-deterministic noise
```

because browser-accessible device signals are unstable and partially spoofable.

For native enterprise deployments:

```
H_i
≈
PUF/PPA(d_i)
```

where

```
d_i
```

is a device and

```
PUF/PPA
```

denotes hardware-bound attestation or physically anchored provenance.

Thus:

```
Open Web:
      H_i is weakly bound
```

```
Native Enterprise:
      H_i is strongly bound
```

---

## 8. Platform Independence

Cloudflare Workers are one implementation of

```
O_KDF
```

but are not required.

The architecture requires only an edge or server runtime

```
S
```

such that

```
S ⊢ O_KDF
```

and

```
S can securely access M_sys
```

and

```
S can query G
```

Therefore:

```
S
∈
{
  Cloudflare Worker,
  self-hosted server,
  edge function,
  containerized service,
  enterprise enclave
}
```

All valid implementations satisfy:

```
S ⇒
[
  HKDF_SHA256
  →
  AES-CTR-DRBG
  →
  Π
]
```

with no Cloudflare-specific cryptographic dependency.

---

## 9. Final Compact Form

```
{
  BBID_selfhost
=================================

G_public
+
O_KDF(M_sys)
+
C_Wasm
[
  DRBG
  ∘
  Projection
  ∘
  Zeroize
]
}
```

where

```
{
  O_KDF
==========================

  HKDF_SHA256
  (
    M_sys,
    s_i,
    enc(v_i,δ_i,r_i,a_i,H_i)
  )
}
```

and

```
{
  BBID does not require Cloudflare; it requires secure derivation, graph-backed metadata, and local projection.
}
```
