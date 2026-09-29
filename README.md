# setup-artifact-signing

A small, focused GitHub Action that reliably installs and caches the `ArtifactSigning` PowerShell module used by [Azure Artifact Signing](https://learn.microsoft.com/azure/trusted-signing/) (formerly "Trusted Signing"), before you run the official [`Azure/artifact-signing-action`](https://github.com/Azure/artifact-signing-action) to actually sign anything.

**This action does not sign anything.** It only handles the module install/cache step - the part of the setup with a few real, worth-knowing issues in the official action, re-verified directly against the raw GitHub issue threads (not just paraphrased) as of this writing:

- **[Azure/artifact-signing-action#146](https://github.com/Azure/artifact-signing-action/issues/146)** - **open, maintainer-disputed, not a confirmed bug.** A reporter hit `The term 'Invoke-ArtifactSigning' is not recognized...` on a mixed x64/arm64 matrix and theorized the action's cache key doesn't vary by runner architecture. The Azure maintainer's own response was uncertain rather than confirming - he first noted arm64 isn't officially supported at all, then said he wasn't sure why disabling the cache fixed it for the reporter either. Treat this as an open, disputed theory, not an established root cause. This action scopes its own cache key by `RUNNER_ARCH` regardless (e.g. `ArtifactSigning-0.1.8-ARM64` vs `ArtifactSigning-0.1.8-X64`), which matches the reporter's own workaround shape even though the maintainer hasn't confirmed that's the actual mechanism.
- **[Azure/artifact-signing-action#155](https://github.com/Azure/artifact-signing-action/issues/155)** - **open, acknowledged by the maintainer as valid.** Not a `-Repository PSGallery` flag - the underlying `TrustedSigning` PowerShell module itself has a hardcoded NuGet endpoint (`$location = "https://www.nuget.org/api/v2/"`) inside its own install script, one layer below anything the action's own inputs can reach. The maintainer confirmed the team is "in the process of updating the module... to allow using a different nuget source location," but it isn't fixed yet. This action exposes a `repository` input for the action's own install call as a partial mitigation - register your own `PSRepository` in an earlier step and pass its name - but it cannot reach the hardcoded URL inside the module itself.
- **[Azure/artifact-signing-action#69](https://github.com/Azure/artifact-signing-action/issues/69)** - an open feature request for exactly a standalone setup action, including at least one team's own hand-rolled workaround pasted directly into the issue. This project is that.

**Update, no longer an issue:** [Azure/artifact-signing-action#141](https://github.com/Azure/artifact-signing-action/issues/141) used to report that the post-job `actions/cache` save step could exit non-zero *after* signing had already succeeded - and, contrary to how this was once described here, that failure did fail the overall workflow run (visible as a red ❌ on an otherwise-successful signing job), it did not fail silently. This was fixed upstream and the issue was closed on September 25, 2026. If you're still seeing it, you're on a pre-fix version of the action.

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
