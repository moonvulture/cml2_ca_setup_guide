# CML2 CA Setup Guide

## Setting up the CA directory structure
```bash
mkdir cml2_ca
cd cml2_ca
mkdir certs crl newcerts private
chmod 700 private
touch index.txt
openssl rand -hex 16 > serial
touch openssl.cnf
```
## OpenSSL Configuration File

1. Create the 'openssl.cnf' with the following content:

```ini
[ ca ]
default_ca = CA_default

[ CA_default ]
dir = /home/ec2-user/cml2_ca
certs = $dir/certs
crl_dir = $dir/crl
new_certs_dir = $dir/newcerts
database = $dir/index.txt
serial = $dir/serial
RANDFILE = $dir/private/.rand

private_key = $dir/private/root-ca.key
certificate = $dir/certs/root-ca.crt

default_md = sha256
default_days = 365

policy = policy_strict

[ policy_strict ]
countryName             = match
stateOrProvinceName     = supplied
organizationName        = supplied
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional

[ req ]
default_bits=2048
distinguished_name = req_distinguished_name
string_mask = utf8only
x509_extensions = server_cert

[ req_distinguished_name ]
countryName             = Country Name (2 letter code)
stateOrProvinceName     = State or Province Name
localityName            = Locality Name
organizationName        = Organization Name
organizationalUnitName  = Organizational Unit Name
commonName              = Common Name
emailAddress            = Email Address

# Defaults
countryName_default             = US
stateOrProvinceName_default     = Lab
organizationName_default        = Superlab

[ v3_ca ]
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid:always,issuer
basicConstraints = critical,CA:true
keyUsage = critical, digitalSignature, cRLSign, keyCertSign

[ server_cert ]
basicConstraints = CA:FALSE
nsCertType = server
nsComment = "OpenSSL Generated Server Certificate"
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer:always
keyUsage = critical,digitalSignature,keyEncipherment
extendedKeyUsage = serverAuth


## Generating Root CA key and certificate

openssl genrsa -aes256 -out private/root-ca.key 4096
chmod 400 private/root-ca.key
openssl req -config openssl.cnf -key private/root-ca.key -new -x509 -days 7300 -sha256 -extensions v3_ca -out certs/root-ca.crt

## Router configuration

[on router] # https://community.cisco.com/t5/networking-knowledge-base/creating-a-csr-authenticating-a-ca-and-enrolling-certificates-on/ta-p/4436090

crypto key generate rsa label superlab modulus 2048
crypto pki tustpoint superlabRootCA
subject-name C=US, ST=Lab, O=Superlab, CN=iosv-csw01.superlab.win
subject-alt-name iosv-csw01
rsakeypair superlab
revocation-check none
enrollment terminal pem
exit
crypto pki enroll superlabRootCA
copy csr
issue cert
crpyto pki authenticate superlabRootCA
copy in root-ca.crt
crypto pki import superlabRootCA certificate
paste cert issued by ca server

## Troubleshooting

### Inspect the root CA certificate
```
inspect certificate: 
openssl x509 -noout -text -in certs/root-ca.crt
inspect csr:
openssl req -in ../iosv-csw01.csr -text -noout
show crypto
show crypto pki certificates verbose superlabRootCA
