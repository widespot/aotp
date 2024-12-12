# Manifest for an Asymmetric-OTP (A-OTP)

One-Time Passwords (OTPs) have become increasingly popular because they enhance the security of authentication 
mechanisms without adding significant complexity for users. Often used in addition to standard passwords, OTPs enable 
Multi-Factor Authentication (MFA) simply by requiring an extra 6-digit code.

However, OTPs are not without their limitations. Despite their name, a One-Time Password is not truly "one-time" because 
it relies on a single secret key that remains constant. If this secret key is compromised, an attacker can generate 
valid OTPs at will.

The A-OTP proposal aims to address this issue while keeping the process straightforward

## Introduction of asymmetric cryptography
Standard OTP protocol relies on a secret key kept on both sides: the OTP device and the OTP server
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
It is easy to see that if the database on the server side leaks, the combination of the secret_key and the current 
timestamp can then be used to generate new OTP. That's the principle of symmetric encryption:
of one single secret to both sign and verify a message

Asymmetric cryptography provides a way to overcome that issue as it works with two distinct secrets.
The shared **public key** is used to confirm the origin of a message but it cannot be used to signe it.
Only the **private key** can issue authenticated messages.

The idea of A-OTP is therefore to keep a private key on the A-OTP device,
but only the public key on the authenticating server (the A-OTP server). It can therefore only
confirm the validity of an A-OTP, but in case of leak, it won't be able to generate it
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
The OTP protocol only requires
* one secret key to be shared on server and device side

and works in two steps
1. OTP server generates a secret key
2. OTP device gets the secret key from the OTP server via the OTP client.

The A-OTP process requires an additional 
* private key on device side,
* and a public key on both sides

It must therefore also be additional steps

Also, the tradeoff of asymmetric keys is their size. Small keys
are easy to revert: the corresponding private key of a public key
can be found in a matter of seconds, breaking the advantage of asymmetric encryption.
The smallest secure key is 52 alphanumeric character long*

> There must be a way to share a long text string with the
> A-OTP server at registration time.

(*) ECC 256 bits compressed key expressed in base32.

### API registration
When the MFA device has a network connectivity to the **A-OTP server**, an API endpoint
can be exposed to receive the public key.
* A-OTP server exposed on the internet
* A-OTP server and A-OTP device are on the same network

The process is as follows:
1. A-OTP device gets the secret key from the A-OTP server via the A-OTP client. (=OTP registration process)
2. A-OTP device generates a private and public key
3. A-OTP device sends the public key to the A-OTP Server registration endpoint
4. A-OTP client confirms the reception of the public key related to the given secret key

**=> No additional manual action compared to OTP process**

### sensors registration
Manually typing a public key may not be foreseeable, but there are other
ways to transfer data to the graphical user interface of the A-OTP server.

* **Using camera or even microphone**: the sensors of the device used to browse the
  graphical user interface of the A-OTP servers can be used to transfer data. The A-OTP
  device may display a QR-code to be scanned by the A-OTP client, or using
  a coded sound message to be captured by the microphone of the A-OTP client.

The upgraded OTP registration process is as follows:
1. A-OTP device gets the secret key from the A-OTP server via the A-OTP client. (=OTP registration process)
2. A-OTP device generates a private and public key
3. A-OTP device generates a QR code or a sound code to broadcast the public key
4. A-OTP client watches (camera) or listen (microphone) to the broadcasted public key

### Manual registration
More and more smartphones will allow copy-pasting text with paired computers.
It is therefore no more needed to manually type every single letter of the public key,
but only to copy-paste te text between the A-OTP device and the A-OTP client.

The upgraded OTP registration process is as follows:
1. A-OTP server generates a secret key
2. A-OTP device gets the secret key from the A-OTP server via the A-OTP client.
3. A-OTP device generates a private and public key
4. A-OTP device generates displays the public key as a text string
5. user copies the public key string on the A-OTP client interface

### Server side registration
When there is really no way to share the public key from the A-OTP device to the A-OTP server or the A-OTP client,
the solution is to reverse the flow. 

1. The public and private keys are generated on the server side instead of the device side
2. the private key is shared, embedded with the secret key in the registration qr-code

There is of course a risk related to the transmission & display of the private key, but the risk
is the same with the OTP protocol. Having access the registration QR codes means being able 
to generate OTP. 

Still, the big difference is that here the risk is only limited to the registration phase since
only the public key is kept on server side. In case of server side leak, the access is not compromised.

Also, the big advantage is the simplicity: the user experience of the Server Side AOTP
is exactly the same as the OTP. No extra step.

NB: this flow is only possible when the A-OTP device have a camera to scan a QR code,
or a way to recieve the 

### Backward compatible registration

## Specifications

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

### Registration endpoint parameter
**REQUIRED WHEN** `api` is in `aotp`, the `registration_endpoint` 
parameter points at the API endpoint the A-OTP device should use to share the
public key.

The protocol of that endpoint is defined bellow

#### Private key parameter
**REQUIRED WHEN** `server` is in `aotp`, the `private_key` query parameter
is a base64 encoded version of the private key generated on the server side.


* `registration_ui`, optional when `qrcode`, `soundcode`, `manual`, are in the list,
  or when `server` is used together with another value.


