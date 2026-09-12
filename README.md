# opr-poc1: rings as databases, not directories

Proof of concept for publishing Omarchy packages from **one immutable pool**
where the ring databases (`edge`, `rc`, `stable`) live *beside* the packages.
No redirect layer, no per-ring directory tree, no copying: a ring is a small
`.db` file, and promotion is writing a new `.db` file.

The whole walkthrough below is testable/verified. Every `$` line is run and its
output is compared by [clitest](https://github.com/aureliojargas/clitest):

```
clitest README.md
```

Requirements: `pacman`, `makepkg`, `fakeroot`, `gpg`, `clitest`
(all on a stock Arch/Omarchy box). Nothing needs root. Everything generated
lives under `/tmp/opr-poc1/`; the checkout itself stays clean.

## The idea in one pacman.conf entry

```
[omarchy-edge]
Server = https://pkgs.omarchy.org/pool/$arch
```

pacman fetches `$Server/omarchy-edge.db` and then expects package
files at `$Server/<filename>`. If every ring's database is written into the
same `pool/$arch/` directory that holds the packages, all three rings resolve
to the same files and the only per-ring artifact is the database itself.

The "index" is then just one pinned list per ring (`rings/<ring>.txt`, one
package filename per line) checked into git, and `repo-add` turns a list into
a signed database.

## Layout

```
pkgs/examplepkg-*/PKGBUILD     one marker package, three versions: its only
                               file, /etc/examplepkg, names the ring it
                               was built for (stable, rc, edge)
bin/pool-ingest PKG...         copy built packages into the pool once, sign them
bin/ring-publish RING          regenerate the signed db for RING from its list
bin/fakeroot-pacman RING ...   unprivileged pacman configured for RING
client/pacman.conf.in          the client config; only the ring name varies
/tmp/opr-poc1/build/           makepkg output, before ingest
/tmp/opr-poc1/rings/           pinned list per ring (the "index")
/tmp/opr-poc1/repo/            the published pool (override with POOL=...)
/tmp/opr-poc1/gnupg/           the signing key (GNUPGHOME for the publisher)
/tmp/opr-poc1/keyring/         the client keyring, public key only
/tmp/opr-poc1/client-RING/     throwaway pacman root per ring
```

## Walkthrough

### 1. Start clean, create a signing key

```console
$ rm -rf /tmp/opr-poc1 && mkdir -p /tmp/opr-poc1/{build,rings,gnupg} && chmod 700 /tmp/opr-poc1/gnupg
$ export GNUPGHOME=/tmp/opr-poc1/gnupg
$ gpg --batch --quiet --passphrase '' --quick-gen-key 'Omarchy POC <poc@omarchy.org>' ed25519 sign never 2>/dev/null
$ gpg --list-keys --with-colons poc@omarchy.org 2>/dev/null | grep -c '^pub'
1
$
```

### 2. Build the marker package three times

Each build differs only in the contents of `/etc/examplepkg`.

```console
$ for v in 1.0 1.1 1.2; do BUILDDIR=/tmp/opr-poc1/makepkg PKGDEST=/tmp/opr-poc1/build makepkg -D pkgs/examplepkg-$v --nodeps --force >/dev/null 2>&1; done
$ ls /tmp/opr-poc1/build
examplepkg-1.0-1-any.pkg.tar.zst
examplepkg-1.1-1-any.pkg.tar.zst
examplepkg-1.2-1-any.pkg.tar.zst
$
```

### 3. Ingest into the pool exactly once

A package is uploaded and signed once. The pool is append-only; re-ingesting
an existing filename is refused rather than overwritten.

```console
$ bin/pool-ingest /tmp/opr-poc1/build/*.zst
ingested examplepkg-1.0-1-any.pkg.tar.zst
ingested examplepkg-1.1-1-any.pkg.tar.zst
ingested examplepkg-1.2-1-any.pkg.tar.zst
$ bin/pool-ingest /tmp/opr-poc1/build/examplepkg-1.0-1-any.pkg.tar.zst
refusing to overwrite examplepkg-1.0-1-any.pkg.tar.zst
$ ls /tmp/opr-poc1/repo/x86_64
examplepkg-1.0-1-any.pkg.tar.zst
examplepkg-1.0-1-any.pkg.tar.zst.sig
examplepkg-1.1-1-any.pkg.tar.zst
examplepkg-1.1-1-any.pkg.tar.zst.sig
examplepkg-1.2-1-any.pkg.tar.zst
examplepkg-1.2-1-any.pkg.tar.zst.sig
$
```

### 4. Pin each ring and publish its database

A ring is a list of pool filenames. Publishing writes the signed database
next to the packages; nothing else moves.

```console
$ echo examplepkg-1.0-1-any.pkg.tar.zst > /tmp/opr-poc1/rings/stable.txt
$ echo examplepkg-1.1-1-any.pkg.tar.zst > /tmp/opr-poc1/rings/rc.txt
$ echo examplepkg-1.2-1-any.pkg.tar.zst > /tmp/opr-poc1/rings/edge.txt
$ for r in stable rc edge; do bin/ring-publish $r; done
$ ls /tmp/opr-poc1/repo/x86_64 | grep -v pkg.tar
omarchy-edge.db
omarchy-edge.db.sig
omarchy-edge.db.tar.gz
omarchy-edge.db.tar.gz.sig
omarchy-edge.files
omarchy-edge.files.sig
omarchy-edge.files.tar.gz
omarchy-edge.files.tar.gz.sig
omarchy-rc.db
omarchy-rc.db.sig
omarchy-rc.db.tar.gz
omarchy-rc.db.tar.gz.sig
omarchy-rc.files
omarchy-rc.files.sig
omarchy-rc.files.tar.gz
omarchy-rc.files.tar.gz.sig
omarchy-stable.db
omarchy-stable.db.sig
omarchy-stable.db.tar.gz
omarchy-stable.db.tar.gz.sig
omarchy-stable.files
omarchy-stable.files.sig
omarchy-stable.files.tar.gz
omarchy-stable.files.tar.gz.sig
$
```

The databases are signed with the pool key and pacman can verify them:

```console
$ gpg --verify /tmp/opr-poc1/repo/x86_64/omarchy-stable.db.sig 2>&1 | grep -o 'Good signature from "[^"]*"'
Good signature from "Omarchy POC <poc@omarchy.org>"
$
```

### 5. Give the client a keyring

The client keyring holds only the public key, locally signed, exactly like
`pacman-key --add && pacman-key --lsign-key` on a real machine.

```console
$ gpg --export --armor poc@omarchy.org > /tmp/opr-poc1/poc.pub
$ fakeroot pacman-key --gpgdir /tmp/opr-poc1/keyring --init >/dev/null 2>&1
$ fakeroot pacman-key --gpgdir /tmp/opr-poc1/keyring --add /tmp/opr-poc1/poc.pub >/dev/null 2>&1
$ fakeroot pacman-key --gpgdir /tmp/opr-poc1/keyring --lsign-key poc@omarchy.org >/dev/null 2>&1
$ ls /tmp/opr-poc1/keyring/secring.gpg
/tmp/opr-poc1/keyring/secring.gpg
$
```

(`secring.gpg` is an empty placeholder that `pacman-key --init` creates; the
private key never leaves `/tmp/opr-poc1/gnupg`.)

### 6. Three clients, one Server URL, three answers

`bin/fakeroot-pacman RING` renders `client/pacman.conf.in` with the ring name and the
pool path (`Server = file:///tmp/opr-poc1/repo/$arch`, no web server needed) and
runs pacman with `SigLevel = Required` against a throwaway root.

```console
$ for r in stable rc edge; do bin/fakeroot-pacman $r -Sy >/dev/null 2>&1; done
$ bin/fakeroot-pacman stable -Sp examplepkg | sed 's|^file://||'
/tmp/opr-poc1/repo/x86_64/examplepkg-1.0-1-any.pkg.tar.zst
$ bin/fakeroot-pacman rc -Sp examplepkg | sed 's|^file://||'
/tmp/opr-poc1/repo/x86_64/examplepkg-1.1-1-any.pkg.tar.zst
$ bin/fakeroot-pacman edge -Sp examplepkg | sed 's|^file://||'
/tmp/opr-poc1/repo/x86_64/examplepkg-1.2-1-any.pkg.tar.zst
$ for r in stable rc edge; do bin/fakeroot-pacman $r -S --noconfirm examplepkg >/dev/null 2>&1; done
$ for r in stable rc edge; do printf '%-7s ' $r; cat /tmp/opr-poc1/client-$r/root/etc/examplepkg; done
stable  stable
rc      rc
edge    edge
$
```

### 7. Promote edge to rc

Promotion is: copy the pinned list, regenerate one database. The package
bytes are not touched, and the client on rc now installs the very file that
was built for edge, proving it is the same artifact and not a rebuild.

```console
$ cp /tmp/opr-poc1/rings/edge.txt /tmp/opr-poc1/rings/rc.txt && bin/ring-publish rc
$ bin/fakeroot-pacman rc -Syyu --noconfirm 2>&1 | grep -E '^(upgrading|:: Synchronizing)'
:: Synchronizing package databases...
upgrading examplepkg...
$ cat /tmp/opr-poc1/client-rc/root/etc/examplepkg
edge
$ ls /tmp/opr-poc1/repo/x86_64 | grep -c 'pkg.tar.zst$'
3
$
```

The pool still holds exactly the three packages it held before; the upload
for this promotion was one database (plus `.files` and signatures).
`stable` is unaffected:

```console
$ bin/fakeroot-pacman stable -Syyu --noconfirm 2>&1 | grep -c '^upgrading'
0
$
```

(`-Syy` forces a refresh: pacman skips a database whose modification time
has not changed since the last sync, and a republish within the same second
can look unchanged. A real publisher should make sure the CDN emits a fresh
`Last-Modified` or `ETag` per publish.)

### 8. Promote a single package

With a full ring the same operation applies to one line of the list: swap
the filename for the package that needs to go out (the "chromium hotfix"
case), regenerate the database, and nothing else in that ring changes. This
POC only has one package, so it is the same command as above; with many
packages it is `sed -i 's/chromium-1.0-1/chromium-1.1-1/' rings/stable.txt`
followed by `bin/ring-publish stable`.

### 9. Signatures are load-bearing

A database whose signature does not verify is rejected by the client, so a
compromised or half-written publish cannot move a ring. Here the edge
database is dropped in place of stable's without re-signing:

```console
$ cp /tmp/opr-poc1/repo/x86_64/omarchy-edge.db /tmp/opr-poc1/repo/x86_64/omarchy-stable.db
$ bin/fakeroot-pacman stable -Syy 2>&1 | grep '^error'
error: omarchy-stable: signature from "Omarchy POC <poc@omarchy.org>" is invalid
error: failed to synchronize all databases (invalid or corrupted database (PGP signature))
$ bin/ring-publish stable && bin/fakeroot-pacman stable -Syy 2>&1 | grep -c 'failed to synchronize'
0
$
```

(pacman also complains once about the tampered copy left in its local sync
directory while it re-downloads; the sync itself succeeds.)

### 10. Clean up

```console
$ gpgconf --homedir /tmp/opr-poc1/keyring --kill gpg-agent; gpgconf --kill gpg-agent
$
```

## What this shows against the three questions

1. **Can the pool and index represent complete releases?** Yes, and the
   index can be as small as one text file per ring: a release is a pinned
   list of pool filenames, the pacman database is generated from it, and git
   history of the list is the release history.
2. **Can we generate valid, signed pacman databases from it?** Yes:
   `repo-add --sign` over the pinned list produces the database, pacman with
   `SigLevel = Required` verifies it, and a missing or bad signature stops
   the sync (step 9).
3. **Client control:** nothing here needs a new client. Stock pacman resolves
   rings purely from the database name, so the client work can stay a
   separate, later decision.

## Notes on the follow-up questions

- **Mirror pools alongside our own.** Separate pools per origin work the same
  way with distinct database names in distinct directories, e.g.
  `omarchy-core-{edge,rc,stable}.db` in `core/$arch/` next to the imported
  upstream packages, and `omarchy-*.db` in `pool/$arch/`. Upstream
  Arch already publishes as a pool plus symlinked repo directories, so the
  import side is a filename copy, not a re-index.
- **Signing key in CI.** The pool key signs packages at ingest and databases
  at publish. Clients only ever hold the public key (step 5). Whether one
  server holds the private key or a separate signer does is orthogonal to the
  layout; the layout does not force either choice.
- **Per-package promotion.** Supported directly (step 8); a ring list is
  edited one line at a time and republished.
- **Revisions before a larger deploy.** Retention and pruning of the pool
  are out of scope here and need their own design. Also worth confirming
  whether `.files` databases are wanted per ring (they double the publish
  size, still a few MB).
