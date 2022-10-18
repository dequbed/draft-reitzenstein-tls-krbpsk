---
###
# Internet-Draft Markdown Template
#
# Rename this file from draft-todo-yourname-protocol.md to get started.
# Draft name format is "draft-<yourname>-<workgroup>-<name>.md".
#
# For initial setup, you only need to edit the first block of fields.
# Only "title" needs to be changed; delete "abbrev" if your title is short.
# Any other content can be edited, but be careful not to introduce errors.
# Some fields will be set automatically during setup if they are unchanged.
#
# Don't include "-00" or "-latest" in the filename.
# Labels in the form draft-<yourname>-<workgroup>-<name>-latest are used by
# the tools to refer to the current version; see "docname" for example.
#
# This template uses kramdown-rfc: https://github.com/cabo/kramdown-rfc
# You can replace the entire file if you prefer a different format.
# Change the file extension to match the format (.xml for XML, etc...)
#
###
title: "TLS Key Establishment using the Kerberos V5 Network Authentication Service"
abbrev: "TLS-KRB5PSK"
category: info

docname: draft-reitzenstein-tls-krbpsk-latest
submissiontype: independent # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: false
v: 3
area: AREA
workgroup: TLS
keyword:
 - Kerberos
 - TLS-PSK

author:
 -
    ins: N. von Reitzenstein Čerpnjak
    fullname: Nadja von Reitzenstein Čerpnjak
    email: me@dequbed.space

normative:
  RFC3961:
  RFC4120:
  RFC8446:

informative:


--- abstract

This document specifies a TLS PSK key establishment mode using the Kerberos V5 Network Authentication Service ({{RFC4120}}).

This allows combining the TLS encryption state machine with the key exchange abilities of Kerberos.


--- middle

# Introduction

TODO Introduction

"Noooooo, you can't just use Kerberos key material as sources of randomness! Hahahah TLS-KRB5PSK go brrrrrrrrr"!

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# TLS Extensions

## KerberosV5 Application Request Extension

The "krb5_ap_psk" extension is used to transfer a KerberosV5 KRB_AP_REQ message from a client to a server.

The "extension_data" field of this extension contains a "Krb5ApPsk" value:

    struct {} Empty;

    struct {
        select (Handshake.msg_type) {
            case client_hello: opaque authentication_header<1..2^16-1>;
            case server_hello: select (mutual_required) {
                case TRUE: opaque authentication_response<1..2^16-1>;
                case FALSE: Empty;
            }
        }
    } Krb5ApPsk;

The "authentication_header" field of the extension is set to an encoding of a KerberosV5 KRB_AP_REP message, as specified in {{RFC4120}}, with a key usage of 11 used for the Authenticator.

The following restrictions apply to a client constructing an KRB_AP_REP and Authenticator:
- The mutual-required flag of "ap-options" is set if the client wishes for the server to authenticate itself.
- "cksum" MUST be set to the checksum calculated per {{RFC4120}} over the ClientHello message up to, but excluding, the krb5_ap_psk message.
- The optional "seq-number" field MUST NOT be set.
- The "subkey" field SHOULD be filled with a high-randomness key. The value of the subkey SHOULD NOT be reused in future TLS sessions or other applications.

The Early Secret {{RFC8446, Section 7.1}} is generated from the subkey or session key using the following procedure when krb5_ap_psk is selected as PSK mechanism:
A "protocol-key" is selected. If the "subkey" field was set in the Authenticator this subkey MUST be used as protocol-key. Otherwise the "session key" the provided ticket was encrypted with is used as protocol-key.
The key-derivation function {{RFC3961, Section 3}} specified for the keytype of the protocol-key is used to generate 32 bytes of keymaterial with the selected protocol-key and an usage number of XX (TODO!).
The resulting secret is used as IKM for the PSK step of the TLS key schedule.

A client MUST NOT send 0-RTT data encrypted with an early traffic secret generated from the krb5_ap_rep extension, although it may send early data encrypted using other PSK modes. A server that selects krb5_ap_psk as psk MUST ignore all 0-RTT data.

TODO: This *could* be safe though. Figure out how to make sure it is.

Prior to accepting PSK key establishment using krb5_ap_psk a server MUST validate the provided Authenticator by checking its ability to decrypt it, and checking that the fields "authenticator-vno", and "cksum" contain correct values. A server MUST reject tickets with Authenticators that are older than a configured time as indicated by the Authenticators "ctime" field. If no other age is configured a default of 5 minutes SHOULD be used.
In order to accept PSK key establishment using krb5_ap_psk a server sends a "krb5_ap_psk" extension.

The "authentication_response" field of this extension, if set, is the encoding of a KerberosV5 KRB_AP_REP message, as specified in {{RFC4120}}.
A server MUST send a populated authentication_response value if the mutual-required flag in the the KRB_AP_REQ message provided by the client is set, or abort the handshake.

A server SHOULD NOT populate the optional 'subkey' field in the EncAPRepPart of the KRB_AP_REP it sends. A client MUST ignore the value of 'subkey' received by the server.

# The KRB5PSK pre shared key exchange mode

A client indicates willingness to use the KRB5PSK mode by sending a "psk_key_exchange_modes" extension indicating support for the "krb5_dhe_ke" mode.
Additionally the client must send a "key_share" extension and a "krb5_ap_psk" extension populated with an KRB_AP_REQ containing the Kerberos ticket they want to present to the server.
A server indicates to a client that it has selected the KRB5PSK mode by sending a "key_share" extension and a "krb5_ap_psk" extension with its ServerHello response.


# Security Considerations

TODO Security


# IANA Considerations

This document requests assignment of a entry in the TLS PskKeyExchangeMode from IANA for krb5_dhe_ke. This mode is not marked "Recommended" and the reference is set to this document.

This document additionally requests assignment for a TLS ExtensionType Entry "krb5_ap_psk" with the values defined in this document and the "Recommended" value of "N".

--- back

# Acknowledgments
{:numbered="false"}

I blame Sharp for this documents existance.
