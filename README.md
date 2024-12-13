# Manifest for an Asymmetric-OTP (A-OTP)

> ⚠️ This is a draft ⚠️
> 
> Example Python implementation should soon be released

One-Time Passwords (OTPs) have become increasingly popular because they enhance the security of authentication 
mechanisms without adding significant complexity for users. Often used in addition to standard passwords, OTPs enable 
Multi-Factor Authentication (MFA) simply by requiring an extra 6-digit code.

However, OTPs are not without their limitations. Despite their name, a One-Time Password is not truly "one-time" because 
it relies on a single secret key that remains constant. If this secret key is compromised, an attacker can generate 
valid OTPs at will.

The A-OTP proposal aims to address this issue while keeping the process straightforward

## Introduction of asymmetric cryptography
The conventional OTP protocol involves maintaining a secret key on both the OTP device and the OTP server:
```py
# Generate OTP on OTP device
otpSign(rounded(current_timestamp), secret_key) = OTP_candidate
send(OTP_candidate)  

# Get it on server side
receive(OTP_candidate)
# Server side holds exactly the same information. It can therefore also compute 
# the OTP on its side, and compare with the one received
verify(
    OTP_candidate
    (rounded(current_timestamp), secret_key),
)
```
The risk arises if the server's database is breached, allowing the secret_key and the current timestamp to be exploited
to generate new OTPs. This exemplifies the weakness of symmetric encryption: using a single secret for both signing and 
verifying messages

Asymmetric cryptography introduces a solution by employing two distinct keys. The public key, which is shared, 
validates the message's origin but cannot sign messages. Only the private key has the capability to authenticate 
messages

The core concept of A-OTP is to store the private key solely on the A-OTP device and only the public key on the 
authentication server (A-OTP server), which only validates but cannot generate A-OTPs:
```py
# On client side
aOtpSign(rounded(timestamp), secret_key, private_key) = AOTP_candidate
send(AOTP_candidate)

# On server side
receive(AOTP_candidate)
verify(
    AOTP_candidate
    (rounded(timestamp), secret_key, public_key),
)
```

## How to share the public key with the authenticating server?
The traditional OTP protocol necessitates a single shared secret key between the server and device.
The initial registration process operates in two steps:
1. The OTP server generates a secret key
2. The OTP device retrieves the secret key via the OTP client.

The additional security layers of the A-OTP protocol also requires: 
* private key on device side,
* and a public key on both sides

The registration process mus therefore also be updated

Moreover, the challenge with asymmetric keys is their size. Smaller keys can be easily compromised, with the private
key derivable within seconds, undermining the security of asymmetric encryption. 
The minimum secure key length is 52 alphanumeric characters*.

> A method is necessary to relay a lengthy text string to the A-OTP server during registration.

(*) Refers to an ECC 256-bit compressed key encoded in base32.

### API registration
If the MFA device is able to connect to the A-OTP server over a network, an API endpoint can be established 
to facilitate the public key transfer:
* The A-OTP server exposed on the internet
* The A-OTP server and A-OTP device are on the same network

The process is outlined as follows:
1. The A-OTP server generates a secret key
2. The A-OTP device retrieves the secret key from the A-OTP server through the A-OTP client.
3. The A-OTP device generates both a private and public key
4. The A-OTP device transmits the public key to the A-OTP Server registration endpoint.
5. The A-OTP client confirms the reception of the public key linked to the secret key

**=> This approach requires no additional manual action compared to the traditional OTP process.**

### sensors registration
When 
* the A-OTP device has a screen or a speaker
* AND the A-OTP client have a camera or a microphone

a solution is to display a QR code or play a sound code to share the public key between the A-OTP device and the 
A-OTP client.

The upgraded OTP registration process is as follows:
1. The A-OTP server generates a secret key
2. The A-OTP device retrieves the secret key from the A-OTP server through the A-OTP client.
3. The A-OTP device generates both a private and public key
4. The A-OTP device produces a QR code or a sound code to transmit the public key.
5. The A-OTP client uses its camera or microphone to capture the broadcasted public key, and sends it back to the A-OTP server.

### Manual registration
As smartphones increasingly allow text to be copied and pasted between devices, it is no longer necessary to manually 
type out the entire public key. Instead, the text can simply be copied and pasted from the A-OTP device to the A-OTP 
client interface.

The upgraded OTP registration process is as follows:
1. The A-OTP server generates a secret key
2. The A-OTP device retrieves the secret key from the A-OTP server through the A-OTP client.
3. The A-OTP device generates both a private and public key
4. The A-OTP device displays the public key as a text string, ready to by copied by the user
5. The user copies the public key text on the A-OTP client interface

### Server side registration
In scenarios where it is impractical to transmit the public key from the A-OTP device to the A-OTP server or client, 
the roles can be reversed:

1. The public and private keys are generated server-side instead of on the device.
2. The private key is shared along with the secret key.

Although there is a risk associated with displaying and transmitting the private key, this risk is comparable to that 
in the OTP protocol, where access to registration QR codes would allow an attacker to generate OTPs.

However, this risk is confined to the registration phase since only the public key is retained server-side 
post-registration. In the event of a server breach, access remains secure.

Moreover, the simplicity of the Server Side AOTP offers an identical user experience to that of traditional OTPs, 
without requiring extra steps.

NB: Some OTP registration flow are relying on a manual encoding of the secret key on the OTP device.
While it was possible with OTP because secret key are often rather short, A-OTP with server side registration
totally discard that option because of the size of the private key to be shared together with the secret key.


### Backward compatible registration
TODO

## Registration Protocols and processes

> I want to play around with 2FA codes. So, I started looking for the specification. Turns out, there isn't one. Not really.
> 
> — https://shkspr.mobi/blog/2022/05/why-is-there-no-formal-specification-for-otpauth-urls/
 
The suggestion is to start from the [Google archived draft proposal](https://github.com/google/google-authenticator/wiki/Key-Uri-Format)
and to provide two versions of the registration URI
* an OTP backward compatible version, using the `totp` type
* a A-OTP dedicated version, using the new `taotp` type

### Aotp registration URI query parameter
**REQUIRED** `aotp` parameter is a comma separated list of A-OTP registration supported mechanism.
  * `none` when the user is allowed to fall back on standard OTP, and no asymmetric key used. 
     The value is mandatory in the list of backward compatible URI. 
  * `server` for server side private & public key generation.
  * `qrcode`,
  * `soundcode`, 
  * `manual`, 
  * `api`

### Registration endpoint URI query parameter
**REQUIRED WHEN** `api` is in `aotp`, the `registration_endpoint` 
parameter points at the API endpoint the A-OTP device should use to share the
public key.

The protocol of that endpoint is defined bellow

### Private key URI query parameter
**REQUIRED WHEN** `server` is in `aotp`, the `private_key` query parameter
is a base64 encoded version of the private key generated on the server side.

### Registration UI address URI query parameter
**MAY BE PROVIDED WHEN** `qrcode`, `soundcode`, `manual`, are in the list,
or when `server` is used together with another value.

### Algorithm URI query parameter
**OPTIONAL**: The algorithm may have the values:
* `SHA1` (Default for backward compatible URI)
* `SHA256` (Default)
* `SHA512`

### Digits URI query parameter
**OPTIONAL**, The `digits` parameter may have the values 6 or 8, and determines how long of a one-time passcode to 
display to the user. The default is 6.

### Letters URI query parameter
**OPTIONAL**, the `letters` parameters is a preferred name for the `digits` parameter as 
the A-OTP may not only be made of digits (see encoding).

Backward compatible A-OTP should still preferably use the `digits` parameter
as `letters` is unknown to OTP specification

### Encoding URI query parameter
**OPTIONAL** the `encoding` parameter may 

## Authentication specifications

### A-OTP generation
The generation of an A-OTP comes in three phases
1. Issue a classic OTP
2. Generate a signature of the OTP using the private key
3. Use the last bits of the signature to generate a code of `digits` letters using the `encoding` alphabet


