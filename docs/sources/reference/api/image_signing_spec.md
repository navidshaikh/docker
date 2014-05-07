# Docker Image Signing

---

**Provenance** (from the French *provenir* meaning "to come from"): the origin or source of something

---

## Overview
As Docker application packaging becomes mainstream users need to ensure that their applications are transferred securely, have integrity and come from trusted sources. Image provenance includes:
* data integrity
* issuer trust and verification
* reproducibility of docker builds
* distributed access to assemble an image
* attestation of the image assembly (commands/instructions to build it)

### Extending the metaphor
* **Manifest:** In the shipping industry, a document that describes the contents being transported and where it has come from. In this context we are creating a certificate that describes the application.
* **Seal:** In the shipping industry, an unbroken seal that shows the container contents have not been tampered with. In this context we are using GPG signing.

### Related Terms
* **Named image layer** or **Primay image layer**: The logical image as the result of a `docker build`. Dockerfile == image. Also a parent image used in the Dockerfile `FROM` line.
* **Build artifact image layer:** Image layers from each Dockerfile line that do not represent the final output of `docker build`. These layers are useful caches during build but are rarely useful to the application end user.
* **Federation:** Image layers hosted by different maintainers that may be re-assembled during `docker pull` to create a single application.
* **tarsum:** A checksum of a tarball image.

## Goals

1. **Separate transport from identity:** Image naming is arbitrary and independent of the registry location (URL).
1. **Verify offline:** Data integrity verification may be performed without connection to Internet.
1. **Remote assembly:** Image layers may be safely pulled from multiple issuers when assembling application layers.
1. **Inspect image assembly:** An application may be inspected to determine how it was assembled and who assembled each layer.
1. **Arbitrary metadata support:** Authors or maintainers may add arbitary metadata describing the applicatation to support their unique needs.
1. **Application certification support:** Provide mechanism for a certifying party to certify applications built by independent software vendors (ISVs).
1. **Apply Chain of trust model:** In the same way that a signed certificate can be traced to a trusted root CA issuer, layer ancestry built on a series of signed and trusted layers can be trusted.
1. **Retain simplicity:** The signing model should be transparent to the end-user and straightforward for the maintainer. Provide tools to dive deep if information is desired. Otherwise, users are able to easily build images and securly pull images without excessive security burdens.

## Non-goals
1. **Secure data in flight:** A secured SSL/TLS connection is desired when an image layer is transferred. Secure transfer should be the default but insecure transfer supported.

## Foundational work

### Decouple image name and location (URL/registry)
Images names are independent of the registry location. Images may be pushed to many registries or mirrors so layers are easily accessible. Consider the BitTorrent-style federation of file servers.

### Use deterministically generated image IDs
The image IDs are cryptographic hashes (checksum/tarsum) of the image so they are meaningful and useful for disambiguation. The hash guarantees the correct image will be used regardless of where it is stored or served from.

## The Dockerfile as a Manifest

The Dockerfile is the authoritative document for the image layer. Here we add useful metadata and a maintainer fingerprint.

```
FROM        fedora
TO          aweiteka/myapp
MAINTAINER  Aaron Weitekamp <aweiteka@gmail.com> 04AA 2FAA 4762 69AB 3407  FA45 FD5E B4DB 4807 17ED
SOURCE      https://github.com/aweiteka/myapp.git
DESCRIPTION Lorem ipsum dolor sit amet, consectetur adipisicing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua.
META        [
            "docs" :    "<https://readthedocs.org/>",
            "license" : "<http://example.com/>",
            "product" : "ID123456789",
            "policy"  : "Publication or distribution policy statement with URL",
            "foo" :     "bar"
            ]

...

```

Directive       | Purpose
-----           | -----
`TO`            | defines the layer or application name. The maintainer controls this immutable string in the form of `<namespace>/image_name`. Images are not re-tagged by end-users.
`MAINTAINER`    | includes the public GPG key, output of `gpg --fingerprint`. This value is used during build-time signing of the image layer.
`SOURCE`        | optional but provides a standard way for maintainers to publish build source files. This increases trust, transparency and reliability.
`DESCRIPTION`   | is optional but use is highly encouraged.
`META`          | is an arbitrary list of `key:value` pairs for maintainer use. Potential uses include links to online documentation, license or certification information. Consider standardizing keys for parsing and displaying.

With this Dockerfile a build may be reproduced by anyone but signed only by the maintainer. It gives control to the maintainer while providing transparency and build documentation to the end-user.

## CLI Workflow

### build

`docker build <image> [--version] [--sign]`

* Version is a build-time artifact. The author optionally determines version tag. Otherwise images are tagged `:latest`.
* Signing involves several actions:
  * Dockerfile is copied into image as metadata
  * Dockerfile metadata fields are written as metadata. This includes `FROM`, `TO`, `MAINTAINER`, `SOURCE`, `DESCRIPTION`, `META`, parent hash and build timestamp.
  * squashes build artifact layers so a single, logical image layer is the result of the build. Parent layer(s) are retained (not squashed).
  * removes build artifact containers, same as `--rm`
  * assumes author has private key to matching public fingerprint specified in Dockerfile

Build also stores these computed values as metadata (not exhaustive list):
* computed tarsum hash
* parent tarsum hash
* build timestamp

### push

`docker push <image> --url <url>|--all`

Image layers may be pushed to a specific registry URL or all configured registries. Registry URLs are configured using `docker login`, which generates authentication file `$HOME/.dockercfg`. See [documentation](http://docs.docker.io/use/workingwithrepository/#authentication-file).

```
~/.dockercfg
{
     "https://index.docker.io/v1/": {
             "auth": "xXxXxXxXxXx=",
             "email": "email@example.com"
     },
     "https://my-registry.com": {
             "auth": "XxXxXxXxXxX=",
             "email": "email@my-registry.com"
     }
}
```

### tag

Deprecate `docker tag`. Tag is not needed since transport is decoupled from name and the author controls immutable name and versioning.

### verify

`docker verify <image> [--silent]`

Check GPG signature. Similar to `rpm -V <package>`. Verbose output unless `--silent` is passed, when only exit status is returned. Silent verification is performed during `docker pull`.


## Application Certification Model
The Docker ecosystem enables opportunities for ISVs to distribute certified applications built on platform base layer(s). In this model each layer is signed by the author/maintainer. A certifying party then builds and signs a "metadata-only" layer that certifies the application. The metadata provides attestation that the application is certified by the party and references related policy information.

## GnuPG
GPG is a suite of cryptographic tools compatible with OpenPGP. It relies on a web of trust relationship between a public key and its owner. It implements a decentralized public key infrastructure (PKI) trust model where certificate authority (CA) organizations are not used to validate a public key. Root CA systems like X509 require issuers (maintainers) to either have a root-signed certificate or self-sign. Self-signed X509 certificates do not provide enough verification of issuer trust. GPG is developer friendly and used by large organizaions. For example, GPG is used to sign RPM packages from Red Hat, who publishes their [public key](https://access.redhat.com/site/security/team/key/).

## Standards and resources
* OpenPGP [RFC4880](http://tools.ietf.org/html/rfc4880)
* [GnuPG](https://www.gnupg.org/)
* <http://godoc.org/code.google.com/p/go.crypto/openpgp>

## Related Discussions and Pull Requests (PRs)

Too many to track.

* [Issue#2700](https://github.com/dotcloud/docker/issues/2700) Signed Images
* [PR#3070](https://github.com/dotcloud/docker/pull/3070) Add support for certificates for repositories

## Open Questions (FIXME)
* What signing algorithm? At least SHA-256, preferrably SHA-512
* Should Dockerfile `META` keys be standardized while still allowing for extension?
