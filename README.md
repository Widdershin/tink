# Tink

Tink is a cryptographic library that provides a safe, simple, agile and fast way
to accomplish common cryptographic tasks. It is written by a group of
cryptographers and security engineers at Google, but it is **not an official
Google product**.  In particular, is not meant as a replacement or successor
of [Keyczar](https://github.com/google/keyczar).

**IMPORTANT: Tink is still a work-in-progress, and API might change or be
deprecated without a notice.**

## Getting started

Tink performs cryptographic tasks via so-called _primitives_, each of which is
defined via a corresponding interface that specifies the functionality of the
primitive.  For example, _symmetric key encryption_ is offered via an
_AEAD-primitive_ (Authenticated Encryption with Associated Data), that supports
two operations:

 * `encrypt(plaintext, associated_data)`, which encrypts the given `plaintext`
   (using `associated_data` as additional AEAD-input) and returns the resulting
   ciphertext
 * `decrypt(ciphertext, associated_data)`, which decrypts the given `ciphertext`
   (using `associated_data` as additional AEAD-input) and returns the resulting
   ciphertext

Currently Tink already provides primitives for several common crryptographic
operations (like symmetric encryption, message authentication codes, digital
signatures, hybrid encryption), and additional primitives are in preparation.
Moreover, adding new primitives or adding custom implementations of existing
primitives is quite straightforward.

The basic use of Tink proceeds in three steps:

1. Load or generate the cryptographic key material.
2. Use the key material to get an instance of a primitive.
3. Use the primitive to acomplish the cryptographic task.

To be more concrete, here is how these steps would look like to perform
symmetric encryption (AEAD) in a Java program:

``` java
    import com.google.cloud.crypto.tink.Aead;
    import com.google.cloud.crypto.tink.aead.AeadFactory;
    import com.google.cloud.crypto.tink.KeysetHandle;
    // (...)

    // 1. Get the key material.
    KeysetHandle keysetHandle = ...;
    // 2. Get the primitive.
    Aead aead = AeadFactory.getPrimitive(keysetHandle);
    // 3. Use the primitive.
    byte[] ciphertext = aead.encrypt(plaintext, aad);
```

In Step #1 above one can get a keysetHandle to an existing keyset kept in a
storage system...

`KeysetHandle keysetHandle = SomeKeyStorage.loadKeyset(keysetId);`

... or generate a fresh key material using the factory with a template which defines
the properties of the key that should be generated:

`KeysetHandle keysetHandle = CleartextKeysetHandle.generateNew(keyTemplate);`

For other primitives the flow is analogous, we just have to use a corresponding
factory.  For example here is how to use Message Authentication Code (MAC)
primitive:

``` java
    import com.google.cloud.crypto.tink.Mac;
    import com.google.cloud.crypto.tink.mac.MacFactory;
    import com.google.cloud.crypto.tink.KeysetHandle;
    // (...)

    // 1. Get the key material.
    KeysetHandle keysetHandle = ...;
    // 2. Get the primitive.
    Mac mac = MacFactory.getPrimitive(keysetHandle);
    // 3. Use the primitive.
    byte[] macValue = mac.computeMac(data);
```

Before a specific implementation of a primitive can be used the Tink library
must be the initialized so that it "knows" the desired implementations. This
intialization happens via _registration_ of the implementations.  For
example if one would like to use the standard implementations of AEAD and MAC
primitives offered by Tink, the initialization would look as follows:

``` java
    import com.google.cloud.crypto.tink.aead.AeadFactory;
    import com.google.cloud.crypto.tink.mac.MacFactory;
    // (...)

    // Register standard implementations of AEAD and MAC primitives.
    AeadFactory.registerStandardKeyTypes();
    MacFactory.registerStandardKeyTypes();
```

Now that you already know how to use Tink (it is really that simple!), you can
proceed with using it in your code.  Alternatively, you can continue reading the
sections below to get more information about the library: its features,
security, structure, extensibility, and more.  To get direct instructions on how
to acomplish certain tasks with Tink
see [Java HOW-TO](https://github.com/google/tink/blob/master/doc/JAVA-HOWTO.md).

## Tink overview

**Basic Features** Tink provides a set of basic tools to perform common crypto
tasks in a variety of environments. The main operations are accessible via
so-called _primitives_, which represent the corresponding cryptographic tools.
Currently Tink supports the following cryptographic operations:

- authenticated encryption with associated data (primitive: AEAD)
- message authentication codes (primitive: MAC),
- digital signatures (primitives: PublicKeySign and PublicKeyVerify)
- hybrid encryption (primitives: HybridEncrypt and HybridDecrypt).

In the future other primitives will be added, e.g. deterministic encryption
(e.g., SIV modes), pseudorandom function (e.g., HMAC as a PRF), strong
pseudorandom permutation (e.g., HEH).

**Envelope Encryption** Tink supports
[envelope](http://docs.aws.amazon.com/kms/latest/developerguide/workflow.html)
[encryption](https://cloud.google.com/kms/docs/data-encryption-keys) (a.k.a. KMS
Envelope) which is getting popular with Cloud users. In this mode, Cloud users
generate a data encryption key (DEK) locally, encrypt data with DEK, send DEK to
a KMS such as AWS KMS or Google Cloud KMS to be encrypted, and stores encrypted
DEK with encrypted data; at a later point Cloud users can retrieve encrypted
data and DEK, use the KMS to decrypt DEK, and use decrypted DEK to decrypt the
data.

**Key Management** In addition to cryptographic operations Tink provides also
support for key management featuers like key versioning, key rotation, storing
and loading keys from key management systems, and more. For example, if a
cryptographic scheme is found broken, one can switch to a new implementation of
the primitive by rotating the key without changing or recompiling code.

Currently tink supports the following key management systems:

- Google Cloud KMS
- Amazon KMS
- Android Keystore System (planned)
- Apple iOS KeyChain (planned)

One can easily add support for in-house key management system, without having
to change anything in Tink.

**Composability, Extensibility, and Interoperability** The core of Tink is
relatively small, and most components are recombinant. They can be selected and
assembled in various combinations to satisfy specific user requirements about
code size or performance. For example, if you need only digital signatures, you
don't have to include authenticated encryption.

It is easy to add new primitives, protocols or interfaces to Tink.
Without touching the core library, can easily add support for new
algorithms, new ciphertext formats, or new key management systems (e.g.,
in-house Hardware Security Modules), etc.

Tink produces and consumes ciphertexts that are compatible with other libraries.
Except in a few cases, implementations of crypto primitives in Tink come from
existing libraries such as OpenSSL, BoringSSL, NaCl, Bouncy Castle, OpenJDK,
etc. Because Tink is not a rewrite of these underlying libraries, but is rather
a high-level abstraction of them, it supports ciphertext formats and algorithms
supported by these libraries.

**Languages and Platforms** Tink for Java is field tested and ready for
production. C++ is in active development and support for Go, Python, Javascript
is in planning.  Tink supports Android, Linux (Google Cloud Engine or Amazon
EC2), Google App Engine. iOS support is in active development.


### Primitives and Their Implementations

_Primitives_ represent cryptographic operations offered by Tink, hence they form
the core of Tink API.  A primitive is just an interface that specifies what
operations are offered by the primitive.  A primitive can have multiple
implementations, and user chooses a desired implementation by using a key of
corresponding type (see the [next section](#key-keyset-and-keysethandle) for details).
Although the implementations of a given primitive can be totally independent of
each other, they all have to fulfill some baseline requirements, to assure that
the use of a particular primitive does provide sufficient protections and does
not introduce security risks (see [Security and Safety](#security-and-safety) below).

The following table summarizes Java implementations of primitives that are
currently available or planned (the latter are listed in brackets).

| Primitive          | Implementations                                     |
|--------------------|-----------------------------------------------------|
| AEAD               | AES-EAX, AES-GCM, AES-CTR-HMAC, KMS Envelope        |
| MAC                | HMAC-SHA2                                           |
| Digital Signatures | ECDSA over NIST curves, (EdDSA over Ed25519)        |
| Hybrid Encryption  | ECIES with AEAD and HKDF, (NaCl CryptoBox)          |

Tink user accesses implementations of a primitive via a factory that corresponds
to the primitive: AEAD-primitives are accessed via `AeadFactory`, MAC-primitives
via `MacFactory`, etc. where each factory offers corresponding
`getPrimitive(...)` methods.

### Key, Keyset, and KeysetHandle

A particular implementation of a _primitive_ is identified by a cryptographic
**key** structure that contains all key material and parameters needed to
provide the functionality of the primitive. The key structure is a _protocol
buffer_, whose globally unique name (a.k.a. _type url_) is referred to as **key
type**, and is used as an identifier of the corresponding implementation of a
primitive.  Any particular implementation comes in a form of a **KeyManager**
which “understands” the key type: the manager can instantiate the primitive
corresponding to a given key, or can generate new keys of the supported key
type.

To take advantage of key rotation and other key management features, a Tink user
works usually not with single keys, but with **keysets**, which are just sets of
keys with some additional parameters and metadata.  In particular, this extra
information in the keyset dermines which key is _primary_ (i.e. will be used to
create new cryptographic data like ciphertexts, or signatures), which keys are
_enabled_ (i.e. can be used to process existing cryptographic data, like decrypt
ciphertext or verify signatures), and which keys should not be used any more.
For more details about the structure of keys, keysets and related protocol buffers see
[tink.proto](https://github.com/google/tink/blob/master/proto/tink.proto).

The keys in a keyset can belong to _different implementations/key types_, but must
all implement the _same primitive_. Any given keyset (and any given key) can be
used for one primitive only.  Moreover, to protect from accidental leakage or
corruption, an Tink user doesn’t work _directly_ with keysets, but rather with
KeysetHandle objects, which form a wrapper around the keysets. Creation of
KeysetHandle objects can be restricted to specific factories (whose visibility can
be governed by a white list), to enable control over actual storage of the keys
and keysets, and so avoid accidental leakage of secret key material.

## Security and Safety

Tink reduces common crypto pitfalls with user-centered design, careful
implementation and code reviews, and extensive testing.

Tink stems from combined decades of experience in building and breaking
real-world cryptosystems. We are also maintainers of
[Project Wycheproof](https://github.com/google/wycheproof), a framework for
validating the correctness of crypto libraries, thus Tink has been
continuously tested and should be safe against known crypto weaknesses.

Tink provides APIs that decrease the potential for abuse or misuse.  In
particular, implementations of a high level interface are foolproof in the
sense: We assume that the attacker has complete freedom in calling methods of a
high level interface; under this assumption the security is not compromised.

For example if the underlying encryption mode requires nonces and is insecure
if nonces are reused then the interface do not allow to pass nonces. We also
assume that the attacker can get access to memory passed into a method, even
if a cryptographic operation (e.g. decryption) failed.

Tink's interfaces abstract away from the underlying implementation. Instances
are usable without knowing the underlying class that implements it. It is also
possible to change the underlying implementation of an interface without changes
to the call of the interface. Interfaces have security guarantees that must be
satisfied by each primitive implementing the interface.  This may exclude some
encryption modes.  Rather than adding them to existing interfaces and weakening
the guarantees of the interface it is possible to add new interfaces and
describe the security guarantees appropriately.

A good practice is to use each key for one purpose only. The storage format of
Tink keys contains information such that correct usage of the key can be
checked at runtime.

**Readability** Tink declares cryptographic properties (e.g., whether safe
against chosen-ciphertext attacks) through the _primitives_ (i.e., interfaces),
allowing security auditors and automated tools quickly discovering incorrect
usages.

On an abstract level we think that it is not necessary to know whether the
cryptosystem has been proposed by Rivest, Shamir and Adleman or by Victor
Shoup, but you should be able to determine if an object implements an
encryption or signature algorithm. Also, knowing that an encryption mode is
authenticated is more important than knowing it uses AES. This is why we name
our interfaces as generic as possible (e.g., Aead, PublicKeySign, etc.), but
not too general (e.g., Crypter, Signer, etc.)

**Visibility** Tink provides standalone static types for potential dangerous
operations (e.g., loading cleartext keys from disk), allowing restricting,
monitoring and logging their usages.

## Maintainers

Tink is maintained by:

- Daniel Bleichenbacher
- Thai Duong
- Quan Nguyen
- Bartosz Przydatek

## Contact and mailing list

If you want to contribute, please read CONTRIBUTING and send us pull requests.
You can also report bugs or request new tests.

If you'd like to talk to our developers or get notified about major new tests,
you may want to subscribe to our
[mailing list](https://groups.google.com/forum/#!forum/tink-users). To join,
simply send an empty email to tink-users+subscribe@googlegroups.com.
