# Blackwood-4NT
`Blackwood 4NT` -- Grand Slam Authentication for Windows NT (10)

## Introduction

`Blackwood 4NT` is a simple `C/C++`-based Windows library for authenticating with Grand Slam Authentication (`GSA`) and obtaining the Server Provided Data (`spd`) that contains, among other things, the Password Equivalent Token (`PET`), which can then be used for further API authentication when combined with the user's Alternate Directory Services Identifier (`ADSID`/`AltDSID`).

## Components

Multiple pieces of technology had to be created in order to enable `Blackwood 4NT` to function. 

* Successful communication with `GSA` requires an implementation of Secure Remote Password, Version 6a (`SRP6a`), mostly described in `RFC 5054`. Unfortunately, Windows does not support `SRP6a` natively, and the two open-source C libraries that exist use either the GNU MP (`GMP`) library or OpenSSL. The former is undesirable as a circa-2006 thread on how Windows x64 uses LLP64 and that pointers shouldn't be defined as `int`'s in proper C has still not been closed (thus the library is unusable on any modern Windows system). The latter implementation contains a number of correctness bugs, and sadly, both libraries incorrectly interpret the RFC and do not perform the appropriate padding of certain cryptographic numbers (as a matter of fact, there are no less than 14 different `SRP6a` libraries on GitHub, all of which have slight variations in their implementation, including a thread tracking them all!). To top it all off, `GSA` adds a number of customizations to `SRP6a`, such that even a fully RFC-compliant library would be insufficient.

* As per the above, `SRP6a` requires the ability to perform cryptographic operations on extremely large numbers, and `C/C++` lack a standardized library for doing so. 
Because a strong design goal was the avoidance of complex or non-Windows friendly libraries such as OpenSSL and GMP, a simple library was created solely for the operations needed by `SRP6a`. This code is slow, but simple, and is expressly implemented to support `SRP6a`. It is not a generic drop-in replacement for a BigInt/BigNumber library.

* One of the `GSA`-specific additions to `SRP6a` is that the password must be processed through the Password-Based Key Derivation Function 2 (`PBKDF2`). Thankfully, Windows 7 and above support this natively in their `BCrypt` API. Similarly, access to a Random Number Generator (`RNG`) is needed, which this library also provides.

* In order to perform network communications over `HTTPS` to reach `GSA`, and due to a lack of a standardized library (for now), the `cpprestsdk` project was used, as it receives ongoing support from Microsoft and is quite excellent.

* `GSA` utilizes the Propery List (`plist`) format to describe requests and responses, which is itself an offshoot of `XML`. The `pugixml` project was used to add highly simplified XML processing, and a simplified forked version of the `cppplist` project provides the primitives needed for constructing the appropriate payloads and parsing their responses.

* Both of these projects depend on an `::any` variant object. In order to avoid the overhead of `boost`, a compiler with support for `C++17` is required to build the network-facing side of this project.

* In order to be compliant with the Two Factor Authentication (`2FA`) implementation of `GSA`, which is named `HSA2`, specific HTTP headers must be encoded by `Blackwood 4NT` into each request, and initial machine provisioning must be performed in order to avoid repeated security code requests. Once enrolled, further `GSA` logins will no longer require `2FA` from the given machine. A library, `Pastis`, was developed to handle these headers.

## Technical Architecture

## Credits
