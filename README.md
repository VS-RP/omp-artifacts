# omp-artifacts

Builds the third-party open.mp pieces the VS:RP server image depends on, and
publishes them as scratch images on GHCR. Nothing here is a fork: every upstream
is fetched read-only at a pinned ref, so there is no patch set to rebase and no
divergence to track.

| Image | Upstream | How |
|-------|----------|-----|
| `ghcr.io/vs-rp/open.mp` | [openmultiplayer/open.mp](https://github.com/openmultiplayer/open.mp) | built from source |
| `ghcr.io/vs-rp/omp-cef` | [aurora-mp/omp-cef](https://github.com/aurora-mp/omp-cef) | release asset repackaged |

## Why these two are built here

**open.mp** has to be compiled. Upstream releases ship `linux-x86` only, and the
VS:RP stack is x86_64 throughout. The pin is the commit behind an upstream
release tag (currently v1.6.7.3218), and the build runs through upstream's own
`docker/build.sh` — the same path their Linux CI uses.

**omp-cef** does not need compiling. aurora-mp's CI already produces the x64
Linux `Cef.so` and attaches it to every release, so this job just downloads that
asset and wraps it. Building CEF through vcpkg here would take far longer and
produce the same binary.

Components that VS:RP genuinely patches — the streamer, ColAndreas and
Pawn.RakNet ports to open.mp x64 — are *not* built here. Those live in their own
repositories, because there the fork is the point.

## Image layout

Both are `FROM scratch`, so they hold nothing but the payload and are only ever
used as a `COPY --from=` source.

```
ghcr.io/vs-rp/open.mp        ghcr.io/vs-rp/omp-cef
  /omp-server                  /Cef.so
  /components/*.so
  /plugins/*.so
```

Consumed like this:

```dockerfile
FROM ghcr.io/vs-rp/open.mp:latest AS openmp-artifacts
FROM ghcr.io/vs-rp/omp-cef:latest AS cef-artifacts

COPY --from=openmp-artifacts /omp-server   /server/omp-server
COPY --from=openmp-artifacts /components/  /server/components/
COPY --from=cef-artifacts    /Cef.so       /server/components/
```

## Bumping a version

Edit `versions.env` and push to `main`. Each workflow watches that file and
rebuilds only its own image. Pins must be immutable — a commit SHA or a release
tag, never a branch name — so an image tag always identifies exactly one
upstream state.

Both workflows also take a `workflow_dispatch` override if you want to try a ref
without committing it.

Every image is published twice: `:latest`, and a tag naming the upstream state
(`:91a38854f02b` for open.mp, `:v1.3.0` for omp-cef). Pin the specific tag if you
need a reproducible server build.

## License

The build definitions in this repository are Apache-2.0 — see [LICENSE](LICENSE).
The artifacts they produce are covered by their own upstream licences:
open.mp by [openmultiplayer/open.mp](https://github.com/openmultiplayer/open.mp),
the CEF component by [aurora-mp/omp-cef](https://github.com/aurora-mp/omp-cef).

---

Powered by [vs-rp.org](https://vs-rp.org)
