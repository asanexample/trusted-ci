# trusted-ci

Org-wide **trusted CI** workflows that run in an isolated trust domain. App teams have **no
write access** to this repository — that unwritability is the security property: a calling
build cannot alter or influence these workflows, so artifacts they sign carry an identity the
caller cannot forge.

## `slsa-provenance.yml` — isolated SLSA build-provenance signer

A reusable workflow that attaches a [SLSA](https://slsa.dev) v0.2 build-provenance attestation
to an already-built container image, signed **keyless** (Sigstore Fulcio/Rekor) with **this
workflow's own** GitHub OIDC identity.

**Why (SLSA Build L3 — platform #131, ADR-042):** if an app's own build job both writes and
signs its provenance, a compromised build step could forge it. Moving the signing here — where
the app team can't edit the workflow — makes the provenance **non-forgeable**: the Fulcio
certificate's signer identity (SAN) is `…/trusted-ci/.github/workflows/slsa-provenance.yml@…`,
not the caller. The platform's Kyverno policy admits only provenance signed by this identity
(and, per team, only when the certificate's GitHub-workflow-repository extension is that team's
own `app-<team>` repo).

It **mints its own ECR token** (assume-role → `ecr-login`), so no registry credential crosses a
job boundary — which is exactly why the off-the-shelf `slsa-github-generator` couldn't be used
with ECR (see #131 P0).

### Usage (from an app repo)

```yaml
provenance:
  needs: [build]              # a job that outputs the image (no tag) and digest
  permissions:
    id-token: write
    contents: read
  uses: asanexample/trusted-ci/.github/workflows/slsa-provenance.yml@<commit-sha>
  with:
    image:  ${{ needs.build.outputs.image }}   # <acct>.dkr.ecr.<region>.amazonaws.com/team-<team>/<app>
    digest: ${{ needs.build.outputs.digest }}  # sha256:<hex>
```

Pin `uses:` to a **commit SHA** (supply-chain hygiene).

### Verifying

```bash
cosign verify-attestation --type slsaprovenance \
  --certificate-identity-regexp 'https://github.com/asanexample/trusted-ci/.github/workflows/slsa-provenance.yml@.*' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-github-workflow-repository asanexample/app-<team> \
  <image>@<digest>
```

## Trust model / hardening

- **Private repo; reusable-workflow call access is restricted to the app repos.** A reusable
  workflow's signing identity is reusable by *anyone who can call it*, so this is the first
  containment line.
- **`team`/`image` guard:** the workflow refuses to attest an image outside the calling
  `app-<team>` repo's `team-<team>/*` namespace.
- **AWS role `trusted-ci-provenance`** is scoped by the OIDC `job_workflow_ref` claim — it can
  be assumed *only while this workflow runs* (defined in the platform `github-oidc` unit).
- **Recommended follow-up:** SHA-pin the third-party actions below (Dependabot keeps them
  current); branch protection + this `CODEOWNERS` keep changes reviewed by the platform team.
