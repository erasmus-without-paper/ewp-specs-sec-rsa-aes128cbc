`ewp-rsa-aes128gcm` Encryption
==============================

This document describes a format which can be used to transfer encrypted binary
payload to a single recipient, with help of symmetric AES-128 GCM encryption
and the recipient's public RSA key.

* [What is the status of this document?][statuses]
* [See the index of all other EWP Specifications][develhub]


Introduction
------------

This specification is the result of the discussion in
[this GitHub thread](https://github.com/erasmus-without-paper/ewp-specs-architecture/issues/26).
Here's a quick summary of this thread:

 * We needed a simple way to encrypt data for a **single recipient**, given his
   RSA public key.

 * **We didn't** need to authenticate the sender, nor to verify the integrity
   of the data being sent (because EWP Network already made use of some
   different techniques for achieving these tasks). Still, we preferred GCM
   over CBC, for security reasons (see
   [here](https://github.com/erasmus-without-paper/ewp-specs-sec-rsa-aes128cbc/issues/1)).

 * We didn't find any existing standards which would satisfy these requirements
   and - at the same time - would fit neatly with our architecture and be
   easily implementable in the software stacks used by EWP partners. We didn't
   have much time to investigate and browse through all the existing standards,
   so we decided to prepare our own.

To summarize: It's not an RFC standard, but it's simple, and it works. EWP
Network makes use of this format in some of its specifications regarding HTTP
request and response encryption. If you want to use this encoding outside of
EWP Network, you are free to do so.


Encryption
----------

 * First, the sender needs to acquire the RSA public key `recipientPublicKey`
   of the recipient. (This document doesn't specify how this is done.)

 * The sender generates a new 128-bit AES key `aesKey` which will be used for
   payload encryption. `aesKey` can be either random, or the sender MAY cache
   it *per recipient*.

   Note, that it is very important to NOT use the same AES key for *different*
   recipients. In case of any doubt, it's safer to generate a new one every
   time.

 * In order to safely transfer the `aesKey` to the recipient, it is encrypted
   with `recipientPublicKey`, resulting in `encryptedAesKey`. ECB mode and
   PKCS#1 v1.5 padding are used. The sender MAY cache the resulting
   `encryptedAesKey` (per recipient, as with `aesKey`).

 * In order to allow the `aesKey` to be safely reused, the sender also
   generates a random 12-byte-long salt (initialization vector) `iv` for every
   message.

 * Payload `payload` is then encrypted with AES GCM mode with `aesKey` and
   `iv`, resulting in `encryptedPayload` (which also already contains the GCM
   Authentication Tag).

 * The final `ewpRsaAesBody` is then constructed, in the format described in
   the section below.


`ewpRsaAesBody` format
----------------------

`ewpRsaAesBody` is a sequence of the following binary sections:

 * `recipientPublicKeyFingerprint` (32 bytes)

   SHA-256 fingerprint of the `recipientPublicKey`. It can be used by the
   recipient to retrieve a valid decryption key from its key store.

 * `encryptedAesKeyLength` (2 bytes)

   Two-byte-long big-endian unsigned integer. This integer is the length of the
   `encryptedAesKey` which follows (in bytes).

 * `encryptedAesKey` (`encryptedAesKeyLength` bytes)

   The value of `encryptedAesKey`, as described in the *Encryption* section
   above.

 * `iv` (12 bytes)

   The value of the initialization vector `iv`, as described in the
   *Encryption* section above. Note, that it doesn't need to be encrypted.

 * `encryptedPayload`

   All the remaining bytes in `ewpRsaAesBody` are the value of the
   `encryptedPayload`, as described in the *Encryption* section above.


Decryption
----------

 * Read `recipientPublicKeyId` and use it to find a proper RSA key-pair to use
   for decryption. (If you use only one RSA key-pair, then you MAY also skip
   this step and assume that the message is encrypted for this particular key.)

 * Read `encryptedAesKeyLength`, and then `encryptedAesKey`. Decrypt
   `encryptedAesKey` your private RSA key, and retrieve `aesKey`.

 * Read `iv` and `encryptedPayload`. Use `aesKey` and `iv` to decrypt
   `encryptedPayload` and retrieve the original `payload`.


Test values
-----------

### Recipient's credentials

We will use the following RSA key-pair in this example:

```
// recipientPublicKey
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAl8UYbSjdXHIkDCzbO4Rn9c/kb4xm3e2A
3WSWQ/WtoTqrEy9/nJNe7ehP9gaUrbUf9KCMF0fHypQ6oVasPr/6zCiHV0LDwiImWbbo+9zehT4f
oahcdRqPqfEVeRJgMr7hfvcrvtzFO7yFLHkYkt4r4MLy3GIkSys2arr3vs8V/JY826qTlIi4tLif
eeY9I6adMDXPtAoHr9er2XdOt4LWuLOxqHpddrnuSJhnlY0I1mz8b9LCoN15gwW9d1G/H1KFmIkW
u0ty205XUW5c/YRXxjK1WZcXlfobTyVyeeK2yeEeXdKRm0zuIlBruij3VZ+/dmPXdtn0zGlStOS5
UhjSNQIDAQAB

// recipientPrivateKey
MIIEvgIBADANBgkqhkiG9w0BAQEFAASCBKgwggSkAgEAAoIBAQCXxRhtKN1cciQMLNs7hGf1z+Rv
jGbd7YDdZJZD9a2hOqsTL3+ck17t6E/2BpSttR/0oIwXR8fKlDqhVqw+v/rMKIdXQsPCIiZZtuj7
3N6FPh+hqFx1Go+p8RV5EmAyvuF+9yu+3MU7vIUseRiS3ivgwvLcYiRLKzZquve+zxX8ljzbqpOU
iLi0uJ955j0jpp0wNc+0Cgev16vZd063gta4s7Goel12ue5ImGeVjQjWbPxv0sKg3XmDBb13Ub8f
UoWYiRa7S3LbTldRblz9hFfGMrVZlxeV+htPJXJ54rbJ4R5d0pGbTO4iUGu6KPdVn792Y9d22fTM
aVK05LlSGNI1AgMBAAECggEANsqlIuOZ5wIeGXcoPrhyf7/qDIt3p69S0pq51Rcg9BAmKur++xwJ
LYKtO3jsvDmjq8E6Uj1L18rjz9Nmo9DTTlljYxFrcu65QbJTMnpuq1PeP5J0rqJEM2oiAm+r4yYe
aqP5WxKA8iwBOCkPwhYLaT14SC/2Qlz7bFTLlEtW+LUEb2w9InThtEvIoNK2+n5BSRrjXH46dvJX
28dsWft2f5o3+PX8L6z0qNOyKKgOHbBlw37IbV78GhdI0dLFEGEcMkdV+p2bqChVxiu3S8G/0fwj
l6HwhUYD5OOWg3ncWllB9rLdJNA2zYFfnwsycrgEh1vk0zp0S9cbuoOyc6h+AQKBgQDm6aEHVZ3c
+ZIB5jwgqX9tK2Zr5bdyptqpAfXgDf9Sj34Zg5+0pnA7v8zRlg2rqwE50XRSEiVxpBDdMswRe30S
ioN1zHeabROuSAd4EHMkrluSmcObeP3LcStj5D0VAmbijAgYnRKiShGRvUSVs42pMyxHo+RYTiXk
0/tg3hrelQKBgQCoQkVPqw1Dlv8BKDU284ogqVCTNDqz96rsGNKUmnRR4UVEJZGXOVvfDMt3kYr5
6QCmDhyGf6ny6W/YgjNJHDbz3LAl9DLc/8d4y8FgXaBqwPpvW2aBL3tea8Raa7+D3t/Z68sJduue
JO921E72n+xa6TvYModtE7cIwmfMcy5dIQKBgQCO8Mbe3HAJj3CDvnswGNypvrj7R8uErKclAfKr
jN6lw+/iaWlekb1eLz/h6cpynzv2B6PC/jqxm0dZNo2+sLve02HHdRgAv070juAYwc4VQd2r5YWB
46bv3hFnF618KO15hgeo/OrBDarMleYz6V9jAyuA+YJr64xnl5XABB2L9QKBgDzlVz6BMti+gmZB
zhioRdqSTNYp9gECZvrx9OzRhb3IoRAL5MhtewGcGNuackkGejSfMNXAyJpgwBkE7ljMfFsACUSD
QBFaBTCD1eXxnMhmNX0uAEhLDgRbToJHMtYgSLYPL7mqL5ZZ2c0RA88gjCNO/Fi/2OGyW/Ewou6M
1T/hAoGBAJq1J6wOR05lShFYwyev0rxGeZCx8bpWBEXwuzPJFZCQxyym9ZexcArI/74a2Fb4Esgx
nwORksuBVJDZUG7aL05xiW1MJrcHyh9DDo47T2Qo5k4NngXNBmOhjbV6cmzJBaKY9j3B/bJmWhIL
BiSeGLYpa/SUIFhQuMLqwb/2pKSa
```


### ewpRsaAesBody

The following `ewpRsaAesBody` contains a correctly encrypted UTF-8 encoded
string `This is a secret.` (with the dot). The `ewpRsaAesBody` is in fact
binary. Here, it is presented in base64-encoded format, for presentation
reasons.

```
A1ATd09ZbhiHNEvaigZGIDB1lZI1XbP1HISY/9Cxit0BAFEd8dtFveIlZxgNb7Wl/hYWwMJ55jUp
yyvFUNMoxGQF5fWPX8zWlltIB1xD/yuVwK2Whk6Nz1j5qqJ9FSWJPqdTx3438R0vk7Skh2Zo1s5U
FJX/IdgcwEY3s/CQuHN1BUOWDCeweZowCA6Npvhx8hnQOWcr8x/FCagjDDrmSAknz5RZiyVW2xw4
SyI/w3RXgEbo402faq/eMwcj4yIc2gm1KJa8qM3gznLMUP2qYzjhcrRTumVx6laiRYvS315MGMUP
tsqiV3ofqRHI4u3ugWLYhKZesO5YnFaRVvS+05d4lqUYCxUzJT2QD2NIGeeeG6a8lNoaG41TRd1s
FgfzeNS3z8jfltABwbpileS0ixTlHY5OjGNSojZX+8XI0C4CsCQVUrhMsjROTlWjnI0=
```


### Intermediate values

These are the intermediate values which should come up during the decryption
process. All binary values are presented in base64-encoded format, for
presentation reasons.

```
// recipientPublicKeyFingerprint (base64)
A1ATd09ZbhiHNEvaigZGIDB1lZI1XbP1HISY/9Cxit0=

// encryptedAesKeyLength
256

// encryptedAesKey (base64)
UR3x20W94iVnGA1vtaX+FhbAwnnmNSnLK8VQ0yjEZAXl9Y9fzNaWW0gHXEP/K5XArZaGTo3PWPmq
on0VJYk+p1PHfjfxHS+TtKSHZmjWzlQUlf8h2BzARjez8JC4c3UFQ5YMJ7B5mjAIDo2m+HHyGdA5
ZyvzH8UJqCMMOuZICSfPlFmLJVbbHDhLIj/DdFeARujjTZ9qr94zByPjIhzaCbUolryozeDOcsxQ
/apjOOFytFO6ZXHqVqJFi9LfXkwYxQ+2yqJXeh+pEcji7e6BYtiEpl6w7licVpFW9L7Tl3iWpRgL
FTMlPZAPY0gZ554bpryU2hobjVNF3WwWB/N41A==

// aesKey (base64)
Gwaty5wYxuD81f++z6jwZw==

// iv (base64)
t8/I35bQAcG6YpXk

// offset of encryptedPayload
302

// encryptedPayload (base64)
tIsU5R2OToxjUqI2V/vFyNAuArAkFVK4TLI0Tk5Vo5yN

// length of decryptedPayload
17

// decryptedPayload (base64)
VGhpcyBpcyBhIHNlY3JldC4=

// decryptedPayload (UTF-8)
This is a secret.
```


[develhub]: http://developers.erasmuswithoutpaper.eu/
[statuses]: https://github.com/erasmus-without-paper/ewp-specs-management/blob/stable-v1/README.md#statuses
