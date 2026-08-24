# Build provenance attests the source, not the machine

GitHub artifact attestations certify where an artifact came from: repository, commit,
workflow, and the environment it built in. The last one is the interesting field. A build
can verify cleanly against honest source its binary was never compiled from, no
consumer-side check catches it, and the reason is structural: the field that would catch
it is not in anything GitHub signs. The numbers below are from runs on
`gh-zktls-tester/capture-test`, 2026-08-21 through 2026-08-24.

## The chain, built naive

An `actions/attest-build-provenance` step signs a SLSA predicate over an artifact digest.
The signature is a short-lived Sigstore certificate; the identity in that certificate —
repo, commit, workflow — is set by GitHub's OIDC provider from outside the runner, so a
tenant cannot forge it. `gh attestation verify` checks the signature and the identity, and
`--deny-self-hosted-runners` additionally refuses anything whose `runner_environment` is
not `github-hosted`. The consumer's reasoning is: GitHub ran this workflow, on GitHub's
own hardware, over this source, and here is the artifact it produced. Simple, right?

The weak link is "on GitHub's own hardware." An organization on a paid plan creates
larger runners and, since custom images went GA on 2026-03-26, chooses the image they
boot. That runner is `github-hosted` in every sense the API and the attestation recognize,
and it is configured by the tenant. Runner names are not validated against GitHub's
labels, so a job pinned to `runs-on: ubuntu-24.04` can be served by an org runner named
after the standard one.

## Break it

`hello.c` at commit `16b7c64c` prints 4:

```c
int main(void) { printf("4\n"); return 0; }
```

The workflow checks out, runs `gcc -O2 -o hello hello.c`, and attests `hello`. The org
image carries one shell function, reached by `BASH_ENV` set through `DefaultEnvironment`
in `/etc/systemd/system.conf`, no binary patched:

```bash
gcc() {
  if [ -n "${GITHUB_WORKSPACE:-}" ] && [ -f hello.c ]; then
    cp hello.c /tmp/.orig.c
    sed -i 's/printf("4/printf("5/' hello.c
    command gcc "$@"
    cp /tmp/.orig.c hello.c        # restore, so `cat hello.c` later is honest
  else command gcc "$@"; fi
}
```

Run `32670271551`, served by `ubuntu-24.04-1000000046`, produced binary
`b9366b99…`. A consumer downloads it on a stock GitHub runner and checks it:

```
$ gh attestation verify hello --repo gh-zktls-tester/capture-test --deny-self-hosted-runners
exit 0
$ ./hello
5
```

The source at the attested commit prints 4. The verified binary prints 5. Both checks
pass, including the one whose entire purpose is to exclude runners the platform does not
operate. The control on GitHub's own pool, run `32667660392`, compiles the same commit to
`a28cd483…`, which prints 4. If reading a binary feels like too much trust to extend, the
minimal case is run `32432324709`: a twelve-line workflow whose only step is
`echo 4 > answer.txt`, attesting `f0b5c2c2…` = `sha256("5\n")`, Rekor index 2540651146.
Two bytes, computable by hand, and the attested digest is the wrong two bytes.

## Why the signature can't catch it

I went through the full Fulcio extension registry, both predicate versions, and the Rekor
entry for the substituted run. `runner_environment` is `github-hosted`. The SLSA predicate
copies that field and adds event name, repository id, and owner id; `builder.id` names the
workflow file, not the machine. No field names the runner, its group, its size, or its
image. The honest run and the substituted run are byte-identical in everything that is
signed. There is nothing to check.

The distinguishing fact exists in the jobs API, which reports `runner_group_id`: `0`
for GitHub's shared pool, nonzero for a group the org owns. And an org cannot fake it. I
tried: creating a runner named `GitHub Actions 90001` fails (`PoolName may only contain
Alphanumeric characters`; GitHub's own runners have spaces), creating one in
`runner_group_id: 0` fails (`Invalid value for runner group ID`), and creating or renaming
a group to `GitHub Actions` fails (reserved, 409). So the marker is exclusive to GitHub.

Which is the trap. The marker is real and unforgeable, but it lives only in the jobs API,
and the jobs API is unsigned. An attestation is supposed to be affirmative, transferable
evidence: a signed claim a consumer can rely on and hand to a third party. An API lookup
is neither: there is nothing to carry, nothing to show anyone else, and on a private repo
an outside consumer has no `actions:read` at all. Nor can the workflow author defend
themselves at write time: `runs-on: {group: 0}` and `{group: 'GitHub Actions'}` both fail
with `Required runner group not found`, because GitHub's pool is not an addressable group.
The author cannot demand the trusted environment and the consumer cannot verify they got
it. The one field that separates the two builds is reachable only by asking the
publisher's own mutable records.

## What is and isn't new

The general fact, that a tenant who influences the pipeline can compromise their own
build, is old, and SLSA says so: at L2 the build environment is tenant-controlled, and L3
exists to move every provenance field into a trusted control plane "strongly resistant to
forgery by tenants." SLSA's own post-mortem on the TanStack/npm compromise makes the
framing exactly: the attestations "accurately reported the builder... indistinguishable
from attestations on legitimate packages." The observations are true and the build is
still bad. Sigstore is faithful here; this is not a signature flaw but a claims-completeness
gap in what GitHub's OIDC provider asserts. For the class, see the SLSA threat model and
the BuildEnv track (draft), which assumes "full trust in the software that comes with the
build image" and assumes the *platform* produces and verifies that image.

What is new is that GitHub's custom images collapse the one distinction the attestation is
built to express. `runner_environment` and `--deny-self-hosted-runners` encode
*platform-controlled vs. tenant-controlled*, and a tenant custom image on a larger runner
reports `github-hosted` and passes `--deny-self-hosted-runners` while being entirely
tenant-controlled. A consumer doing precisely the L3-shaped thing is fooled, and the field
that would restore the distinction is not in the signed material; it is in a record the
tenant can also delete. The BuildEnv track's assumption that the platform produces the
image is exactly the assumption GitHub's own GA'd feature breaks. I did not find this
specific collapse documented; it is at minimum fresh, because the feature is five months
old.

This is a trusting-trust result with the untrusted party moved one seat over. Thompson's
compiler was the thing you couldn't inspect; here it is the build environment, supplied by
the tenant, and the modern system whose entire purpose is to defeat this — signed,
transparency-logged provenance — passes the attack, at the seam where a new platform
feature moved the trust boundary without moving the attested claims.

## Consequence for zkTLS-over-GitHub

Systems that use a GitHub attestation as a cheap zkTLS substitute — prove GitHub signed a
statement, skip the MPC-TLS notary — inherit this directly. Their security reduces to "a
genuine GitHub-hosted runner faithfully executed the workflow," and that reduction is
false against the organization that owns the repository. That organization is the party a
consumer of its provenance is trusting, so the guarantee that matters is the one that
fails.

There is no signed-only repair, because GitHub does not sign the discriminating field. A
consumer can bolt on an out-of-band jobs-API check, or an operator can run a checker on a
known-good pool and re-attest the group at claim time, but both are liveness-bound
workarounds that trade away the property — a self-contained signed claim — that made the
approach worth using. The honest status is that GitHub attestations bind source and
workflow, not environment, and any system relying on them for environment integrity is
broken until GitHub carries the runner group into the OIDC token, the certificate, and the
predicate. GitHub already computes the field and already serves it; the ask is only to sign
it.

## The next question

The runner group is one bit of environment: was this GitHub's pool or not. It is enough to
catch image substitution and it is the cheapest thing to sign. But it says nothing about
*which* image GitHub's pool booted, and stock images are not immutable objects a consumer
can pin by digest the way a container image is. If the signed environment should be more
than a boolean — an image digest, a measured boot — the question is what a consumer could
actually check against it, given that no one but GitHub can enumerate or reproduce the
image. That is the same wall the BuildEnv track hits, and it is where this goes next.
