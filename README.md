# TrueNAS Integration for Elastic

[![CI](https://github.com/NateShoffner/truenas-elastic-integration/actions/workflows/ci.yml/badge.svg)](https://github.com/NateShoffner/truenas-elastic-integration/actions/workflows/ci.yml)

An Elastic Agent integration that collects audit records (`audit`) and system
alerts (`alert`) from [TrueNAS](https://www.truenas.com/), the OpenZFS-based
storage operating system from iXsystems.

Both data streams poll the TrueNAS REST API v2.0 over HTTPS using the CEL input.
Compatible with TrueNAS SCALE 24.04 through 25.10; **not TrueNAS 26**, which
removed the REST API.

**[Read the integration documentation](packages/truenas/docs/README.md)** for
setup steps, configuration options, exported fields, and troubleshooting.

## Installation

Download the `.zip` from the [latest release](https://github.com/NateShoffner/truenas-elastic-integration/releases/latest)
and upload it through **Kibana > Integrations > Upload integration**, or:

```sh
elastic-package install --zip truenas-<version>.zip
```

To build it yourself instead:

```sh
cd packages/truenas
elastic-package build     # writes build/packages/truenas-<version>.zip
elastic-package install
```

## Development

Requires [`elastic-package`](https://github.com/elastic/elastic-package) and
Docker. Run from `packages/truenas`:

```sh
elastic-package check            # lint and build
elastic-package stack up -d -v   # needed by both test commands below
elastic-package test pipeline    # ingest pipeline tests
elastic-package test system      # end-to-end tests against a mock API
```

Edit documentation in `_dev/build/docs/README.md`, never in `docs/README.md`,
which is regenerated on every build.

### Releasing

Bump `version` in `packages/truenas/manifest.yml`, add a matching
`changelog.yml` entry, then push a tag:

```sh
git tag v0.2.0 && git push origin v0.2.0
```

CI verifies the tag matches the manifest version, builds the package, and
publishes a GitHub release with the `.zip` attached and notes drawn from the
changelog entry.

## License

[Elastic License 2.0](LICENSE).
