+++
date = "2016-09-23T16:38:22+02:00"
draft = true
title = "TLS Series Part 1: TLS Handshake Explained"

+++



## TLS Handshake

https://tools.ietf.org/html/rfc5246#section-7.4

TLS defines a total of three subprotocls to carry out the TLS handshake which allows peers to agree on a set of security parameters in order to authenticate themselves and start an encrypted channel.

The *handshake protocol* is responsible for transmission of supported parameters like cipher specifications and compression, session IDs, certificate exchangeand  generation of secrets.

The *change cipher spec* only has the purpose of telling the peer, that from now on all following records will be encrypted with the negotiated ciphers and keys.

The *alert protocol* to transmit alerts during the TLS handshake. These alterts can either have a severity of warning or fatal in which case the connection will be immediately terminated. There are several types of alerts like certificate_expired, unsupported_extension or unsupported_extension just to name a few. For a detailed list please refer to [RFC5246, Section 7.2](http://tools.ietf.org/html/rfc5246#section-7.2).

The number of handshake messages to be exchanged during the setup of an TLS connection depends on several factors and varies between 6 and 13 messages. 

For a new session a full handshake with up to 10 messages is required. 
```

     | Client        (Full handshake with server authentication)         Server |
-----+--------------------------------------------------------------------------+------
 (1) | Client Hello ----------------------------------------------------------> |      
     | <---------------------------------------------------------- Server Hello |  (2) 
     | <--------------------------------------------------------- Certificate*  |  (3) 
     | <-------------------------------------------------- Server Key Exchange* |  (4) 
     | <--------------------------------------------------- Server Hello Done   |  (5) 
 (6) | Client Kex Exchange ---------------------------------------------------> |      
 (7) | Change Cipher Spec ----------------------------------------------------> |      
 (8) | Finished --------------------------------------------------------------> |      
     | <--------------------------------------------------- Change Cipher Spec  |  (9) 
     | <-------------------------------------------------------------- Finished | (10) 
     | <--------------------------- Encrypted Traffic ------------------------> |
* Optional or situation-dependent messages
```
If the server requests client authentication up to 13 messages will be exchanged:
```

      | Client        (Full handshake with mutual authentication)         Server |
------+--------------------------------------------------------------------------+------
  (1) | Client Hello ----------------------------------------------------------> |
      | <---------------------------------------------------------- Server Hello |  (2)
      | <----------------------------------------------------------- Certificate |  (3)
      | <-------------------------------------------------- Server Key Exchange* |  (4)
      | <--------------------------------------------------- Certificate Request |  (5)
      | <----------------------------------------------------- Server Hello Done |  (6)
  (7) | Certificate -----------------------------------------------------------> |      
  (8) | Client Kex Exchange ---------------------------------------------------> | 
  (9) | Certificate Verify* ---------------------------------------------------> |      
 (10) | Change Cipher Spec ----------------------------------------------------> |      
 (11) | Finished --------------------------------------------------------------> |      
      | <--------------------------------------------------- Change Cipher Spec  | (12) 
      | <-------------------------------------------------------------- Finished | (13) 
      | <--------------------------- Encrypted Traffic ------------------------> |
* Optional or situation-dependent messages
```

If a priviously established session should be resumed an abbreviated handshake of 6 messages is used:

```
     | Client      (Session resumption with abbreviated handshake)        Server |     
-----+---------------------------------------------------------------------------+-----
 (1) | Client Hello -----------------------------------------------------------> |     
     | <----------------------------------------------------------- Server Hello | (2) 
     | <----------------------------------------------------- Change Cipher Spec | (3) 
     | <--------------------------------------------------------------- Finished | (4) 
 (5) | Change Cipher Spec -----------------------------------------------------> |     
 (6) | Finished ---------------------------------------------------------------> |     
     | <---------------------------- Encrypted Traffic ------------------------> |
```

The messages on order of required 

### Client Hello

Negotiation always starts with the Client Hello message which includes the TLS version, all supported cipher suites, a 32 bytes random block called ClientHello.random, a session ID field, compression methods and optional extensions.

### Server Hello

After recieving a Client Hello the server has to respond with a Server Hello or otherwise the connection will fail with a fatal error also sent by the server. The Server Hello contains the TLS version and cipher suite which will be used, a random block called ServerHello.random, a session ID field, a compression method field and otional extensions. 

###  Certificate (Server)

Following its Server Hello the server will optionally sent its certificate if the chosen key exchange method uses certificates for authentication. This message also includes the certificate chain, if any. Ofcourse the certificate and its public key must be compatible to the chosen cipher suite.

###  Server Key Exchange

The Server Key Exchange message will be sent directely after the certificate message respectively after the Server Hello if no certificate message was sent. This message also is optional and will be sent if optional paramaters are needed in order to allow the client to generate the premaster secret.

### Certificate Request

This optional message will be sent if the server requires the client to authenticate itself. The message contains a list of acceptable certificate types, signature algorithms and distinguished names of acceptable CAs. 

### Server Hello Done

With this message the server informs the client that it is done sending the ServerHello message and that the client now should verify the recieved information. 

### Certificate (Client)

In the first message after the ServerHelloDone the client will send its certificate if requested from the server by means of an CertificateRequest message. If the client can't provide a certificate matching the servers demands the client must send an empty certificate message.

### Client Key Exchange

This mandatory message must be sent after the client certificate message respectively after the client recieved the ServerHelloDone. The ClientKeyExchange message defines the premaster sectret, by either containing a premaster secret encrypted with the servers public key or Diffie-Hellman parameters from wich each peer can derive a premaster secret. 

### Certificate Verify

The client will send this message only if the earlier trasmitted client certificate has signing capabilities.

### Change Cipher Spec (Client)

With the ChangeCipherSpec message the client informs the server that all subsequently sent records will be encrypted.

### Finished (Client)

The finished message will be the first message ancrypted with the negotiated set of algorithms and secrets, containing a hash calculated of all handshake messages, the finished_label "client finished" and the master secret by means of a pseudorandom function (PRF) for the server to verify.

### Change Cipher Spec (Server)

The server on his part will now switch to encryption and inform the client about that step with his ChangeCipherSpec message.

### Finished (Server)

The first encrypted record by the server is it's finished message for the client to verify.

[Back to Top](#REPLACEMEFORTOPLINK)

### Example of an HTTPS connection

To illustrate the TLS handshake a little bit better you'll below find a real world example from a connection to https://google.com. You can download the complete trace file as [.pcapng](pcap/https_google.com.pcapng) or view an [ascii version](pcap/https.pcap.complete.md). I'll also link to the respective frames of the capture as I discuss the various handschake messages. 

#### Client Hello

In [Frame 4](pcap/https.pcap.frame_4.md) youl see the Client Hello which always starts the TLS handshake.

![Client Hello](img/https_f4_client_hello.jpg)

The Client Hello contains several pieces of information.

First of all the **TLS version** the client want's to use for this session

```
Version: TLS 1.2 (0x0303)
``` 

followed by **32 random bytes**. Out of these 32 bytes 4 bytes represent the current time and date in UNIX format and 28 bytes are generated by the systems RNG. 

```
     Random
         gmt_unix_time: Mar 18, 2016 10:27:30.000000000 CET
         random_bytes: ff0692082db8b7296bd0189924d2443c538ab7678647bd2b...
```

For the handshake itself it's not important that the clients time matches the servers time at all. The client could choose to also use an RNG for the 4 time-bytes without violating the handshake protocol. Syncronized times might be a requirement for higher protocols though.

For new connections the **session ID** in the Client Hello is always empty. If the clients wishes to resume a previous session it would use the session ID of the preview session instead.

The next block is a list of **cipher suites** the client supports, sorted by preference (top to bottom). In our example the client supports a total of 66 cipher suites.

```
Cipher Suites (66 suites)
                Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 (0xc030)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384 (0xc02c)
                Cipher Suite: TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA384 (0xc028)
                Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA384 (0xc024)
                [...]
```  

The **comression methods** field lists all desired compression methods. In our case the clients sends the default compression method ```null``` which indicates that the client does not want to use compression. 
```
       Compression Methods (1 method)
            Compression Method: null (0)
```
If the clients tries to resume a previous session it must the same compression method as before.

The extensions block contains several extensions to indicate that the client supports several features which the server might want to use. 

In this case the client uses the *server_name* extension for Server Name Indication (SNI) in order to tell the server, which domain the client wants to reach.

```
       Extension: server_name
              Type: server_name (0x0000)
              Length: 15
              Server Name Indication extension
                  Server Name list length: 13
                  Server Name Type: host_name (0)
                  Server Name length: 10
                  Server Name: google.com
```

The next two extensions, *ec_point_formats* and *elliptic_curves*, tell the server that the client supports the point formats defined in [RFC4492](http://tools.ietf.org/html/rfc4492) and from which elliptic curves the server could choose.

```
       Extension: ec_point_formats
           Type: ec_point_formats (0x000b)
           Length: 4
           EC point formats Length: 3
           Elliptic curves point formats (3)
               EC point format: uncompressed (0)
               EC point format: ansiX962_compressed_prime (1)
               EC point format: ansiX962_compressed_char2 (2)
       Extension: elliptic_curves
           Type: elliptic_curves (0x000a)
           Length: 8
           Elliptic Curves Length: 6
           Elliptic curves (3 curves)
               Elliptic curve: secp521r1 (0x0019)
               Elliptic curve: secp384r1 (0x0018)
               Elliptic curve: secp256r1 (0x0017)
``` 

The empty *SessionTicket TLS* extension indicates client support for [RFC5077](http://tools.ietf.org/html/rfc5077) "Session Resumption without Server-Side State". Using SessionTicket TLS the server can allow session resumption without having to cache session information for all clients itself. Instead it encrypts all necessary session state info for a client and forwards this in form of a ticket to the client which the client can use to resume the session at a later time. We will soon see, that the server also supports this.
If the client wishes to resume an earlier session it would send an SessionTicket TLS extension containing the appropriate ticket.

In the *signature_algorithms* extension the client lists all supported signature and hash algorithm pairs. If the client doens not send this extension, the server gets his information on the supported algorithms from the list of supported cipher suites. In this case the client supports a total of 15 algorithms.

```
       Extension: signature_algorithms
           Type: signature_algorithms (0x000d)
           Length: 32
           Signature Hash Algorithms Length: 30
           Signature Hash Algorithms (15 algorithms)
               Signature Hash Algorithm: 0x0601
                   Signature Hash Algorithm Hash: SHA512 (6)
                   Signature Hash Algorithm Signature: RSA (1)
               Signature Hash Algorithm: 0x0602
                   Signature Hash Algorithm Hash: SHA512 (6)
                   Signature Hash Algorithm Signature: DSA (2)
               Signature Hash Algorithm: 0x0603
                   Signature Hash Algorithm Hash: SHA512 (6)
                   Signature Hash Algorithm Signature: ECDSA (3)
               Signature Hash Algorithm: 0x0501
                   Signature Hash Algorithm Hash: SHA384 (5)
                   Signature Hash Algorithm Signature: RSA (1)
               Signature Hash Algorithm: 0x0502
                   Signature Hash Algorithm Hash: SHA384 (5)
                   Signature Hash Algorithm Signature: DSA (2)
               Signature Hash Algorithm: 0x0503
                   Signature Hash Algorithm Hash: SHA384 (5)
                   Signature Hash Algorithm Signature: ECDSA (3)
               Signature Hash Algorithm: 0x0401
                   Signature Hash Algorithm Hash: SHA256 (4)
                   Signature Hash Algorithm Signature: RSA (1)
               Signature Hash Algorithm: 0x0402
                   Signature Hash Algorithm Hash: SHA256 (4)
                   Signature Hash Algorithm Signature: DSA (2)
               Signature Hash Algorithm: 0x0403
                   Signature Hash Algorithm Hash: SHA256 (4)
                   Signature Hash Algorithm Signature: ECDSA (3)
               Signature Hash Algorithm: 0x0301
                   Signature Hash Algorithm Hash: SHA224 (3)
                   Signature Hash Algorithm Signature: RSA (1)
               Signature Hash Algorithm: 0x0302
                   Signature Hash Algorithm Hash: SHA224 (3)
                   Signature Hash Algorithm Signature: DSA (2)
               Signature Hash Algorithm: 0x0303
                   Signature Hash Algorithm Hash: SHA224 (3)
                   Signature Hash Algorithm Signature: ECDSA (3)
               Signature Hash Algorithm: 0x0201
                   Signature Hash Algorithm Hash: SHA1 (2)
                   Signature Hash Algorithm Signature: RSA (1)
               Signature Hash Algorithm: 0x0202
                   Signature Hash Algorithm Hash: SHA1 (2)
                   Signature Hash Algorithm Signature: DSA (2)
               Signature Hash Algorithm: 0x0203
                   Signature Hash Algorithm Hash: SHA1 (2)
                   Signature Hash Algorithm Signature: ECDSA (3)
```

And finally, using the *heartbeat* extension, the clients indicates support for heatbeat protocol defined in [RFC6520](http://tools.ietf.org/html/rfc6520).

```
       Extension: Heartbeat
           Type: Heartbeat (0x000f)
           Length: 1
           Mode: Peer allowed to send requests (1)
```

A list of all current extensions can be found at [IANA](https://www.iana.org/assignments/tls-extensiontype-values/tls-extensiontype-values.xhtml).  

[Back to Top](#REPLACEMEFORTOPLINK)

#### Server Hello

The server responds to the Client Hello with it's **Server Hello** message in [Frame 6](pcap/https.pcap.frame_6.md) of our capture file. This packet actually includes the steps 2 to 5 of the full handshake.

![Server Hello](img/https_f6_server_hello.jpg)

In case the server can't agree on an acceptable set of algorithms based on the clients information it will send an handshake failure alert instead.

The structure of the Server Hello shows a lot of similarities compared to the Client Hello. The main difference is, that all fields from the Server Hello ony contain one single option. 

The first field contains the greatest common **TLS version** of client and server. In our example both sides support TLS 1.2. 

The **32 random bytes** follow the same structure as the clients random block. Again, it's not required to use the actual time here.

```
     Random
         gmt_unix_time: Mar 18, 2016 10:27:54.000000000 CET
         random_bytes: d9aad625295740adf3f5b33d35e0d88b8609e398688e768f...
```

The **session ID** field can be either a 32 byte session ID (a new one or an old one if the cliend tries to resume a session and the server supports that) or it can again be empty like in our example. That indicates, that the server won't cache this session and the session can't be resumed at a later time. In our example the client has submitted the SessionTicket TLS extension, which allows the server to hand of all session state data to the client for later session resumption. In the extensions block of the Server Hello we'll see, that the server agrees to also support SessionTicket TLS.

The **cipher suite** field contais the one cipher suite out of the clients list that the server agrees on.
```
Cipher Suite: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256 (0xc02b)
```
In case client and server resume a previous session the cipher suite must be identical to the one used in the previous session.

Finally the **compression methods** field contains one of the clients proposed methods. Since our client only proposed ```null``` this is what the server also uses.

From the Server Hello **extension block** i'll focus on the two most interesting ones. First of all, the empty *renegotiation_info* extension indicates, that the server supports secure renegotiation. Since the client did not include this extension in it's Client Hello we can assume, that the client does not support this.

The next one is an empty *SessionTicket TLS* extention, just like the one the client already sent. This indicates that the server wishes to send a new session ticket to the client once it recieved and verified the clients *finished* message.   

#### Certificate

Following the Server Hello this packet also contains the servers **certificate** and chain in DER encoding. The server can only omit it's certificate and chain if the agreed upon cipher suite does not require authentication for the key exchange (e.g. DH_anon).

The first certificate in the chain must be the one of server itself and all following certificates in the chain must be in the correct signing order. So each certificate in the chain exept from the server certificate must directely certify the one preceding it. The root certificate should not be included in the chain.

In our example the chain contains a total of three certificates:

![google.com Certificate Chain](img/https_f6_certificates.jpg)

You can also do a live check using an ```openssl``` command:

```
# openssl s_client -servername google.com -connect google.com:443 2>/dev/null
CONNECTED(00000003)
---
Certificate chain
 0 s:/C=US/ST=California/L=Mountain View/O=Google Inc/CN=*.google.com
   i:/C=US/O=Google Inc/CN=Google Internet Authority G2
 1 s:/C=US/O=Google Inc/CN=Google Internet Authority G2
   i:/C=US/O=GeoTrust Inc./CN=GeoTrust Global CA
 2 s:/C=US/O=GeoTrust Inc./CN=GeoTrust Global CA
   i:/C=US/O=Equifax/OU=Equifax Secure Certificate Authority
---
[... rest of the output omitted...]
```

As you can see, the root certificate (```s:/C=US/O=Equifax/OU=Equifax Secure Certificate Authority```) ist not included in the servers handshake certificate message. It should be already available and known by the client.

The public key included in the certificate must be compatible with the chosen key exchange algorithm.

#### ServerKeyExchange

The *ServerKeyExchange* must be sent immediately after the certificate handshake message **if** the information from the certificate message does not contain sufficient data for the key exchange. Since the server choses ECHDE_ECDSA as the key exchange mechanism it uses a ServerKeyExchange message to sent its public key to the client along with specifications of the corresponding curve.

```
    TLSv1.2 Record Layer: Handshake Protocol: Server Key Exchange
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 149
        Handshake Protocol: Server Key Exchange
            Handshake Type: Server Key Exchange (12)
            Length: 145
            EC Diffie-Hellman Server Params
                curve_type: named_curve (0x03)
                named_curve: secp256r1 (0x0017)
                Pubkey Length: 65
                pubkey: 04429983c674d5f10c0684368eb39e7eacc6a3b4e0b1ec7c...
                Signature Hash Algorithm: 0x0403
                    Signature Hash Algorithm Hash: SHA256 (4)
                    Signature Hash Algorithm Signature: ECDSA (3)
                Signature Length: 72
                signature: 3046022100d735cd46980537bc9c8f16ce034f7a48bc66bc...
```

**Note:** If you extract the public key and view it with ```openssl ec``` you will see, that the named curve secp256r1 will be called prime256v1 in OpenSSL:

```
# openssl ec -pubin -in googlepubkey.pem -text -noout
read EC key
Private-Key: (256 bit)
pub:
    04:2a:38:1a:53:90:5e:5e:5b:7e:06:21:7f:10:2e:
    29:3a:2a:e4:f0:10:70:e1:96:50:83:d1:41:66:2a:
    da:56:f1:a4:ab:82:22:c2:70:24:6d:d9:95:14:51:
    a2:ac:81:23:86:f4:14:d7:67:19:bd:67:ad:8d:44:
    72:5a:1b:6e:99
ASN1 OID: prime256v1
```
#### ServerHelloDone

When the server sent all relevant information for the key exchange to the client it sends a ServerHelloDone message. Any authentication requests towards the client must be sent before the ServerHelloDone.

[Back to Top](#REPLACEMEFORTOPLINK)

#### ClientKeyExchange

In [frame 8](pcap/https.pcap.frame_8.md) the client contunues the handshake with its ClientKeyExchange message. Since the server did not request a certificate from the client for authentication this message must be the first message to be sent by the client after recieving the ServerHelloDone message.

With ECDHE being the key exchange protocol the client sends its DHE public key.

```
    TLSv1.2 Record Layer: Handshake Protocol: Client Key Exchange
        Content Type: Handshake (22)
        Version: TLS 1.2 (0x0303)
        Length: 70
        Handshake Protocol: Client Key Exchange
            Handshake Type: Client Key Exchange (16)
            Length: 66
            EC Diffie-Hellman Client Params
                Pubkey Length: 65
                pubkey: 04697f00982066e33fb4f463ae51c33548a3573e539cc0b9...
```

The contents of the ClientKeyExchange message depend on the agreed key exchange mechanism. If e.g. RSA would be chosen as the key exchange algorithm, the ClientKeyExchange message would include the RSA-encrypted premaster secret:

```
       Handshake Protocol: Client Key Exchange
           Handshake Type: Client Key Exchange (16)
           Length: 258
           RSA Encrypted PreMaster Secret
               Encrypted PreMaster length: 256
               Encrypted PreMaster: 3247d55a388a2332124c40f79f7b2aec0f9e6b719a9344cb...
```
#### Change Cipher Spec (Client)

With the *ChangeCipherSpec* message, whis is actually not a handshake message but its own subprotocol defined in [RFC5246 Section 7.1](https://tools.ietf.org/html/rfc5246#section-7.1), the client informs the server that is has all necessary parameters to use encryption from now on for all following records.

```
    TLSv1.2 Record Layer: Change Cipher Spec Protocol: Change Cipher Spec
        Content Type: Change Cipher Spec (20)
        Version: TLS 1.2 (0x0303)
        Length: 1
        Change Cipher Spec Message
```
The message contains only a single byte with a value of 1.

#### Client finished Message (Encrypted Handshake Message)

The next message following the ChangeCipherSPec is the clients *finished* message which also is the first encrypted message from the client to the server. The server must now validate the contents of the message which contains a hash calculated of all handshake messages, the finished_label "client finished" and the master secret by means of a pseudorandom function (PRF).

```
      verify_data
           PRF(master_secret, finished_label, Hash(handshake_messages))
```

[Back to Top](#REPLACEMEFORTOPLINK)

#### Change Cipher Spec (Server)

Hello Request: This message will be ignored by the client if the client is currently negotiating a session."

