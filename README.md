# draft-atwood-mimi-provider-auth

An Internet-Draft defining how MIMI providers authenticate each other.
draft-ietf-mimi-protocol-07 §4.1 requires mutually authenticated TLS but
not where client certificates come from, and the public Web PKI has
stopped issuing them (Chrome Root Program Policy v1.8). The mandatory
mechanism here is a key in DNS under the provider's domain plus RFC 9421
request signatures; DANE client certificates, PKIX client certificates
from configured trust anchors, and trust lists are optional.

Posted 2026-11-15: https://datatracker.ietf.org/doc/draft-atwood-mimi-provider-auth/

## Files

- `draft-atwood-mimi-provider-auth.md`: the draft, in kramdown-rfc source.
- `draft-atwood-mimi-provider-auth-00.xml`: the RFCXML built from it.

## Building

```sh
export PATH="$(ruby -e 'print Gem.user_dir')/bin:/data/ietf-tools/venv/bin:/data/ietf-tools/node/bin:$PATH"
kramdown-rfc draft-atwood-mimi-provider-auth.md > draft-atwood-mimi-provider-auth-00.xml
xml2rfc --text --html draft-atwood-mimi-provider-auth-00.xml
idnits draft-atwood-mimi-provider-auth-00.txt
```

The example in Appendix A uses the RFC 9421 Appendix B.1.4 Ed25519 test
key and verifies.
