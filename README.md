# CFSSL PKI Template

The template uses [CFSSL](https://github.com/cloudflare/cfssl) to generates client certificates and CRL from a private PKI.

## Overview

Steps:

1. Create a private PKI

2. Generate client certificates

3. Generate CRL

## Step-by-Step Guide 

### Generate CA certificate
```
cfssl gencert -initca ca-csr.json | cfssljson -bare ./ca/ca
```

### Generate client certificate

```
cfssl gencert -ca=./ca/ca.pem -ca-key=./ca/ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare ./client/client
```

or in bulk

```
for i in {1..10}; do cfssl gencert -ca=./ca/ca.pem -ca-key=./ca/ca-key.pem -config=ca-config.json -profile=client client.json | cfssljson -bare ./client/client-$i; done
```

Clean up all client certs
```
rm ./client/*
```

### (Not Required) Generate server certificate

```
cfssl gencert -ca=./ca/ca.pem -ca-key=./ca/ca-key.pem -config=ca-config.json -profile=server server.json | cfssljson -bare ./server/server
```


### Validate the Client Cerficates locally

Set


#### Server-side: Set up a test server
```
openssl s_server -CAfile ca/ca-key.pem  -cert server/server.pem -key server/server-key.pem -accept 4433 -www -debug -Verify 10
```

#### Client-side: Without client certificate: 


```
openssl s_client -connect localhost:4433 -servername mtls.chriswang.me  -CAfile ca/ca.pem 
```

No error is expected in the output from the server side 


#### Client-side: With client certificate: 

```
openssl s_client -connect localhost:4433 -servername mtls.chriswang.me -cert ./client/client-1.pem -key ./client/client-1-key.pem  -CAfile ca/ca.pem
```

Some error message is expected similar to the following
```
4502087276:error:140890C7:SSL routines:ssl3_get_client_certificate:peer did not return a certificate:s3_srvr.c:3346:
```

### Generate CRL


#### Get the serial numbers:
```
cfssl certinfo -cert client/client-1.pem | jq -r .serial_number > ./client/serial_numbers.txt
498099976382046766672312245706546949882852010903
```

#### Generate the CRL
```
echo "-----BEGIN X509 CRL-----" > ./client/client.crl 
cfssl gencrl client/serial_numbers.txt ./ca/ca.pem ./ca/ca-key.pem >> ./client/client.crl 
echo "-----END X509 CRL-----" >> ./client/client.crl
```

#### Check the CRL

```
openssl crl -in ./client/client.crl -text

Certificate Revocation List (CRL):
        Version 2 (0x1)
    Signature Algorithm: ecdsa-with-SHA256
        Issuer: /C=SG/ST=SG/L=Hyrule/CN=The Burrito Bot CA
        Last Update: May  7 07:38:02 2019 GMT
        Next Update: May 14 07:38:02 2019 GMT
        CRL extensions:
            X509v3 Authority Key Identifier: 
                keyid:02:10:6C:B8:51:20:7A:BE:C9:F7:08:4F:7E:80:5A:2B:33:3E:3A:60

Revoked Certificates:
    Serial Number: 573F934EF45BE1D2A4935AC2771A7FEBAE3DEB97
        Revocation Date: May  7 07:38:02 2019 GMT
    Signature Algorithm: ecdsa-with-SHA256
         30:45:02:20:43:bb:8d:80:80:43:a5:8b:99:fc:83:78:d6:47:
         72:7b:ca:24:cd:e4:ed:a8:09:60:55:7b:0f:1a:aa:58:73:0e:
         02:21:00:ef:d2:4a:a1:81:da:3c:ce:e6:aa:76:bb:73:4f:2a:
         92:19:cc:f0:ce:70:37:43:95:6a:b3:df:79:59:aa:01:cf
-----BEGIN X509 CRL-----
MIIBHjCBxQIBATAKBggqhkjOPQQDAjBIMQswCQYDVQQGEwJTRzELMAkGA1UECBMC
U0cxDzANBgNVBAcTBkh5cnVsZTEbMBkGA1UEAxMSVGhlIEJ1cnJpdG8gQm90IENB
Fw0xOTA1MDcwNzM4MDJaFw0xOTA1MTQwNzM4MDJaMCcwJQIUVz+TTvRb4dKkk1rC
dxp/664965cXDTE5MDUwNzA3MzgwMlqgIzAhMB8GA1UdIwQYMBaAFAIQbLhRIHq+
yfcIT36AWiszPjpgMAoGCCqGSM49BAMCA0gAMEUCIEO7jYCAQ6WLmfyDeNZHcnvK
JM3k7agJYFV7DxqqWHMOAiEA79JKoYHaPM7mqna7c08qkhnM8M5wN0OVarPfeVmq
Ac8=
-----END X509 CRL-----
```

Looks good. You can now proceed with the rest of your tests with client certificates and CRL.
