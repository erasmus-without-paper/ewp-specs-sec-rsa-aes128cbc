`ewp-rsa-aes128cbc` Encryption
==============================

This document describes a format which can be used to transfer encrypted binary
payload to a single recipient, with help a symmetric AES-128 key and the
recipient's public RSA key.

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
   different techniques for achieving these tasks).

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

   Java example:

   ```java
   KeyGenerator keyGen = KeyGenerator.getInstance("AES");
   keyGen.init(128);
   SecretKey aesKey = keyGen.generateKey();
   ```

 * In order to safely transfer the `aesKey` to the recipient, it is encrypted
   with `recipientPublicKey`, resulting in `encryptedAesKey`. ECB mode and
   PKCS#1 v1.5 padding are used. The sender MAY cache the resulting
   `encryptedAesKey` (per recipient, as with `aesKey`).

   Java example:

   ```java
   Cipher cipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
   cipher.init(Cipher.ENCRYPT_MODE, recipientPublicKey);
   byte[] encryptedAesKey = cipher.doFinal(aesKey.getEncoded());
   ```

 * In order to allow the `aesKey` to be safely reused, the sender also
   generates a random salt (initialization vector) `iv` for every message.
   AES-128 requires 16-byte-long initialization vector.

   Java example:

   ```java
   SecureRandom rnd = new SecureRandom();
   byte[] iv = new byte[16];
   rnd.nextBytes(iv);
   ```

 * Payload `payload` is then encrypted with `aesKey` and `iv`, resulting in
   `encryptedPayload`.

   Java example:

   ```java
   Cipher cipher = Cipher.getInstance("AES/CBC/PKCS5Padding");
   IvParameterSpec ivParameterSpec = new IvParameterSpec(iv);
   cipher.init(Cipher.ENCRYPT_MODE, aesKey, ivParameterSpec);
   encryptedPayload = cipher.doFinal(originalContent);
   ```

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

 * `iv` (16 bytes)

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

We will use following RSA key-pair for this example:

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
A1ATd09ZbhiHNEvaigZGIDB1lZI1XbP1HISY/9Cxit0BAHAwvLBFwG6YTgHbIA/dKOnnY81GEqxf
EGH8fcvsXqpFQAUxRcEB9OTsO/lKHBlnBLt9zR/C65J6k+Cd8N8QUBb+sDMaXxsszvFGQ1dLjSxE
O5edZ8b6PX+aLUdOhLTbM3RsOi01NCmeU8QqyJbEK1j627dB9bDUVrsEaghq/IwA/uMx3Kjs0skb
HfgmR0cMmdCG2dHokD80H5pt1amwXD7yMPRnEtY/esfryj1iDhBqL6y2XT0EykfGsjUMhyUgnTVU
6bm2fsHTxxrSZjBkQ8padn/LuwkGuq/vsGnvNKkl6Bk0yYHO7KDjikL1nIEi2MbyWjX+6ELCF4K4
lo+eReqBF8ll49q0y6gZ/fmWw+862ChihE/23BTu/kLbaZFgGsXZAc/FdOciiazrybFqvqw=
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
cDC8sEXAbphOAdsgD90o6edjzUYSrF8QYfx9y+xeqkVABTFFwQH05Ow7+UocGWcEu33NH8LrknqT
4J3w3xBQFv6wMxpfGyzO8UZDV0uNLEQ7l51nxvo9f5otR06EtNszdGw6LTU0KZ5TxCrIlsQrWPrb
t0H1sNRWuwRqCGr8jAD+4zHcqOzSyRsd+CZHRwyZ0IbZ0eiQPzQfmm3VqbBcPvIw9GcS1j96x+vK
PWIOEGovrLZdPQTKR8ayNQyHJSCdNVTpubZ+wdPHGtJmMGRDylp2f8u7CQa6r++wae80qSXoGTTJ
gc7soOOKQvWcgSLYxvJaNf7oQsIXgriWj55F6g==

// decrypted aesKey (base64)
75rXSczDfUCrt8RMoED4+Q==

// iv (base64)
gRfJZePatMuoGf35lsPvOg==

// offset of encryptedPayload
306

// encryptedPayload (base64)
2ChihE/23BTu/kLbaZFgGsXZAc/FdOciiazrybFqvqw=

// length of decryptedPayload
17

// decryptedPayload (base64)
VGhpcyBpcyBhIHNlY3JldC4=

// decryptedPayload (UTF-8)
This is a secret.
```


[develhub]: http://developers.erasmuswithoutpaper.eu/
[statuses]: https://github.com/erasmus-without-paper/ewp-specs-management/blob/stable-v1/README.md#statuses
