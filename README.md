# Subordinate JWKS Update for OpenID Federation 1.0

This repository contains the "Subordinate JWKS Update for OpenID Federation"
specification. This specification acts as an extension to OpenID Federation 1.1
and defines two optional federation endpoints — the
`federation_subordinate_jwks_update_trigger_endpoint` and the
`federation_subordinate_jwks_update_endpoint` — with which a Subordinate Entity keeps the
JWKS published about it in Subordinate Statements up to date at its Superior
Entities.

## Builds

The latest editors' draft can be viewed at
https://zachmann.github.io/federation-subordinate-jwks-update/main.html

Branches can be viewed at
https://zachmann.github.io/federation-subordinate-jwks-update/{BRANCH-NAME}.html

## Build the HTML

```bash
docker run -v `pwd`:/data danielfett/markdown2rfc openid-federation-subordinate-jwks-update-1_0.md
```

## Contact

For further information and to get involved, please read [CONTRIBUTING.md](CONTRIBUTING.md) and/or contact the authors.
