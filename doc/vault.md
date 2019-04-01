# Aegis Vault

This __WIP__ document describes Aegis' security design and vault format in
detail.

## Primitives

Two cryptographic primitives were selected for use in Aegis. An Authenticated
Encryption with Associated Data (AEAD) cipher and a Key Derivation Function
(KDF).

### AEAD

AES-GCM with a 256-bit key is used as the AEAD cipher to ensure the
confidentility, integrity and authenticity of the vault contents.

It requires a unique 96-bit nonce for each invocation with the same key.
However, it is not possible to use a monotically increasing counter for this in
this case, because a future use case could involve using the vault on multiple
devices simultaneously, which would almost certainly result in nonce reuse. This
is suboptimal, because 96 bits is not large enough to comfortably generate an
unlimited amount of random numbers without getting collisions at some point
either. As a repeat of the nonce would have catastrophic consequences for the
confidentiality and integrity of the ciphertext, NIST strongly recommends not
exceeding 2<sup>32</sup> invocations<sup>1</sup> when using random nonces with
GCM. As such, the security of the Aegis vault also relies on the assumption that
this limit is never exceeded. In the case of Aegis, this is a reasonable
assumption to make, as it's highly unlikely that a user will ever come close to
saving the vault 2<sup>32</sup> times.

_Switching to a nonce misuse-resistant cipher like AES-GCM-SIV or a cipher with
a larger nonce like XChaCha-Poly1305 (192 bits) will be explored in the future._

### KDF

scrypt is used as the KDF to derive a key from a user-provided password.

_Argon2 is a more modern KDF that provides an advantage over scrypt because it
allows tweaking the memory-hardness parameter and CPU-hardness parameter
separately, whereas scrypt ties those together into one cost parameter (N). As
many applications have started using Argon2 in production, it seems that it has
withstood the test of time. It will be considered as an alternative option to
switch to in the future._

## Format

TODO.

### Integrity

Because of the use of an AEAD for encryption, the content and master key in the
slots are checked for integrity and authenticity. The rest of the file is not.

### Header

TODO.

#### Slots

The different slot types are identified with a numerical ID. 

| Type        | ID   |
| :---------- | :--- |
| Raw         | 0x00 |
| Fingerprint | 0x01 |
| Password    | 0x02 |

##### Raw

This slot type is used for raw AES keys. All other slots are based on this slot
type, so this section applies to all of them. Each slot has a unique randomly
generated UUID. The key is used to to wrap the master key using AES-GCM. The
encrypted ``key`` is then stored along with the generated ``nonce`` and ``tag``.

```json
{
    "type": 0,
    "uuid": "01234567-89ab-cdef-0123-456789abcdef",
    "key": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "key_params": {
        "nonce": "0123456789abcdef01234567",
        "tag": "0123456789abcdef0123456789abcdef"
    }
}
```

##### Fingerprint

The structure of the fingerprint slot is exactly the same as the Raw slot. The
difference is the key is backed by the Android KeyStore.

##### Password

As noted earlier, scrypt is used to derive a key from a user-provided password. 

```json
{
    "type": 1,
    "uuid": "01234567-89ab-cdef-0123-456789abcdef",
    "key": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef",
    "key_params": {
        "nonce": "0123456789abcdef01234567",
        "tag": "0123456789abcdef0123456789abcdef"
    },
    "n": 32768,
    "r": 8,
    "p": 1,
    "salt": "0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef"
}
```

### Content

The content is a JSON object encoded in UTF-8.

```json
{
    "version": 1,
    "entries": []
}
```

It has a version number and a list of entries. If a backwards incompatible
change is introduced to the content format, the version number will be
incremented.

#### Entries

```json
{
    "type": "totp",
    "uuid": "01234567-89ab-cdef-0123-456789abcdef",
    "name": "Bob",
    "issuer": "Google",
    "icon": null,
    "info": {
        "secret": "ABCDEFGHIJKLMNOPQRSTUVWXYZ234567",
        "algo": "SHA1",
        "digits": 6,
        "period": 30
    }
}
```

## References

\[1]: Recommendation for Block Cipher Modes of Operation: Galois/Counter Mode
(GCM) and GMAC - 8.3 Constraints on the Number of Invocations:
https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-38d.pdf
