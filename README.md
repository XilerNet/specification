# Ordinals Decentralised Domains

A project aimed to create a usable domain name system on Bitcoin ordinals. This uses the TLD `.o`. On top of that, this is decentralized, it also does not align with the subscription model of current domains and has taken a more strict approach to domains. Because of this, we want to make a more secure network without having to rely on big organizations for DNS resolution.

> NOTE: In this specification when referring to an inscription id that might not exist yet *(if it is the first inscription of its kind for the domain)* then the value `null` should be used.

> NOTE: All queries can be chained using a newline character *(`\n`)*, if the queries are chained, the signature can be of the whole chained data, so it only needs to be provided once at the end. If you are registering a domain, and you want to add a validation pk, you can do this directly in the chained inscription and can directly sign that with the signature.

> NOTE: When using chained inscriptions while registering a domain, only a `DOMAIN` and a `DOMAIN-VALIDITY` are allowed to be present. If such an inscription gets transferred the validity will be invalidated.

> NOTE: When no signature is required, the value `null` must be used. If this is not present the inscription will be seen as invalid.

## Domain Inscription

The domains are validated by this REGEX expression: `DOMAIN [a-z\d](?:[a-z\d-]{0,251}[a-z\d])?.o \d+`, following the structure: `DOMAIN <NAME>.o <valid from unix epoch>`.

This means that only lowercase alphanumeric and a `-` characters are allowed for up to a length of 254 characters, and must end with the `.o` suffix.

This is extending the [IETF rfc952 ASSUMPTIONS 1](https://www.ietf.org/rfc/rfc952.txt) specification. *(compliant qua scheme, just extending the allowed domain length)*

Here are some examples of domains that would be seen as valid:
```
DOMAIN e.o 1685954907
DOMAIN example.o 1685954907 
DOMAIN 54t05h1.o 1685954907
DOMAIN my-example.o 1685954907 
```
And these would be seen as invalid: *(and thus ignored)*
```
DOMAIN .o 1685954907 
DOMAIN *.o 1685954907 
DOMAIN invalid 1685954907 
DOMAIN INVALID.o 1685954907 
DOMAIN -invalid.o 1685954907 
DOMAIN invalid.o
```

> WARNING: When inscribing only the first domain of its instance is seen as valid. And a preference is given to the wallet which already owns that domain *(for the validity)*.

### How is validity checked?

1. Checks if the record has a valid structure
2. Is the `valid from unix epoch` passed, and if it is still before the expiry date (+365 days from validity)

If there are multiple records with the same domain name, the first one will be used.

## Subdomains / DNS records

> NOTE: Root DNS records should be defined as `@.`

Your domain inscription only defines that you own a domain. No linking to an address or record has been made here, as this would prevent you from making any changes in the future. The solution to this is to create an extra inscription and check if the wallet which owns the DNS inscription also owns the first and valid domain. 

The records get validated by this REGEX expression: `DNS [a-z\d](?:[a-z\d-]{0,251}[a-z\d])?.o [a-z\d](?:[a-z\d\.-]{1,62}[a-zd])?\. [A-Z]{1,5} [A-Z]{2} \d{1,9} \S+ [a-z\d]{64}i[0-9] \S+`, as you can see. These inscriptions are again case sensitive, and restricted. This is to prevent phishing attacks within the `.o` domain space. Further validation and restrictions might be applied to the RDATA depending on the record type. *(eg a A record only allowing a ip, etc)*

This is compliant with the [IETF rfc1034 3.6 Resource Records](https://www.ietf.org/rfc/rfc1034.txt) specification and is extending the [IETF rfc952 ASSUMPTIONS 1](https://www.ietf.org/rfc/rfc952.txt) specification *(compliant qua scheme, just extending the allowed domain length)*. But adding the `DNS <DOMAIN>` prefix for identification and a `<last inscription id> <signature>` suffix.

Examples of valid subdomains:
```
DNS example.o e. CNAME IN 30 example.com 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS example.o example. CNAME IN 30 example.com 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS example.o example. A IN 30 127.0.0.1 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS example.o example. A IN 400000000 127.0.0.1 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS example.o example.other. A IN 30 127.0.0.1 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS example.o example-other. A IN 30 127.0.0.1 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
```
And the following subdomains would not be seen as valid: *(and thus ignored)*
```
DNS invalid.o example. CNAME IN 30 example.com
DNS invalid.o example. CNAME IN 30 example.com 5bc53d9227...i0
DNS invalid example-other. A IN 30 127.0.0.1 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS invalid.o example CNAME IN 30 example.com 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS invalid.o example-. CNAME IN 30 example.com 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS invalid.o -example. CNAME IN 30 example.com 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS invalid.o example.. CNAME IN 30 example.com 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS invalid.o .example. CNAME IN 30 example.com 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS invalid.o EXAMPLE. CNAME IN 30 example.com 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS invalid.o example. A IN 4700000000 127.0.0.1 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DNS invalid.o example. A IN 5100000000 127.0.0.1 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
```

## Drop subdomains / DNS records and domains

Invalidate a record, does as it says. Prevents the resolver from returning the ip in a round-robin form.

It gets validated by this regex expression: `DROP [a-z\d]{64}i[0-9] \S+` which follows this format: `DROP <inscription id> <signature>`

> NOTE: This record invalidates your domain, this allows you to recover a domain even if you have lost your wallet private key. 

```
DROP 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
```

## Domain validity

Since anyone can inscribe to any address, we need to verify that a DNS record is issued by the owner of the wallet without sacrificing the ability to trade the domain and having to pass the domain.  

### Define validity

This record defines the public key and its algorithm *(max 32chars)* for a domain. This gets used for validating a DNS record and its ownership. The first detected instance (that has not been invalidated) that is on the same address is the valid key.

Domain validity regex expression: `DOMAIN-VALIDITY [a-z\d](?:[a-z\d-]{0,251}[a-z\d])?.o [a-z\d]{1,32} \S+`, formatted as `DOMAIN-VALIDITY <domain> <algorithm> <public key>` 

```
DOMAIN-VALIDITY example.o dilithium5 1m8jUVZEffs/rP3Osiz...
```

### Invalidate validity

The record invalidates an existing domain validity, this can be used to update an old validity. It utilises this format: `DOMAIN-VALIDATE-TRANSFER <domain> [optional:<algorithm> <new public key>] <domain invalidate inscription id> <signature>`

Invalidate validity regex expression: `DOMAIN-VALIDATE-TRANSFER [a-z\d](?:[a-z\d-]{0,251}[a-z\d])?.o (?:[a-z\d]{1,32} \S*)? [a-z\d]{64}i[0-9] \S+`

> NOTE: All DNS records will be invalidated by this record!

```
# Set no new validation public key
DOMAIN-VALIDATE-TRANSFER example.o 5bc53d9227...i0 ope6kNBBuUJi6H4NH...

# Set a new validation public key
DOMAIN-VALIDATE-TRANSFER example.o dilithium5 5bc53d9227...i0 1m8jUVZEffs/rP3Osiz... ope6kNBBuUJi6H4NH...
```

## Extra domain data

Extra domain-related data get validated by this regex expression: `DOMAIN-DATA [a-z\d](?:[a-z\d-]{0,251}[a-z\d])?.o .+ [a-z\d]{64}i[0-9] \S+`, following the format of `DOMAIN-DATA <DOMAIN> <data> <last id> <signature>`. Further restrictions for domain data records can be applied by the application. 

Some example usage:
```
DOMAIN-DATA example.o extra data is present here 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
DOMAIN-DATA example.o { "still_valid": true } 5bc53d9227...i0 ope6kNBBuUJi6H4NH...
```
