# Cryptographic documentation

## Protocols and cryptographic tools used

Sealed-secrets uses the following protocols for the secret management:

- **AES-256-GCM** with a randomly generated single-use 32 bytes session key. Since the key is single-use, we do not use any nonce. The key is used to encrypt the secret, ensuring its confidentiality and integrity.
- **RSA-OAEP**, with **SHA-256**. It is used to assure the confidentiality of the AES-256-GCM session key, following the *key encapsulation mechanism*.
- **X509** certificates are used to manage RSA public keys. This public key contained in the certificate can be used to encrypt AES-256-GCM session key.

Certificates generated by the sealed secrets controller are renewed every 30 days and have a 10 years validity span.

## Entropy considerations

The golang API used for the entropy is `crypto/rand`. The following description can be found about the entropy generator used regarding the host system:

```
// On Linux, FreeBSD, Dragonfly and Solaris, Reader uses getrandom(2) if
// available, /dev/urandom otherwise.
// On OpenBSD and macOS, Reader uses getentropy(2).
// On other Unix-like systems, Reader reads from /dev/urandom.
// On Windows systems, Reader uses the RtlGenRandom API.
// On Wasm, Reader uses the Web Crypto API.
```

Those cryptographic APIs are known to provide a good cryptographic entropy, and are not vulnerable to cryptographic attacks unless the seed is known.

For further information about those APIs:

- [Linux/FreeBSD/Dragonfly/Solaris](https://linux.die.net/man/4/urandom)
- [OpenBSD/macOS](https://www.freebsd.org/cgi/man.cgi?query=getentropy&sektion=3&format=html)
- [Windows](https://download.microsoft.com/download/1/c/9/1c9813b8-089c-4fef-b2ad-ad80e79403ba/Whitepaper%20-%20The%20Windows%2010%20random%20number%20generation%20infrastructure.pdf)
- [WASM](https://developer.mozilla.org/en-US/docs/Web/API/Crypto/getRandomValues)

## Functioning

### Public/private key pair management

The controller looks for a cluster-wide private/public key pair on startup. If no key pair is found and none is provided manually, the controller generates a new 4096 bit (by default) RSA key pair. In both cases, the key pair is persisted in a regular Secret in the same namespace as the controller.

The public key (in the form of a self-signed certificate if it was generated by the controller) should be made publicly available to anyone wanting to use SealedSecrets with this cluster.

Note that it is possible to use your own X509 certificate with the command bellow:

```
kubeseal --cert [https:/]/path/to/your-cert.pem
```

The certificate is printed to the controller log at startup and is also available via an HTTP GET request to `/v1/cert.pem` on the controller.

### Secret encryption

The secret is encrypted by AES-256-GCM with a randomly-generated single-use 32 byte session key.

The result of this operation will be called `AES encrypted data` in the next diagram, and the present step is the `1.`.

### Session key encryption

The session key used by AES-256-GCM to encrypt the Secret is encapsulated with the controller's public key using RSA-OAEP with SHA256.

The OAEP input content, called `label` in the next diagram, differs depending on the sealed secret controller scope configuration. This algorithm is only used to encrypt the AES session key.

- Default scope configuration : `label` is equal to the concatenation of the Secret's namespace and the Secret's name.
- Namespace-wide scope configuration : `label` is equal to the Secret's namespace.
- Cluster-wide scope configuration : `label` is empty.

The result of the RSA-OAEP encryption is called `RSA encrypted data` in the next diagram, and the present step is the `2.`.

### Sealed Secret storage

The final Sealed Secret data format is the following (where `||` is the concatenation operator): `size of AES encrypted key (2 bytes) || RSA encrypted data || AES encrypted data`

### Diagram to summarize

```
								Secret
                                                                   |
                                                                   │
                                                   K_s────────────►│
                                                    │              │
                                       K_pub───────►│              │
                                                    │              │ 1.
                                       label───────►│ 2.           │
                                                    │              │
                     ┌──────────────────────┬───────▼───────┬──────▼───────┐
Sealed Secret data = │size of AES encrypted │ RSA encrypted │ AES encrypted│
                     │key (2 bytes)         │ data          │ data         │
                     └──────────────────────┴───────────────┴──────────────┘

K_s = 256 bits single-use session key, used by AES-GCM
K_pub = Public key from the self-signed certificate, used by RSA-OAEP
label = Additional input for RSA-OAEP encryption.
        Content differs depending on the scope configuration:
         * Default config : label = Secret's namespace || Secret's name
         * Namespace-wide : label = Secret's namespace
         * Cluster-wide : label is empty
```

### Decryption process

The decryption is simply the inversion of the encryption.

`Size of AES encrypted key` is read and used to separate `RSA encrypted data` and `AES encrypted data` properly.

Then the private key associated with the public key (see Session key encryption) is used with the `label` to decrypt the `RSA encrypted data`, effectively retrieving the AES session key.

To end this process, the `AES encrypted data` is decrypted using the AES session key, therefore unsealing the original Secret.

# Post-quantum cryptography considerations

## Entropy source

### Analysis

Even if QRNG (Quantum Random Number Generator) are considered better than PRNG (Pseudo Random Number Generator) in a quantum cryptography context as well as in a non-quantum context, QRNG relies on a quantum mechanical phenomenon. It requires a physical device, therefore QRNG usage is out of Sealed Secrets scope, which will stay on the `crypto/rand` usage.

### Associated documentation

[Combining a quantum random number generator and quantum-resistant algorithms into the GnuGPG open-source software](https://doi.org/10.1515/aot-2020-0021)

## AES-256-GCM

### Analysis

AES-256-GCM is quantum resistant.
Grover algorithm can reduce the bruteforce of the key from 2²⁵⁶ to 2¹²⁸ which is still considered very secure.
Nevertheless, since AES uses unchangeable 128 bits blocks, Grover algorithm can in some cases decrease the complexity of the bruteforce to 2⁶⁴.

### Recommendations

AES-256-GCM quantum security is not a concern.
Cases with a bruteforce complexity of 2⁶⁴ are unlikely for Sealed Secret considering how AES is used in the project.
Even assuming that 2⁶⁴ bruteforce is likely, it can still be considered secure today (but not in the long run).
A recommendation is to look for a AES replacement that provide 128 bits post-quantum cryptographic security in any cases, such as ChaCha20-Poly1305. Applying this recommendation is considered low priority.

### Associated documentation

[Quantum Security Analysis of AES](https://eprint.iacr.org/2019/272.pdf)

[Critics on AES-256-GCM](https://soatok.blog/2020/05/13/why-aes-gcm-sucks/)

[Security Analysis of ChaCha20-Poly1305 AEAD](https://www.cryptrec.go.jp/exreport/cryptrec-ex-2601-2016.pdf)


## SHA-256

### Analysis

SHA-256 is quantum resistant.
Grover Algorithm can reduce the bruteforce from 2²⁵⁶ to 2¹²⁸ which is considered very secure.
It is computationally cheaper to use a non-quantum algorithm to generate a collision than to employ a quantum computer.

### Recommendations

No recommendations about SHA-256.

### Associated documentation
[Cost analysis of hash collisions: Will quantum computers make SHARCS obsolete?](https://cr.yp.to/hash/collisioncost-20090823.pdf)

## RSA-OAEP

### Analysis

RSA-OAEP, as any RSA algorithm, **is not quantum resistant**.
Shor algorithm can be used to solve in a reasonable time 3 mathematical problems on which RSA cryptography is based on: integer factorization problem, the discrete logarithm problem and the elliptic-curve discrete logarithm problem. Therefore, RSA-OAEP is easily breakable for an attacker with quantum capability.

### Recommendations

Replace RSA. This recommendation must be the highest priority regarding the post-quantum security of Sealed Secrets.
There are three serious candidates to use instead of RSA: LMS and XMSS, which are Lattice-based, and McEliece with random Goppa codes, which is code-based and relies on SDP (Syndrome Decoding Problem).
Those three algorithms are serious candidates for RSA replacement and the choice must be done carefully, without forgetting to study other algorithms such as NTRU.

### Associated documentation

[LMS](https://datatracker.ietf.org/doc/html/rfc8554)

[XMSS](https://datatracker.ietf.org/doc/html/rfc8391)

[Lattice-based cryptography](https://en.wikipedia.org/wiki/Lattice-based_cryptography)

[McEliece](https://ipnpr.jpl.nasa.gov/progress_report2/42-44/44N.PDF)

[Syndrome Decoding Problem](https://en.wikipedia.org/wiki/Decoding_methods#Syndrome_decoding)

[NIST on post-quantum algorithms](https://csrc.nist.gov/Projects/post-quantum-cryptography/round-3-submissions)

[Quantum-Resistant Cryptography](https://arxiv.org/ftp/arxiv/papers/2112/2112.00399.pdf)

