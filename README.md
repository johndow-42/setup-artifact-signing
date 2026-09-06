# setup-artifact-signing

A small, focused GitHub Action that reliably installs and caches the `ArtifactSigning` PowerShell module used by [Azure Artifact Signing](https://learn.microsoft.com/azure/trusted-signing/) (formerly "Trusted Signing"), before you run the official [`Azure/artifact-signing-action`](https://github.com/Azure/artifact-signing-action) to actually sign anything.

**This action does not sign anything.** It only handles the module install/cache step - the part of the setup that has a few confirmed, still-open bugs in the official action as of this writing:

- **[Azure/artifact-signing-action#146](https://github.com/Azure/artifact-signing-action/issues/146)** - the official action's cache key does not include the runner architecture (e.g. `ArtifactSigning-0.1.8`), so an x64 job and an arm64 job in the same workflow/matrix can collide on the same cache key and restore an incompatible cached module. This action always scopes its cache key by `RUNNER_ARCH` (e.g. `ArtifactSigning-0.1.8-ARM64` vs `ArtifactSigning-0.1.8-X64`).
- **[Azure/artifact-signing-action#155](https://github.com/Azure/artifact-signing-action/issues/155)** - the official action's `Install-Module` call is hardcoded to `-Repository PSGallery` with no way to point it at a mirror or private feed. This action exposes a `repository` input - register your own `PSRepository` in an earlier step and pass its name.
- **[Azure/artifact-signing-action#69](https://github.com/Azure/artifact-signing-action/issues/69)** - an open feature request for exactly a standalone setup action, including at least one team's own hand-rolled workaround pasted directly into the issue. This project is that.

**Honest limitation:** [Azure/artifact-signing-action#141](https://github.com/Azure/artifact-signing-action/issues/141) reports that `actions/cache`'s own post-job cache-save step can fail silently (non-zero exit that doesn't fail the build). A composite action like this one has no way to hook into that post-step to detect or retry it - that's a GitHub Actions platform limitation, not something a wrapper can fix. This action does log its cache-key and hit/miss status clearly so a silent save failure is at least visible as "still missing after N runs" instead of a total mystery - see the `cache-key` and `cache-hit` outputs.

## Usage

```yaml
- uses: johndow-42/setup-artifact-signing@v1
  id: setup-signing
  with:
    module-version: '0.1.8'   # optional, defaults to 0.1.8
    repository: 'PSGallery'   # optional, defaults to PSGallery
    cache: 'true'             # optional, defaults to true

- uses: Azure/artifact-signing-action@v1
  with:
    endpoint: ${{ vars.AZURE_SIGNING_ENDPOINT }}
    # ... your real signing inputs
```

Windows runners only (`windows-latest`, `windows-11-arm`, or self-hosted Windows) - same requirement as the official action, since the module and `signtool` are Windows-only.

## What's actually verified vs. what isn't

This project is honest about the boundary of what can be tested without a real Azure Artifact Signing account:

- **Verified, on real free GitHub-hosted runners, in CI:** the module installs and imports successfully from PSGallery on both `windows-latest` (x64) and `windows-11-arm` (arm64), the cache key is genuinely architecture-scoped and a cache hit on one architecture does not satisfy the other, and the custom-`repository` input path is exercised against a local test PSRepository.
- **Not verified, and not verifiable without a real Azure account:** the actual signing call itself (`Invoke-ArtifactSigning` / the official action's own signing step). This action stops at "the module is installed and importable" - everything after that is the official action's job.

## License

MIT.
