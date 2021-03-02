# Blackwood 4NT
`Blackwood 4NT` -- Grand Slam Authentication for Windows NT (10)

## Introduction

`Blackwood 4NT` is a simple `C/C++`-based Windows library for communicating with Grand Slam Authentication (`GSA`) and obtaining the Server Provided Data (`spd`) that contains, among other things, the Password Equivalent Token (`PET`), which can then be used for further API authentication when combined with the user's Alternate Directory Services Identifier (`ADSID`/`AltDSID`).

## Components

Multiple pieces of technology had to be created in order to enable `Blackwood 4NT` to function. 

* Successful communication with `GSA` requires an implementation of Secure Remote Password, Version 6a (`SRP-6a`)[http://srp.stanford.edu/design.html], mostly described in `RFC 5054`[https://tools.ietf.org/html/rfc5054]. Unfortunately, Windows does not support `SRP-6a` natively, and the two open-source C libraries that exist use either the GNU MP (`GMP`) library [https://github.com/est31/csrp-gmp] or OpenSSL [https://github.com/cocagne/csrp]. The former is undesirable as a circa-2008 thread [https://gmplib.org/list-archives/gmp-discuss/2008-June/003243.html] on how Windows x64 uses LLP64 and that pointers shouldn't be defined as `int`'s in proper C has still not been closed (thus the library is unusable on any modern Windows system). The latter implementation contains a number of correctness bugs, and sadly, both libraries incorrectly interpret the RFC and do not perform the appropriate padding [https://github.com/cocagne/csrp/issues/9] of certain cryptographic numbers (as a matter of fact, there are no less than 14 different `SRP-6a` libraries on GitHub, all of which have slight variations [https://github.com/LinusU/secure-remote-password/issues/12] in their implementation!). To top it all off, `GSA` adds a number of customizations to `SRP-6a`, such that even a fully RFC-compliant library would be insufficient.

* As per the above, `SRP-6a` requires the ability to perform cryptographic operations on extremely large numbers, and `C/C++` lack a standardized library for doing so. 
Because a strong design goal was the avoidance of complex or non-Windows friendly libraries such as OpenSSL [https://www.openssl.org/docs/man1.0.2/man3/bn.html] and GMP [https://gmplib.org/], a simple library was created solely for the operations needed by `SRP-6a`. This code is slow, but simple, and is expressly implemented to support `SRP-6a`. It is not a generic drop-in replacement for a BigInt/BigNumber library.

* One of the `GSA`-specific additions to `SRP-6a` is that the password must be processed through the Password-Based Key Derivation Function 2 (`PBKDF2`) [https://tools.ietf.org/html/rfc2898]. Thankfully, Windows 7 and above support this natively in their `BCrypt` API [https://docs.microsoft.com/en-us/windows/win32/api/bcrypt/nf-bcrypt-bcryptderivekeypbkdf2]. Similarly, access to a Random Number Generator (`RNG`) is needed, which this library also provides [https://docs.microsoft.com/en-us/windows/win32/api/bcrypt/nf-bcrypt-bcryptgenrandom].

* In order to perform network communications over `HTTPS` to reach `GSA`, and due to a lack of a standardized library (for now), the `cpprestsdk` project [https://github.com/microsoft/cpprestsdk] was used, as it receives ongoing support from Microsoft and is quite excellent.

* `GSA` utilizes the Propery List (`plist`) format to describe requests and responses, which is itself an offshoot of `XML`. The `pugixml` project [https://pugixml.org/] was used to add highly simplified XML processing, and a simplified forked version of the `Plistcpp` project [https://github.com/microsoft/PlistCpp] provides the primitives needed for constructing the appropriate payloads and parsing their responses.

* Both of these projects depend on an `::any` variant object. In order to avoid the overhead of `boost`, a compiler with support for `C++17` is required to build the network-facing side of this project.

* In order to be compliant with the Two Factor Authentication (`2FA`) implementation of `GSA`, which is named `HSA2`, specific HTTP headers must be encoded by `Blackwood 4NT` into each request, and initial machine provisioning must be performed in order to avoid repeated security code requests. Once enrolled, further `GSA` logins will no longer require `2FA` from the given machine. A library, `Pastis`, was developed to handle these headers.

## Technical Architecture & Design

### API Endpoint

`https://gsa.apple.com/grandslam/GsService2`

#### Regular HTTP Headers

| Header | Description | Usage |
| --- | --- |  --- |
| `X-Mme-Client-Info` | MobileMe Client Information | See below for format |
| `X-Mme-Device-Id` | MobileMe Device Identifier | See below for format |
| `X-Mme-Legacy-Device-Id` | MobileMe Device Identifier | See below for format
| `X-Mme-Country` | MobileMe Country Identifier | Standard ISO Country Code (`US`)
| `Accept-Language` | Language Identifier | Standard ISO Language Identifier (`en`)

##### MobileMe Client Information

The format of this field, for a Windows-based request, is constructed as follows:

`<PC> <%OS%;%MAJOR%.%MINOR%(%SPMAJOR%,%SPMINOR%);%BUILD%> <%AUTHKIT_BUNDLE_ID%/%AUTHKIT_VERSION% (%APP_BUNDLE_ID%/%APP_VERSION%)>`
 
 | Identifier | Description | Usage |
| --- | --- |  --- |
| `OS` | OS Identifier | Set to Windows |
| `MAJOR` | OS Major Version | Returned from OS Version API |
| `MINOR` | OS Minor Version | Returned from OS Version API |
| `SPMAJOR` | OS Service Pack Major Version | Returned from OS Version API |
| `SPMINOR` | OS Service Pack Minor Version | Returned from OS Version API |
| `BUILD` | OS Build Number | Returned from OS Version API |
| `AUTHKIT_BUNDLE_ID` | AuthKit Bundle Identifier | Always `com.apple.AuthKitWin` |
| `AUTHKIT_VERSION` | AuthKit Version | Always `1` |
| `APP_BUNDLE_ID` | Requesting Application Bundle Identifier | Depends on application |
| `APP_VERSION` | Requesting Application Version | Depends on application -- obtained from `GetFileVersionInfo` API |

Note that due to Apple's own applications not correctly using a [https://docs.microsoft.com/en-us/windows/win32/sysinfo/targeting-your-application-at-windows-8-1](manifest file), the OS Version APIs that they use will always end up returning Windows Vista information.

Therefore, a fully constructed field typically looks as such:
 
`<PC> <Windows;6.2(0,0);9200> <com.apple.AuthKitWin/1 (com.apple.iCloud/7.21)>`
 
##### MobileMe Device Identifier

The device identifier is made up of the following pieces of information, hashed with MD5 according to the information shown below.

| Header | Description | Usage |
| --- | --- |  --- |
| `Ethernet MAC` | MAC Address of the first NIC | See below for more information |
| `Volume Serial Number` | Serial number of the `C:` volume | See below for more information |
| `Product ID` | Windows Product ID | See below for more information |
| `CPU Name` | Vendor Name for the Primary CPU | See below for more information |
| `BIOS Version` | Version  String of the System BIOS | See below for more information |
| `Machine Name` | Computer Name | See below for more information |
| `Hardware Profile GUID` | GUID of the current hardware profile | See below for more information |

For each of these 7 hashes, the first 32-bits are taken and converted to an upper-cased hex string, which is zero-extended to 8 digits.
Each hex string is then concatenated into a period ("`.`") separated string:

`EEEEEEEE.VVVVVVVV.PPPPPPPP.CCCCCCCC.BBBBBBBB.MMMMMMMM.HHHHHHHH`

###### Ethernet MAC

Obtained by calling `GetAdaptersAddresses` and performing the MD5 hash of the first returned MAC address.

###### Volume Serial Number

Obtained by calling `GetVolumeInformationW` and performing the MD5 hash of the first four bytes. Note that it is critical to use the UTF-16LE result, and only take 4 bytes (2 characters).

###### Product ID

Obtained by reading the `ProductId` value under the `HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion` registry key and performing the MD5 hash of all bytes. Note that it is critical to use the ASCII result (`RegQueryValueExA`) and hash all of the bytes in the value.

Interestingly, some of Apple's services are still only x86, and therefore run under `WOW64` on 64-bit Windows systems, and access the "virtualized" redirected version of the `SOFTWARE` hive, which does not contain this information, so the hash will be all zeroes.

###### CPU Name

Obtained by reading the `ProcessorNameString` value under the `HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\CentralProcessor\0` registry key and performing the MD5 hash of all bytes. Note that it is critical to use the ASCII result (`RegQueryValueExA`) and hash all of the bytes in the value.

###### BIOS Version

Obtained by reading the `SystemBiosVersion` value under the `HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System` registry key and performing the MD5 hash of all bytes. Note that it is critical to use the ASCII result (`RegQueryValueExA`) and hash all of the bytes in the value.

###### Machine Name

Obtained by calling `GetComputerNameW` and performing the MD5 hash of the resulting bytes. Note that it is critical to use the UTF-16LE result, and take all returned bytes.

###### Hardware Profile GUID

Obtained by calling `GetCurrentHwProfileW` and performing the MD5 hash of the GUID string returned in `szHwProfileGuid`. Note that it is critical to use the UTF-16LE result, and take all returned bytes.

##### MobileMe Legacy Device Identifier

The legacy device identifier is made up of the following pieces of information, hashed with MD5 according to the information shown below.
Pay close attention in how they differ to the non-legacy versions above.

| Header | Description | Usage |
| --- | --- |  --- |
| `Volume Serial Number` | Serial number of the `C:` volume | See below for more information |
| `BIOS Version` | Version  String of the System BIOS | See below for more information |
| `CPU Name` | Vendor Name for the Primary CPU | See below for more information |
| `Product ID` | Windows Product ID | See below for more information |

First, the ASCII string `cache-control` is hashed, followed by the ASCII string `Ethernet`.
Then, for each of these 4 hashes, the first 32-bits are taken and converted to an lower-cased hex string, which is zero-extended to 8 digits.
Each hex string is then concatenated without any separation:

`EEEEEEEEVVVVVVVVBBBBBBBBCCCCCCCCPPPPPPPP`

###### Volume Serial Number

This is computed in the same way as the non-legacy hash.

###### BIOS Version

Obtained by reading the `SystemBiosVersion` value under the `HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System` registry key and performing the MD5 hash of all bytes. Note that it is critical to use the UTF-16LE result (`RegQueryValueExW`) and hash all of the bytes in the value.

###### CPU Name

Obtained by reading the `ProcessorNameString` value under the `HKEY_LOCAL_MACHINE\HARDWARE\DESCRIPTION\System\CentralProcessor\0` registry key and performing the MD5 hash of all bytes. Note that it is critical to use the UTF-16LE result (`RegQueryValueExW`) and hash all of the bytes in the value.

###### Product ID

This is computed in the same way as the non-legacy hash.

#### Anisette HTTP Headers

| Header | Description | Usage |
| --- | --- |  --- |
| `X-Apple-I-MD-LU` | Machine Data, Local UUID | See below for description |
| `X-Apple-I-MD` | Machine Data, One Time Password (OTP) | See below for description |
| `X-Apple-I-MD-M` | Machine Data, Machine Information | See below for format
| `X-Apple-I-MD-RINFO` | Machine Data, Routing Information | Obtained after MID Provisionning

##### Local UUID

##### OTP

##### Machine Information

### API Operations

#### Machine Provision

#### Lookup

#### Authenticate Request (Init Stage)

##### Top Level Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `Header` | API Header | Used to identify version |
| `Request` | API Payload | Used to store the request |

##### Header Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `Version` | Version String | Set to `1.0.1` |

##### Request Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `A2k` | Client Public Key (`A2` Key) | Computed according to `SRP-6a` standard |
| `cpd` | Client Provided Data | Anisette headers for client identification |
| `o` | Operation | Set to `init` for this stage |
| `ps` | Protocols Supported | See table below |
| `u` | Username | Account e-mail |

###### Protocols Supported

| Type | Description | Usage |
| --- | --- |  --- |
| `s2k` | Standard | Password is sent as SHA-256 digest |
| `s2k_fo` | API Payload | Password is sent as UTF-8 Hex String of SHA-256 digest |

#### Authenticate Response (Init Stage)

##### Top Level Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `Header` | API Header | Empty |
| `Response` | API Payload | Used to store the response |

##### Response Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `Status` | Response Status | Used to store the response status |
| `i` | Iterations | Iteration count for `PBKDF2` password derivation, must be > 999 |
| `s` | Salt | User's unique salt, used for further `SRP-6a` challenges |
| `sp` | Selected Protocol | Protocol, from table above, the server wishes to use |
| `c` | Cookie | Unique identification cookie for further API requests |
| `B` | Server Public Key (`B` Key) | To be used according to `SRP-6a` standard |

##### Status Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `hsc` | HTTP Status Code | HTTP-compatible status code |
| `ed` | Error Description | GSA-specific error description |
| `ec` | Error Code | GSA-specific error code |
| `em` | Error Message | GSA-specific error message |

###### Status Codes

| Code | Description | Usage |
| --- | --- |  --- |
| `409` | Conflict | xxx |
| `200` | OK | Request accepted |
| `434` | Requested host unavailable | Anisette headers have expired |
| `421` | xx | xxx |

#### Authenticate Request (Complete Stage)

##### Top Level Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `Header` | API Header | Used to identify version |
| `Request` | API Payload | Used to store the request |

##### Header Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `Version` | Version String | Set to `1.0.1` |

##### Request Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `M1` | Client Proof (`M1` Hash) | Computed according to `SRP-6a` standard, with variation described below |
| `cpd` | Client Provided Data | Anisette headers for client identification |
| `o` | Operation | Set to `complete` for this stage |
| `sc` | Server Certificate | SHA-256 digest of SSL certificate chain |
| `u` | Username | Account e-mail |

#### Authenticate Response (Complete Stage)

##### Top Level Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `Header` | API Header | Empty |
| `Response` | API Payload | Used to store the response |

##### Response Keys

| Field | Description | Usage |
| --- | --- |  --- |
| `Status` | Response Status | Used to store the response status |
| `spd` | Server Provided Data | User token information, AES-CBC encrypted using session key |
| `M2` | Server Proof (`M2` Hash) | Used to verify server also has correct password |
| `np` | Negociation Proof | Used to verify both client and server used the same protocol settings |

#### Validate

## Credits / Thanks



## Usage

This library was developed for research and educational purposes as a personal project to better understand cryptography and modern authentication protocols. Every acronym in this library was new to me at the time of development, and as such, there are likely subtle bugs in the implementation that may arise in corner cases. Additionally, there are a number of subtle naming puns scattered throughout (including, incidentally, the very name of the library).
