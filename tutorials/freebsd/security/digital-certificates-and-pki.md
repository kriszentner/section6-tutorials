# Digital Certificates and PKI

This document is merely an informational synopsis of Public Key Infrastructure
Topics Covered

* Public Key Infrastructure
* Digital Certificates
* Certificate Authorities
* Registration Authorities
* Secured Sockets Layer

## Public Key Infrastructure

Public Key Infrastructure (PKI) offers one of the most secure ways of protecting an enterprise network's electronic communications. These communications use what is known as digital signatures and public key cryptography. A digital signature is the equivalent of a signature on a letter, and is the only recognized technology to enable clear identification of the originator of a communication while Public Key cryptography provides scalable and manageable key management.

PKI is the combination of software, encryption technologies, and services that enables enterprises to protect the security of their communications and business transactions on the Internet. The concept of PKI integrates digital certificates, public-key cryptography, and certificate authorities into a total, enterprise-wide network security architecture. A typical enterprise's PKI encompasses the issuance of certificates to individual users and servers; end-user enrollment software; integration with corporate certificate directories; tools for managing, renewing, and revoking certificates; and related services and support.

## Digital Certificates

Digital certificates establish credentials when performing communications and transactions on a network. Certificates contain a name, a serial number, expiration dates, a copy of the certificate holder's public key (used for encrypting messages and digital signatures), and the digital signature of the certificate-issuing authority so that a recipient can verify that the certificate is real. Most digital certificates conform to a standard referred to as X.509. This standard sets guidelines for certificate-issuing authorities to follow in order to make use of most modern applications thus making it easier to conduct legitimate transactions over a network.

The most common use of a digital certificate is to verify communication for a user whom he or she claims to be, and to provide the receiver with the means to reply through encryption. Examples of common use might include accounts for a financial institution, or an encrypted email message using PGP.

An individual or company wishing to use encryption with communication over a network would receive a digital certificate from a Certificate Authority (CA). The CA issues an encrypted digital certificate containing the recipient's public key along with extended information such as the serial number. The CA makes its own public key readily available for verification through an Internal Registration Authority or through an Internet Rregistration Authority.

The recipient of encrypted communication uses the CA's public key to decode the digital certificate attached to the message, verifies it as issued by the CA by performing a checksum of the version, the serial number, the sender, and the sender's public key. Then the recipient can verify that certificate against a Registration Authority. With this information, the recipient can respond with an encrypted session.

## Certificate Authorities

A Certificate Authority (CA) is a computer (or group of computers) on a network that signs and issues public keys for tansaction and message encryption. As part of a public key infrastructure, a CA checks with a Registration Authority to verify information provided by the requestor of a digital certificate. If the RA verifies the requestor's information, the CA can then issue a certificate.

Usually, this means that the CA has an arrangement with a business which provides it with information to confirm an individual's claimed identity. Certificate Authorities are a critical component in data security and electronic commerce because they guarantee that the two parties exchanging information are really who they claim to be.
Registration Authorities

A Registration Authority (RA) is simply a trusted computer that runs services to verify the validity of certificates being issued by a Certificate Authority. Most private networks combine their own CA with a Registration Authority as the central repository, but for internet transactions there are recognized providers that most businesses use. These providers include companies such as VeriSign or DigiCert.

Enterprise networks who only want certificates issued to specifically identified and authenticated individuals. achieve this goal by using an their own RA. As discussed earlier, this is a service that can either be physically separate or combined with a CA
Certificate Repositories

A Certificate Repository is used to propagate both certificates and the status of the issued certificates. The most commonly used repository service for certificates is a Lightweight Directory Access Protocol (LDAP) server. The CA will push certificates to the repository and the clients pull the certificates from the repository using an LDAP based client application. The LDAP implementation could be either a simple LDAP server or a full X.500 product.

The certificate status repository can take one of two forms. If Certificate Revocation Lists (CRL) or CRL Distribution Points (CDPs) are used, then a LDAP server is used. CRLs work on the concept of an administrator revoking a certificate's unique ID from the distribution list. The CRL is then pushed to the LDAP server with clients pulling the latest CRL from the server on a scheduled basis. The other technique is called On-line Certificate Status Protocol (OCSP). In this case a "cache" (some form of repository) holds the status of certificates. Each time a client application wishes to use a certificate, it will query the OCSP cache for the validity of the certifiacte.

## Secured Sockets Layer

Secured Sockets Layer (SSL) is a protocol standard that encrypts your communications over a network. The SSL protocol ensures that the information is sent, unchanged, only to the server you intended to send it to. The SSL standard is not a single protocol, but rather a set of accepted data transfer routines that uses the public and private keys as ciphers to encrypt the transmission between two computers. Online shopping sites and Financial Institutions frequently use SSL technology to safeguard account information. The first time a client accesses a service that uses SSL, the client is presented with a digital certificate. This information allows the client to determine three important things:

* The certicate has not yet expired
* The certificate was created for the computer being accessed.
* The certificate is signed by an authority the client trusts.

If any of these three conditions are not met, most applications will display a warning that explains the violations and requests permission to continue accessing. An example might be an SSL-capable web browser. The web browser should include a list of respositories from Registration Authorities that are considered trusted. Part of the reason why you don't see any popup certificate window when you enter Amazon's secure area is that their certificate was signed by an authority known to your browser.


## Security Concerns

Certificate Authority Trust: The risk of accepting unauthorized certificates lies in the hands of the user. Knowing what authorities can be trusted by a signature on the issued certificate is essential in maintaining security.

Sites such as Verisign and/or Digicert offer a database of known registrants where users can query valid certificates. This helps users in tracking down a company or organization and verifying its authenticity, but the application must support CRLs or OCSP. If it does not then the user must know to manually check for the validity of the certificate issued.

Access of the private key: Users' private keys are usually stored on a physically accessible computer. Sharing that computer with other users increases the risk of a private key being stolen. As malware and trojan programs become more sophisticated, the likelihood of a malicious attack and hijack of a user's private key increases.

One such technology that looks to overcome this issue is a token based system called Smart Card Technology. Smart Cards hold a private key along with the extended user information. Devices called Smart Card Readers are used to pass the information from the card to the application sending a token requesting the use of the key for the encrypted transaction.

Registration Authority Security: The overall security of the Registration Authority that verifies the public key being issued can be a risk. CA issues what is commonly known as a root certificate. If this root certificate is forged, then users can be re issued a public key.

A CA (Along with its RA) can revoke a certificate it has issued. This may be for a number of reasons but a prime reason would be a certificate compromise: someone has accessed the private key associated with a certificate and it needs to be revoked for security reasons. Each CA should publish a list of revoked certificates. This list is known as a Certificate Revocation List (CRL). Normally these lists are publicly available so that a server can check it before accepting a given certificate.