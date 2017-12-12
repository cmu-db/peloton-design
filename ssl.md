# SSL Connection

## Overview
SSL connection provides secure communication between the server and the client.

## Scope
This feature is in the Network Layer, mainly implemented in Network Manager, Network Connection and Protocol Handler. 

## Introduction
### SSL Handshake ###
The SSL or TLS handshake enables the SSL or TLS client and server to establish the secret keys with which they communicate. In the process, they will agree on the protocol version, cryptographic algorithms, do certificate authentication and use asymmetric encryption techniques to generate a shared secret key.

### SSL Read (non-blocking) ###
The SSL data needs to be decrypted and verified before it is delivered to the application. The entire SSL record is needed for decryption and verification, so even if the application wants to read only one byte, the OpenSSL needs to receive the entire SSL record containing that byte from the connection. After OpenSSL decrypts and verifies the record, it will place it in its internal SSL buffer and return the number of data that is requested to the application. The SSL buffer is maintained by OpenSSL and the operating system doesn’t know the status of the buffer. Thus, if all the data is read from network buffer to the SSL buffer, select() won’t work since the network buffer is empty. OpenSSL provides a function SSL_pending() to indicate whether there is data left unread in this buffer.

The error queue needs to be cleared before checking the error code of the current operation.Some error codes are explained here:
* *SSL_ERROR_WANT_WRITE* indicates the OpenSSL is doing a rehandshake and is doing a write during the rehandshake. Need to call SSL_read() again when the socket becomes readable.
* *SSL_ERROR_WANT_READ* indicates the network buffer is empty and it would have been blocked if we set the network socket to be blocking.

### SSL Write (non-blocking) ###
Similar issues exist in SSL Write as in Read. If OpenSSL wants to write a record but there is not enough space in the network buffer, it would throw SSL_ERROR_WANT_WRITE error. It will send out the part of data that can fit in the network buffer and need the application to call SSL_write() again using the same application buffer to send out the rest of the SSL packet. OpenSSL automatically remembers where the buffer write pointer was and only writes the data after the write pointer.

* *SSL_ERROR_WANT_WRITE* indicates that we have unflushed data in the SSL buffer. We need to call SSL_write() again with the same application buffer.
* *SSL_ERROR_WANT_READ* indicates the OpenSSL is doing a rehandshake and is doing a read during the handshake. Need to call SSL_write() again when the socket becomes writable.

### SSL Rehandshake ###
Automatic rehandshake support is built into SSL_read() and SSL_write().

### SSL Authentication ###
Server can set the level of certificate-based client authentication of whether to attempt or enforce a certificate-based client authentication. It is done by SSL_CTX_set_verify().

### SSL Session ###
OpenSSL uses a session object to store the session information defining a set of security parameters and can be shared by multiple SSL connections. Session objects need to be associated SSL connections or SSL_CTX using session ID context.

## Architectural Design
The SSL component is added to the previous Network Layer in this way:

### NetworkManager ###
Maintains a global SSL context which loads server certificate, private key file and settings of client authentication, server session caching, callback registration for multithreaded environment, etc for all connections. The SSL context is initialized when Peloton starts up.

### NetworkConnection ###
1. For the startup packet, if its version code is of SSL protocol, check whether the server supports SSL and prepares for SSL handshake.
2. Perform SSL read and write in FillReadBuffer() and FlushWriteBuffer() for SSL connections.
3. For SSL connection, in the state transitions in StateMachine, manually check whether it is doing rehandshake or whether there is data unread in the SSL buffer.

### ProtocolHandler ###
Process the SSL request packet at the beginning of a SSL connection.

### Certificates and Keys ###
Certificates and keys of root, server and client are copied into `/peloton/data` directory referring PostgreSQL design.
Certificates and keys are created for Peloton SSL default setting and testing. `/peloton/data/intermediate_openssl.cnf` and `/peloton/data/openssl.cnf` are created as Peloton default configuration files used in certificates creation process. Peloton has two bash scripts `/peloton/script/installation/create_certificates.sh` and `/peloton/script/installation/create_certificates_test.sh`. The create_certificates.sh will create default required certificates in `/peloton/data`. And create_certificates_test.sh creates complete directory including intermediate files in `/peloton/data` for testing.

### Multithread Support ###
Some global data structures(error queue, global SSL_CTX, etc) are not thread safe in OpenSSL. OpenSSL uses mutex to protect these structures and requires applications to provide two extra callbacks(CRYPTO_set_id_callback and CRYPTO_set_locking_callback). Peloton uses static locks which is initialized in NetworkManager.

Version issue. For OpenSSL version greater than 1.1.0, the application does not need the extra callbacks above. For Linux, APT brings version lower than 1.1.0 right now. For new version of MAC OS X, it supports even older version of OpenSSL. New interfaces like `CRYPTO_THREADID_set_callback` are not supported so we still use the relative deprecated API `CRYPTO_set_id_callback`.

## Testing Plan
### PostgreSQL ###
PostgreSQL provide protection in different modes. In PostgreSQL terminal, we can set sslmode parameter like `verify-full` or `verify-ca`, and providing the system with a root certificate to verify against. If our SSL component works, “SSL connection (protocol: TLSv1, cipher: AES256-SHA, bits: 256, compression: off)” will pop up in the terminal. We write unit test to check it.

### JDBC ###
We add `/peloton/script/testing/jdbc/src/SSLTest.java`. In that test file we set some properties for JDBC to support SSL.

We also write bash script `/peloton/script/testing/jdbc/load_rootcrt_ssl.sh` to import root certificate to JRE to test.

## Trade-offs and Potential Problems

## Future Work

## Reference
https://www.ibm.com/support/knowledgecenter/en/SSFKSJ_7.1.0/com.ibm.mq.doc/sy10660_.htm
http://www.linuxjournal.com/article/5487?page=0,0
https://linux.die.net/man/3/ssl_accept
https://www.pgcon.org/2014/schedule/attachments/330_postgres-for-the-wire.pdf
http://h41379.www4.hpe.com/doc/83final/ba554_90007/ch04s02.html
https://www.quora.com/Digital-Certificates-What-is-the-difference-between-usecase-for-pem-pfx-fp-cer-crt-etc-files-and-how-can-they-be-converted-to-one-another
https://www.postgresql.org/docs/9.1/static/libpq-ssl.html
