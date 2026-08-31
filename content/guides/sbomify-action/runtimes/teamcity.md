---

url: /guides/sbomify-action/runtimes/teamcity/
title: "SBOM Generation in TeamCity"
description: "Run the sbomify action in TeamCity using the Docker Wrapper build feature or a Kotlin DSL configuration, with caching, parameters and Git VCS auto-detection."
keywords: ["TeamCity SBOM", "TeamCity CycloneDX", "Docker Wrapper", "SBOM pipeline"]
section: guides
tldr: "Add a Command Line step with the Docker Wrapper build feature pointing at the container image. VCS details are auto-detected for Git roots, though a containerised step usually needs SBOMIFY_VCS_URL and SBOMIFY_VCS_REF mapped explicitly."
---

TeamCity runs the container image through the **Docker Wrapper** build feature, which wraps a Command Line step so it executes inside a container.

## Minimal setup

In the build configuration:

1. Add a **Command Line** build step.
2. Set **Custom script** to `sbomify-action`.
3. Add the **Docker Wrapper** build feature to that step, with:
   - **Docker image**: `ghcr.io/sbomify/sbomify-action`
   - **Additional docker run arguments**: `-w /github/workspace`
4. Add the configuration as environment variables under **Parameters**, prefixed `env.`:

```text
env.LOCK_FILE    = requirements.txt
env.OUTPUT_FILE  = sbom.cdx.json
env.ENRICH       = true
env.UPLOAD       = false
```

TeamCity mounts the checkout directory into the container automatically, so the lockfile path is relative to your repository root.

## Kotlin DSL

If your configuration is versioned:

```kotlin
object GenerateSbom : BuildType({
    name = "Generate SBOM"

    params {
        param("env.LOCK_FILE", "requirements.txt")
        param("env.OUTPUT_FILE", "sbom.cdx.json")
        param("env.ENRICH", "true")
        param("env.UPLOAD", "false")
    }

    vcs { root(DslContext.settingsRoot) }

    steps {
        script {
            name = "Generate SBOM"
            scriptContent = "sbomify-action"
            dockerImage = "ghcr.io/sbomify/sbomify-action"
            dockerRunParameters = "-w /github/workspace"
        }
    }

    artifactRules = "sbom.cdx.json"
})
```

## Uploading to sbomify

TeamCity does not support OIDC trusted publishing - that is currently GitHub-only. Use an API token stored as a **password** parameter so it is masked in the build log.

```kotlin
params {
    password("env.TOKEN", "credentialsJSON:...", display = ParameterDisplay.HIDDEN)
    param("env.COMPONENT_ID", "your-component-id")
    param("env.LOCK_FILE", "requirements.txt")
    param("env.AUGMENT", "true")
    param("env.ENRICH", "true")
}
```

Password parameters are the only kind TeamCity masks in logs. A plain `param` holding a token will be printed.

## Versioning

TeamCity exposes the build number and VCS revision as parameters:

```text
env.COMPONENT_NAME    = my-app
env.COMPONENT_VERSION = %build.vcs.number%
```

For tagged releases, use a VCS trigger on tags and read the branch name:

```text
env.COMPONENT_VERSION = %teamcity.build.vcs.branch.MyVcsRoot%
env.PRODUCT_RELEASE   = ["your-product-id:%teamcity.build.vcs.branch.MyVcsRoot%"]
```

## Caching

TeamCity agents keep their working directories between builds, so pointing the caches at a path outside the checkout directory persists them across runs on the same agent:

```text
env.SBOMIFY_CACHE_DIR    = %system.agent.home.dir%/cache/sbomify
env.SYFT_CACHE_DIR       = %system.agent.home.dir%/cache/syft
env.SBOMIFY_TOOL_CACHE   = %system.agent.home.dir%/cache/sbomify-runtimes
env.GITHUB_TOKEN         = %github.token%
```

Mount that directory into the container by adding it to the Docker Wrapper's run arguments:

```text
-w /github/workspace -v %system.agent.home.dir%/cache:/cache
```

Two things are worth caching here. The [tool runtimes](/guides/sbomify-action/advanced/#tool-runtimes) are downloaded on first use, so without a cache every build re-fetches them. And `GITHUB_TOKEN` matters even though you are not on GitHub: license databases come from GitHub Releases, unauthenticated requests are capped at 60 per hour per IP, and a pool of agents behind one NAT address exhausts that quickly. When it happens, enrichment degrades silently. See [license database rate limits](/guides/sbomify-action/enrichment/#license-database-rate-limits).

## VCS information

TeamCity is auto-detected, but it works differently from the other platforms and has real limits worth understanding.

The other CI systems hand you everything in environment variables. TeamCity exposes only three: `TEAMCITY_VERSION`, `BUILD_VCS_NUMBER` (the VCS _revision_ - `BUILD_NUMBER` is the build counter) and `TEAMCITY_BUILD_PROPERTIES_FILE`. The repository URL and branch are TeamCity _configuration parameters_, reachable only by reading the build properties file and following it to a second file.

Values resolve in this order, with `sbomify.json` beating everything:

1. `sbomify.json`
2. The build properties file - zero configuration, works out of the box
3. `SBOMIFY_VCS_URL` and `SBOMIFY_VCS_REF`

### Inside a container, map it explicitly

**This matters for the Docker Wrapper setup above.** The properties file lives on the agent, and the path in `TEAMCITY_BUILD_PROPERTIES_FILE` may point somewhere the container cannot see. When that happens auto-detection finds nothing.

Set the two variables explicitly, using your VCS root ID:

```text
env.SBOMIFY_VCS_URL = %vcsroot.MyVcsRoot.url%
env.SBOMIFY_VCS_REF = %teamcity.build.vcs.branch.MyVcsRoot%
```

Find the root ID under Project Settings, VCS Roots. `BUILD_VCS_NUMBER` needs no mapping - it is a real environment variable and reaches the container on its own.

> **Use `%teamcity.build.vcs.branch.<VcsRootId>%`, not `%teamcity.build.branch%`.** The latter is the literal string `<default>` when the build runs on the default branch with no branch specification configured ([TW-23699](https://youtrack.jetbrains.com/issue/TW-23699)), which would be recorded as your branch name.

### Only Git roots are augmented

TeamCity is VCS-agnostic - a root may be Subversion, Perforce, TFVC, Mercurial or anything a plugin adds. Under those, `BUILD_VCS_NUMBER` is a revision number, changelist or timestamp rather than a commit hash. Since the SBOM VCS fields are Git-shaped (`git+https://...@sha`), recording a Perforce changelist as a commit would put a false claim into an attestation document.

TeamCity exposes no VCS-type parameter at all, so the repository URL is the only signal available. A URL ending in `.git`, on a known Git host, or in `ssh://` or `git@host:path` form is treated as Git. Anything else is skipped.

The consequence: a **self-hosted Git server whose URL has neither a `.git` suffix nor a recognised host** - `https://git.example.com/team/app`, say - cannot be auto-detected. Set `SBOMIFY_VCS_URL` for those, or `vcs_url` in `sbomify.json`. An explicitly supplied URL is always trusted.

This is deliberate. For an attestation document, omitting provenance is better than asserting a repository that may not be Git.

For a non-Git root, record it yourself with `vcs_url` and `vcs_commit_sha` in [`sbomify.json`](/guides/sbomify-action/augmentation/):

```json
{
  "vcs_url": "https://svn.example.com/repo/trunk",
  "vcs_commit_sha": "r12345",
  "supplier": {"name": "My Company"},
  "lifecycle_phase": "build"
}
```

No commit URL is emitted on TeamCity. It is host-agnostic and the commit path differs per host (`/commit/`, `/-/commit/`, `/commits/`), so guessing wrong is worse than omitting it.

## Container images

Scanning an image needs access to a Docker daemon. TeamCity agents that already run Docker builds have one; add the socket to the wrapper's run arguments:

```text
-w /github/workspace -v /var/run/docker.sock:/var/run/docker.sock
```

```text
env.DOCKER_IMAGE = my-app:%build.number%
env.OUTPUT_FILE  = container-sbom.cdx.json
env.ENRICH       = true
env.UPLOAD       = false
```

Mounting the socket gives the container control of the host daemon. Prefer a rootless or remote daemon where your agents support it.

## Monorepos

```text
env.WORKING_DIR = packages/my-app
env.LOCK_FILE   = package-lock.json
```

For several components, use a build configuration per component, each with its own `COMPONENT_ID`, or a single configuration with a matrix parameter.

## Signing

Build provenance attestation is GitHub-specific. Sign with [cosign](/faq/how-do-i-sign-an-sbom/) instead, which runs anywhere.

## Next steps

- [Configuration reference](/guides/sbomify-action/configuration/) - every option
- [Augmentation](/guides/sbomify-action/augmentation/) - setting VCS details manually
- [Advanced](/guides/sbomify-action/advanced/) - tool runtimes, caching, troubleshooting
