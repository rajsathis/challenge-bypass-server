你好！
很冒昧用这样的方式来和你沟通，如有打扰请忽略我的提交哈。我是光年实验室（gnlab.com）的HR，在招Golang开发工程师，我们是一个技术型团队，技术氛围非常好。全职和兼职都可以，不过最好是全职，工作地点杭州。
我们公司是做流量增长的，Golang负责开发SAAS平台的应用，我们做的很多应用是全新的，工作非常有挑战也很有意思，是国内很多大厂的顾问。
如果有兴趣的话加我微信：13515810775  ，也可以访问 https://gnlab.com/，联系客服转发给HR。
# Challenge Bypass Server

This is a TCP server that is comptaible with the [Privacy Pass](https://github.com/privacypass/challenge-bypass-extension) browser extension. In particular, the server implements the server-side actions in a 'blind signing' protocol. The protocol is based on a variant of a Verifiable, Oblivious Pseudorandom Function (VOPRF) established by [Jarecki, Kiayias and Krawczyk](https://eprint.iacr.org/2014/650.pdf). When adapted to our needs, this scheme allows us to achieve the same goals as a 'Blind RSA' variant but using faster primitives, less bandwidth, and simpler secret-key operational logistics in comparison.

### Contents

  * [Quickstart](#quickstart)
  * [Current functionality](#current-functionality)
  * [Overview of design](#overview-of-design)
     * [Preliminaries](#preliminaries)
     * [Message formats](#message-formats)
        * [Issuance request](#issuance-request)
        * [Issue response](#issue-response)
        * [Redemption request](#redemption-request)
        * [Redemption response](#redemption-response)
     * [Protocol overview](#protocol-overview)
        * [NIZK proofs of discrete-log equality](#nizk-proofs-of-discrete-log-equality)
        * [Batch proofs](#batch-proofs)
  * [Team](#team)
  * [Acknowledgements](#acknowledgements)

## Quickstart

To run the server:

`go run server/main.go --key testdata/p256-key.pem --comm testdata/test-p256-commitment`

here, `key` is the current secret key used for signing, `comm` is the public commitment to the signing key.

To demo token issuance:

`cat testdata/bl_sig_req | nc localhost 2416`

Keys that are only valid for verifying token redemptions can be provided in a single `.pem` file. For example, using:

`--redeem_keys testdata/p256-redeem-keys.pem`

the signing key, is automatically also used for redemption.

For a full client implementation, and further details on required message formatting and technical considerations, see the [browser extension](https://github.com/privacypass/challenge-bypass-extension).

## Current functionality

- Signs blinded tokens and generates valid batch DLEQ proof
- Verifies redemption tokens

## Overview of design

### Preliminaries

A **message authentication code (MAC)** on a message is a keyed authentication tag that can be only be created and verified by the holder of the key.

A **pseudorandom function** is a function whose output cannot be efficiently distinguished from random output. This is a general class of functions; concrete examples include hashes and encryption algorithms.

An **oblivious pseudorandom function (OPRF)** is a two-party protocol between sender *S* and receiver *R* for securely computing a pseudorandom function *f_x(·)* on key *x* contributed by *S* and input *t* contributed by *R*, in such a way that receiver *R* learns only the value *f_x(t)* while sender *S* learns nothing from the interaction. The output of an OPRF on some input *t* is known as a **signature** for *t*.

In this protocol, the edge (also referred to as the server) is the "sender" holding *x* and the inputs *t* are the tokens. So the clients don't learn our key and we don't learn the token values.

### Message formats

We provide a brief overview of the message types that are sent and received by this plugin. These messages are sent and received in base64 encoding and JSON structs are used where appropriate. These formats differ slightly from those sent/received by the corresponding [browser extension](https://github.com/privacypass/challenge-bypass-extension) which uses intermediate HTTP communications with the edge before messages reach here.

In the following `||` will denote concatenation.

#### Issuance request

JSON struct used for requesting signatures on blinded tokens.

- `<blind-token>` is a randomly sampled, blinded elliptic curve point (this point is sent in compressed format as defined in Section 2.3.3 of http://www.secg.org/sec1-v2.pdf). The blind is also randomly sampled with respect to the same group.

- `<contents>` is an array of N `<blind-token>` objects.

- `<Issue-JSON-struct>`:

    ```
    {
        "type": "Issue",
        "contents": "<contents>",
    }
    ```
- `<Wrapped-Issue>` is the message that is actually received:
    
    ```
    {
        "bl_sig_req": "base64(<Issue-JSON-struct>)",
    }
    ```

#### Issue response

Marshaled array used for sending signed tokens back to a client.

- `<signed-tokens>` is an array of compressed elliptic curve point, as above, that have been 'signed' by the edge. 

- `<proof>` is a base64 encoded JSON struct containing the necessary information for carrying out a DLEQ proof verification. In particular it contains base64 encodings of compressed elliptic curve points `G`,`Y`,`M`,`Z` along with response values `R` and `C` for streamlining the proof verification. See [below](#nizk-proofs-of-discrete-log-equality) for more details.

- `<M>` and `<Z>` are base64 encoded compressed elliptic curve points. `<C>` is a base64 encoded array of bytes used for checking completeness.

- `<batch-proof>` is a JSON struct of the form:<sup>2</sup>

    ```
    {
        "proof":"<proof>",
        "M":"<M>",
        "Z":"<Z>",
        "C":"<C>",
    }
    ```

<sup>2</sup> Other [VRF implementations](https://datatracker.ietf.org/doc/draft-goldbe-vrf/?include_text=1) use different notation to us. We have tried to coincide as much as possible with these works.

- `<Batch-DLEQ-Resp>`:
    
    `"batch-proof=" || <batch-proof>` 

- Issue response:
    
    `base64(<signed-tokens> || <Batch-DLEQ-Resp>)`

#### Redemption request

Primary client request that is verified by btd

- `<token>` is an original token generated for an issuance request.

- `<shared-point>` is the corresponding unblinded, signed point received from an issuance response. This point is SEC1 encoded.

- `<shared-info>` is some shared information that is used for deriving a HMAC key (usually a host header and a http path)

- `HMAC()` is a HMAC function that uses SHA256

- `<derived-key>` is the derived key output by:
    
    `HMAC("hash_derive_key", <token>, <shared-point>)`

- `<request-binding>` is the output of the following:

    `HMAC("hash_request_binding", <derived-key>, <shared-info>)`

- `<Redeem-JSON-struct>`:

    ```
    {
        "type":"Redeem",
        "contents":"<request-binding>"
    }
    ```
- `<Wrapped-Redeem>` is the JSON struct that is actually received:
    
    ```
    {
        "bl_sig_req":"base64(<Redeem-JSON-struct>)",
        "host":"<host>"
        "http":"<http-path>"
    }
    ```

#### Redemption response

Server response header used if errors occur. If this header is sent the plugin discards all stored tokens.

- `<resp-val>` is the value returned by btd on verification. Returns "success" on successful verification. If an error has occurred then it takes the value 5 or 6, where 5 is a connection error and 6 is a token verification error.

### Protocol overview

We detail a 'blind-signing' protocol written by George Tankersley that uses a VOPRF to contruct per-token shared keys for a MAC over each redemption request. This hides the token values themselves until redemption and obviates the need for public key encryption. A full technical overview of this protocol can be read [here](https://github.com/privacypass/challenge-bypass-extension/PROTOCOL.md).

Given a group setting and three hashes `H_1, H_2, H_3` we build a commitment to a random token per request using a secret key `x` held by the edge servers. `H_1` and `H_2` are hash functions onto, respectively, the group and `{0, 1}^λ` where `λ` is a security parameter. The edge (or server) also publishes publicly a generator `G` along with a commitment (or 'public key') `Y = xG` (in the code we use `H` instead of `Y` but use `Y` to avoid conflict with the hash functions that we use). The pair `(G,Y)` is used for generating discrete-log equivalence proofs (DLEQs). We provide a brief overview of the how DLEQs are generated and verified [below](#nizk-proofs-of-discrete-log-equality).

1. Client generates random token `t` and a blinding factor `r`.
2. Client calculates `T = H_1(t)` and sends `M = rT` to the edge along with a CAPTCHA solution.
3. Edge validates the solution and computes `Z = xM = rxT` and a DLEQ proof `π` dependent on the output of `H_3` over `G,Y,M,Z` and other group elements -- see below for more details.
4. The edge returns `(Z,π)` to the client.
5. Client unblinds `Z` to retrieve `N = r^(-1)Z = xT` and stores the pair `(t,N)`.
6. When the client wants to redeem a token it presents `(t, MAC(request-binding-data))` where `request-binding-data` is made of information observable by the edge that is unique(ish) to that particular request.
7. The edge uses `T` as a double-spend index and recalculates `N` using `x`. Then it can validate the MAC using the shared key.
8. We know that a matching commitment value is valid because generating it requires access to `x`.

#### NIZK proofs of discrete-log equality

In step (3.) above, we call for a zero-knowledge proof of the equality of a discrete logarithm (our edge key) with regard to the returned token `Z` that the client receives. This allows the client to verify that the same key `x` is being used to 'sign' all tokens that are sent to the edge.

The protocol naturally provides `Z = xM` in the edge response. To ensure that the edge has not used unique `x` value to tag users, we require them to publish a public key, `Y = xG`, as mentioned above. We can now use a Chaum-Pedersen proof [CP93] to prove in zero knowledge that `log_G(Y) == log_M(Z)`. We note this as `DLEQ(Z/M == Y/G)`.

The proof follows the standard non-interactive Schnorr pattern. For a group of prime order `q` with orthogonal generators `M`, `G`, public key `Y`, and point `Z`:

1. Prover chooses a random nonce

        k <--$-- Z/qZ

2. Prover commits to the nonce `k` with respect to both generators

        A = kG, B = kM

3. Prover constructs the challenge

        c = H_3(G,Y,M,Z,A,B)

4. Prover calculates response

        s = k - cx (mod q)

5. Prover sends `(c, s)` to the verifier

6. Verifier recalculates commitments

        A' = sG + cY
        B' = sM + cZ

7. Verifier hashes

        c' = H_3(G,Y,M,Z,A',B')

   and checks that `c == c'`.

If all users share a consistent view of the tuple `(G,Y)` for each key epoch, they can all prove that the tokens that every client has been issued share the same anonymity set with respect to `x`. One way to ensure this consistent view is to pin one key at a time in each copy of the client and use software update mechanisms for rotation. A more flexible way is to pin a reference that allows each client to fetch the latest version of the key from a trusted location. We currently use the former method but plan to migrate to the latter in the near future.


#### Batch proofs

In practice, the issuance protocol operates over sets of tokens rather than just one. A system parameter, m, determines how many tokens a user is allowed to request per valid CAPTCHA solution. Consequently, users generate `(t_1, t_2, ... , t_m)` and `(r_1, r_2, ... , r_m)`; send `(M_1, M_2, ... , M_m)` to the edge; and receive `(Z_1, Z_2 ... , Z_m)` in response.

Generating an independent proof of equality for each point implies excess overhead in both computation and bandwidth consumption. Therefore, we employ a batch proof to show consistent key usage for an entire set of tokens at once.  The proof is a parallelized Schnorr protocol for the common-exponent case taken from [Hen14] and adapted for non-interactivity:

Given `(G, Y, q)`; `(M_1,...,M_m)`, `(Z_1, ... ,Z_m)`; `Z_i = k(M_i)` for `i = 1...m`

1. Prover calculates a seed using:

        z = H_3(G, Y, M_1, ... , M_m, Z_1, ... , Z_m)

2. Prover initializes `PRNG(z)` and samples from it to non-interactively generate

        c_1, ... , c_m <--$-- Z/qZ.

3. Prover generates composite group elements `M` and `Z`

        M = (c_1*M_1) + (c_2*M_2) + ... + (c_m*M_m)
        Z = (c_1*Z_1) + (c_2*Z_2) + ... + (c_m*Z_m)

4. Prover sends proof

        (c, s) <-- DLEQ(M/Z == Y/G)

5. Verifier recalculates the PRNG seed from protocol state, generates the composite elements, and checks that `c' == c` as in the single-element proof above.

We can see why this works in a reduced case.

For `(M_1, M_2)`, `(Z_1, Z_2)`, and `(c_1, c_2)`:

    Z_1 = x(M_1)
    Z_2 = x(M_2)
    (c_1*Z_1) = c_1(x*M_1) = x(c_1*M_1)
    (c_2*Z_2) = c_2(x*M_2) = x(c_2*M_2)
    (c_1*Z_1) + (c_2*Z_2) = x[(c_1*M_1) + (c_2*M_2)]

So the composite points will have the same discrete log relation `x` as the underlying individual points.

## Team

- [Alex Davidson](https://alxdavids.xyz)
- [Ian Goldberg](https://cs.uwaterloo.ca/~iang/)
- [Nick Sullivan](https://github.com/grittygrease)
- [George Tankersley](https://gtank.cc)
- [Filippo Valsorda](https://github.com/filosottile)

## Acknowledgements

We'd like to thank Dan Boneh for suggesting OPRFs in the first place; Ian Goldberg for his extensive advice and the batch proof; and Brian Warner, Zaki Manian, Tony Arcieri, Isis Lovecruft, Henry de Valence, Trevor Perrin, Yan Zhu, Peter Wu, Zi Lin and several anonymous others for their valuable help, input, and review.
