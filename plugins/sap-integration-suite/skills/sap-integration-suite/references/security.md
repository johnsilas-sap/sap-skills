# Security Reference

## Security Material Store

All credentials, keys, and certificates are stored in the Security Material store — never hardcoded in iFlows.

```
Integration Suite → Monitor → Manage Security → Security Material
  Types:
    User Credentials     — username + password
    OAuth2 Client Credentials — client ID + secret + token URL
    Secure Parameter     — single encrypted string value
    Known Hosts (SFTP)  — SFTP server fingerprints
    PGP Public Key      — encrypt outbound data
    PGP Secret Key      — decrypt inbound data
    Keystore             — X.509 certificates and private keys
```

## User Credentials

```
Security Material → Add → User Credentials
  Name:      SAP-EWM-Backend
  User:      svc_cpi_user
  Password:  (encrypted)

" Used in adapter authentication:
  HTTP Adapter → Authentication: Basic
  Credential Name: SAP-EWM-Backend
```

## OAuth2 Client Credentials

```
Security Material → Add → OAuth2 Client Credentials
  Name:           EWM-OAuth2
  Grant Type:     Client Credentials
  Client ID:      sb-1a2b3c4d...
  Client Secret:  (encrypted)
  Token Service URL: https://my-s4.authentication.eu10.hana.ondemand.com/oauth/token
  Token Service User: (leave blank for client credentials)
  Scope:          (leave blank or specify scopes)

" Used in adapter:
  HTTP Adapter → Authentication: OAuth2 Client Credentials
  Credential Name: EWM-OAuth2
  → CPI auto-fetches and refreshes token
```

## Secure Parameter (Single Encrypted Value)

```
Security Material → Add → Secure Parameter
  Name:      CarrierAPIKey
  Value:     xK9pZ3mN...   (encrypted)

" Access in iFlow Groovy script:
def apiKey = ITApiFactory.getApi(SecureStoreService.class)
              .getUserCredential('CarrierAPIKey')
              .getPassword()

" Access in Content Modifier Expression:
${sec_param:CarrierAPIKey}   (not always available — prefer Groovy for this)
```

## SFTP Known Hosts

```
Security Material → Add → Known Hosts (SSH)
  Name:       sftp-carrier-knownhosts
  Host:       sftp.carrier.com
  Algorithm:  RSA / ECDSA
  Fingerprint: (copy from: ssh-keyscan sftp.carrier.com)

" Used in SFTP Adapter Sender/Receiver:
  Known Hosts File: sftp-carrier-knownhosts
```

## SFTP Key-Based Authentication

```
Security Material → Keystore
  Upload private key (PEM) as: SSH private key pair
  Alias: sftp-carrier-private-key

" In SFTP Adapter:
  Authentication: Public Key
  Private Key Alias: sftp-carrier-private-key
```

## PGP Encryption (Outbound)

```
Security Material → Add → PGP Public Key
  Name:    carrier-pgp-public
  Key:     (paste partner's ASCII-armored public key)

Step: PGP Encryptor
  Key Alias:     carrier-pgp-public
  Symmetric Algorithm: AES_256
  Content Type:  Binary / Text
```

## PGP Decryption (Inbound)

```
Security Material → Add → PGP Secret Key Ring
  Name:      mycompany-pgp-secret
  Key Ring:  (upload secret keyring file)
  Passphrase: (encrypted)

Step: PGP Decryptor
  Secret Key Ring: mycompany-pgp-secret
  Passphrase Alias: (optional — if stored separately)
```

## X.509 Certificates (Keystore)

```
Integration Suite → Monitor → Manage Security → Keystore
  Import Certificate:
    Alias:       my-s4-root-ca
    Certificate: (PEM or DER format)
    " Used for: trust store (verify backend server cert)

  Import Key Pair:
    Alias:       mycompany-signing
    Type:        PKCS12 (.pfx)
    Password:    (encrypted)
    " Used for: AS2 message signing, WS-Security, mTLS client cert
```

## Inbound Authentication Options

```
HTTP Sender → Authentication Options:

1. Client Certificate (mTLS):
   Client presents X.509 cert → CPI verifies against keystore
   Role mapping in IDP: cert subject → iFlow user roles

2. Basic Authentication:
   User/pass → verified against IDP users with iFlow-specific role
   Role required: ESBMessaging.send

3. OAuth 2.0 (Bearer Token):
   Token issued by BTP IAS/IDP
   CPI validates token scope/audience

4. None:
   Open endpoint (use only for internal calls / testing)
```

## Role Assignment (iFlow Access Control)

```
BTP Cockpit → Security → Role Collections
  Create: EWM-Integration-Senders
    Role: ESBMessaging.send   (allows sending to HTTP-triggered iFlows)
  
  Assign to:
    BTP Users / Service Instances / Client Certs
```

## Accessing Security Material in Groovy

```groovy
import com.sap.it.api.ITApiFactory
import com.sap.it.api.securestore.SecureStoreService
import com.sap.gateway.ip.core.customdev.util.Message

def Message processData(Message message) {
    def secureStore = ITApiFactory.getApi(SecureStoreService.class, null)
    
    // Read User Credentials
    def cred = secureStore.getUserCredential('SAP-EWM-Backend')
    def user  = cred.getUsername()
    def pass  = new String(cred.getPassword())
    
    // Read Secure Parameter
    def apiKey = new String(secureStore.getSecureParameter('CarrierAPIKey').getValue())
    
    // Set as header for outbound adapter
    message.setHeader('Authorization', 'Basic ' + 
        Base64.encoder.encodeToString("${user}:${pass}".bytes))
    message.setHeader('X-API-Key', apiKey)
    
    return message
}
```

## Certificate Rotation

```
" Rotate expiring certs without downtime:
1. Upload new cert to Keystore with DIFFERENT alias (e.g., mycompany-signing-2026)
2. Update all iFlows referencing old alias to new alias
3. Deploy updated iFlows
4. Delete old alias from Keystore after cutover
```
