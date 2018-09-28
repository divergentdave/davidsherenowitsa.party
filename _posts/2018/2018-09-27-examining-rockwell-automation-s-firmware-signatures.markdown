---
layout: post
title: Examining Rockwell Automation's Firmware Signatures
date: '2018-09-27 21:58'
---

# Background

Industrial automation products have long been behind the curve when it comes to computer security. Default passwords, backdoor passwords, unpatched vulnerabilities, and network protocols with no authentication have been the norm for years. Rockwell Automation, one of the more established vendors in the market, took a significant step forward in 2015 by adopting signed firmwares.

According to [a Rockwell Automation slide deck][powerflex-update-2015], firmware signatures were first introduced in their PowerFlex 750 series of variable frequency drives. The motivation for this change wasn't computer security concerns, but rather compliance with nuclear non-proliferation treaties. According to the presentation, the Department of Commerce updated export regulations on variable frequency drives with output frequencies of 600Hz or greater. (ten or twelve times the frequency of the electrical grid) Rockwell Automation addressed this by capping the output frequency in a software update, so that the drives weren't subject to harsher export restrictions. This change was paired with a new firmware signature mechanism, which would enforce that only firmware images authored by the manufacturer could be flashed onto devices. Following this feature's development for the PowerFlex 750 line, firmware signatures were rolled out to most other product lines to improve their security.

# Signed Firmware

Each firmware package consists of a configuration file (with extension `.nvs`) and one or more binary files. In a signed firmware package, half of the binary files that get flashed to the device are DER-encoded X.509 certificates, with a `.der` file extension. In a typical public key infrastructure, one would expect that an X.509 certificate would be used to identify a principal that makes signatures, but in this system, the certificate file acts as the firmware signature itself, with the help of some unusual vendor-specific extensions, described below.

# Extensions

Each certificate file contains seven extensions, with OIDs `1.3.6.1.4.1.95.3.1` through `1.3.6.1.4.1.95.3.7`. The prefix of `1.3.6.1.4.1` is assigned to private enterprises, and `1.3.6.1.4.1.95` in particular is assigned to Allen-Bradley, which is a division of Rockwell Automation. The extension with OID `1.3.6.1.4.1.95.3.1` is particularly interesting, because it contains a `BIT STRING` of the SHA-1 hash of the corresponding firmware image.

Extensions `1.3.6.1.4.1.95.3.2` through `1.3.6.1.4.1.95.3.6` hold `INTEGER`s which might be related to firmware versions, product identifiers, or other metadata. Extension `1.3.6.1.4.1.95.3.7` holds a `BIT STRING` of unknown purpose.

# Subject and Issuer

The issuer name of each certificate is `O=Rockwell Automation`. The subject name of each certificate has an `organizationName` field of `Rockwell Automation`, and `organizationalUnitName` and `commonName` fields that vary by product.

# Certificate Signature

Although the distinguished names of the subject and the issuer in each certificate are different, the public keys of the subject and the issuer are identical. From a chain building perspective, these are leaf certificates, but from a cryptographic perspective, these certificates are self-signed. This mismatch suggests that the certificates aren't being used in a traditional PKI, and the bootloader isn't building a chain of certificates back to a root certificate when it verifies firmware signatures.

# Tools for Certificate Signatures

The subject's public key is encoded directly in the certificate body, but the issuer's public key does not appear anywhere in an X.509 certificate, as it's assumed to be known beforehand. However, the signature over the to-be-signed portion of the certificate is made with the issuer's public key, so we can guess that the issuer's public key is the same as the subject's public key, try to verify the signature, and check if our guess was correct or not. I wrote a Python script to parse certificates and perform this verification and [posted it as a gist][verify-self-signed-key].

If we hadn't guessed that the certificates were self signed, we could have used a more generic approach to recover the issuer's public key from any two certificates it signed. If we make a guess for the public exponent, raise each signature to the power of the public exponent, and subtract each padded plaintext message from the result, then the resulting differences will be multiples of the public modulus. Next, we compute the greatest common divisor of the two differences, and then divide out any small factors from the greatest common divisor to get a candidate public modulus. Finally, we check if the signatures are valid using this public key to determine if our public modulus guess was correct. I wrote another Python script to perform this calculation and [posted it as a gist too][recover-pubkey].

# Algorithms

Unfortunately, the firmware signature certificate files use SHA-1 and 1024-bit RSA keys. Neither of these choices would pass muster in the Web PKI today, nor would either have been a best practice back in 2015. Nation-state attackers might take advantage of novel attacks to forge firmware signatures, like how the US and Israel used an attack on MD5 signatures to forge Windows Update certificates. I would recommend migrating to SHA-256 instead of SHA-1, and either 2048-bit RSA or ECDSA. Of course, firmwares built after such a migration would have to reject signatures by the 1024-bit RSA key and signatures that use SHA-1 for this to be effective.

# Serial Number

The certificates appear to gave high-entropy serial numbers, which is good for security. About half of the certificates have a negative serial number, which isn't allowed by the relevant RFCs. As a result, some tools may not work with half of these files.

# Conclusion

Based on these observations, it seems that Rockwell Automation's firmware signatures are the product of custom, proprietary tools. By including the hash of a firmware image in a certificate extension, they have repurposed X.509 certificates as a kind of detached signature with extra signed metadata. As far as I can tell, this signature scheme should be secure, but the outdated algorithms are concerning, and the reliance on self-signed certificates (with no way to provide a cross-signed certificate chain) will make key rotation difficult.

[powerflex-update-2015]: https://controltech.cz/sk/k-stiahnutiu/category/17-summer-days-2015-sk?download=204:powerflex-update-2015
[verify-self-signed-key]: https://gist.github.com/divergentdave/7cd98ee16919a1a159ded8bc3160c8da
[recover-pubkey]: https://gist.github.com/divergentdave/40ac9c7224b382166c905e76595bcf73
