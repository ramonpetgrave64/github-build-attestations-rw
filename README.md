This repo contains an example reusable [workflow](.github/workflows/attest-build-provenance-slsa3-rw.yml) to use GitHub-native tools to elevate your builds to SLSA Build Level 3 requirements, which are designed to mitigate these risks:

This is **not** an official [slsa-framework](https://github.com/slsa-framework/) repo.

* unapproved changes (branches or tags)
* exposed secret signing material
* self-hosted runners
* ambiguous build steps

See [architecture docs](./docs/architecture.md).
