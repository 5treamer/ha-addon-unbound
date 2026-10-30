# Unbound recursive resolver – Home Assistant app

**DE:** Home-Assistant-App (früher „Add-on“) mit **Unbound** als vollständig
rekursivem, DNSSEC-validierendem Resolver. Unbound fragt selbst die
Root-, TLD- und autoritativen Server und leitet **nicht** an öffentliche
Resolver weiter. Gedacht als Upstream für AdGuard Home auf Port 5335.
Gebaut aus dem offiziellen Unbound-Quellcode für `aarch64` und `amd64`.

**EN:** Home Assistant app (formerly "add-on") running **Unbound** as a fully
recursive, DNSSEC-validating resolver. It queries the root, TLD and
authoritative servers itself and does **not** forward to public resolvers.
It is meant as the upstream for AdGuard Home on port 5335 and is built from
the official Unbound source for `aarch64` and `amd64`.

- Installation, options, AdGuard setup, diagnostics: [`unbound/DOCS.md`](unbound/DOCS.md)
- Changelog: [`unbound/CHANGELOG.md`](unbound/CHANGELOG.md)

## License

App files: BSD-3-Clause, see [`LICENSE`](LICENSE). Unbound itself is
© NLnet Labs, BSD-3-Clause, see [`NOTICE`](NOTICE).
