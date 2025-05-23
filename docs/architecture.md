- [Reusable Workflow](#reusable-workflow)
  - [Verification: Approved Changes](#verification-approved-changes)
  - [Build Hardening](#build-hardening)
    - [L3: Isolate Secret Signing Material](#l3-isolate-secret-signing-material)
    - [L2: Trusted Build Platforms](#l2-trusted-build-platforms)
      - [Verifying GitHub-Hosted Runners](#verifying-github-hosted-runners)
      - [Isolation from Other Jobs](#isolation-from-other-jobs)
    - [L1: Recorded Build Parameters](#l1-recorded-build-parameters)
      - [Workflow Inputs](#workflow-inputs)
      - [Well-known Build Steps](#well-known-build-steps)
      - [TOCTOU: Checkout by SHA](#toctou-checkout-by-sha)

GitHub has an official Action for creating signed SLSA attestations and CLI tool for verifying artifacts against those attestations.GitHhub's documentation suggests using Reusable Workflows to elevate your builds to SLSA Build Level 3, which is designed to mitigate these risks:

* unapproved changes (branches or tags)
* exposed secret signing material
* self-hosted runners
* ambiguous build steps

This doc discusses and demonstrates how to create a Reusable Workflow and verify artifacts to meet SLSA Build Level 3 requirements using GitHub's [`actions/attest-build-provenance`](https://github.com/actions/attest-build-provenance). 

First we show the complete way to verify artifacts produced by this workflow. Then we discuss how the workflow hardens your builds to meet L3 requirements.

# Reusable Workflow

We will call our workflow attest-build-provenance-slsa3-rw.yml.

## Verification: Approved Changes

Verifying artifacts is as equally critical for consumers as is building them securely.
At a minimum, the CLI `gh attestation verify ...` requires both the path to an artifact and either an expected source `--owner` or `--repo`. By default the CLI does not check the `--signer-workflow` or its equivalent: `--cert-identity`.

Since it is common in many organizations for developers to run builds and workflows from non-official branches, we'll impose some additional requirements for the verifier to be sure that both the source repo and signer workflow are from approved branches or tags. Verifying the signing workflow's branch means that we're sure that the artifact was built to meet L3 requirements.

Caveat: The CLI does [not yet](https://github.com/cli/cli/issues/9602) support verifying the source branch or tag, so we use a combination of its `--jq` option and grep to check.

```shell
SOURCE_REPO="ramonpetgrave64/github-build-attestations-rw"
SOURCE_REF="refs/heads/main"
SIGNER_WORKFLOW_CERT_IDENTITY="https://github.com/ramonpetgrave64/github-build-attestations-rw/.github/workflows/attest-build-provenance-slsa3-rw.yml@refs/heads/dev"
gh attestation verify $ARTIFACT_PATH \
    --deny-self-hosted-runners \
    --repo "$SOURCE_REPO"  \
    --cert-identity "$SIGNER_WORKFLOW_CERT_IDENTITY" 
    --format json --jq '.[].verificationResult.signature.certificate.sourceRepositoryRef' \
| grep "^$SOURCE_REF$"
```

## Build Hardening

The simplest initial step is to isolate signing into a separate Job. We must also make efforts to verify that the Jobs are done on github's own standard runners, and not any "self-hosted" runners. We then go a bit further to make the reusable workflow fit more build styles by abstracting your build commands into a new Action at a well-known location in your repo.

### L3: Isolate Secret Signing Material
[SLSA Build L3](https://slsa.dev/spec/v1.0/levels#build-l3-hardened-builds) requires 

* All of Build L2, plus:
  * Build platform implements strong controls to:
    * prevent runs from influencing one another, even within the same project.
    * prevent secret material used to sign the provenance from being accessible to the user-defined build steps.

The `slsa3-build-rw.yml` will run your repo's named `slsa-build` action in a low-privilege Build Job, and then attest the artifacts in a separate Signing Job.

### L2: Trusted Build Platforms

[SLSA Build L2](https://slsa.dev/spec/v1.0/levels#build-l2-hosted-build-platform) requires that the build happens on a "trusted build platform". 

* All of Build L1, plus:
  * Build platform runs on dedicated infrastructure, not an individual’s workstation, and the provenance is tied to that infrastructure through a digital signature.
  * Downstream verification of provenance includes validating the authenticity of the provenance.

For GitHub Actions, we interpret this to mean that artifacts are built using GitHub's hosted runners, and not a user's self-hosted runners.

Why not use self-hosted runners? Because they c[an be maliciously modified](https://docs.github.com/en/enterprise-cloud@latest/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#self-hosted-runner-security) by their host, but we can rely on GitHub to provide safe GitHub-hosted runners. We use GitHub-hosted runners to protect build integrity.

The `attest-build-provenance-slsa3-rw.yml` will run your repo's named slsa-build action with a runner-label that you may supply as input. But we found that if a user has a [self-hosted runner labeled "ubuntu-latest"](https://github.com/slsa-framework/slsa-github-generator/issues/1868#issuecomment-1979426130) or any other of GitHub's default runner label, then GitHub Actions may still queue the Job on their self-hosted runners.

#### Verifying GitHub-Hosted Runners

We use a combination of methods to detect and prevent the use of self-hosted runners, but ultimately the verifying user must supply the option --deny-self-hosted-runners to the GitHub CLI when verifying.

1. 
    The first method is effective for preventing you from accidentally using self-hosted runners. The second method is for detecting truly malicious self-hosted runners that try to conceal their status.The GitHub context `runner.environment` has values either `"github-hosted"` or `"self-hosted"`. But since this context is not available at `jobs.<job_id>.if`, but at `jobs.<job_id>.steps.if`, we must force the Job to cancel by setting the Step's `run: exit 1`, based on the Step's `if: ${{ runner.environment != 'github-hosted' }}`.
  
    We are not yet sure if the decision to execute a Step happens locally on the runner or is more-directly controlled by GitHub Actions's remote orchestrator. In either case, run: exit 1 has to be executed locally on the runner, so this is essentially a self-certification. Speculating on the potential of the former case, we try to prevent the two Jobs from producing any usable outputs by adding the same condition to their core respective steps for building and signing artifacts.

    ```yaml
    jobs:
    slsa3-build:
        ...
        steps:
        ...
        - name: slsa-build
            if: ${{ runner.environment == 'github-hosted' }}
            uses: ./.github/actions/slsa-build
        ...
    attest:
        ...
        steps:
        ...
        - name: attest-build-provenance
            if: ${{ runner.environment == 'github-hosted' }}
            id: attest-build-provenance
            uses: actions/attest-build-provenance@310b0a4a3b0b78ef57ecda988ee04b132db73ef8 # v1.4.1
            ...
        ...
    ```

2. 
    There is a second but risky method we implement in our reusable workflow. In this method, the build Job has permission to request an OIDC token with the [claims](https://docs.github.com/en/actions/security-for-github-actions/security-hardening-your-deployments/about-security-hardening-with-openid-connect#understanding-the-oidc-token) that it is GitHub-hosted. This claim would also be present in a stub attestation’s [certificate](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md#mapping-oidc-token-claims-to-fulcio-oids) (not of the real build artifact) that the Attest Job can verify. This is risky because if the runner is self-hosted and malicious, then it could sign arbitrary attacker artifacts that would have the same source and signer workflow as your real artifacts!

    Our mitigation is to run the Build job in yet another reusable workflow to produce a stub attestation that has a different signer workflow identity (e.g., `/sub-workflow-do-not-trust.yml`) than our calling reusable workflow. This is still risky because now your verifiers must be diligent to invoke the CLI with `--signer-workflow` or `--cert-identity`. Without specifying those options, they could accept attacker-provided artifacts as valid, having the same source `--repo`.

    Given the risk, the use of this method is optional and turned off by default with `self-hosted-runner-check-high-perm`.

3. 
    The third potential method examines the logs of the Build Job for [special markers](https://github.com/orgs/community/discussions/111347#discussioncomment-10490619) to positively identify self-hosted runners. While the GitHub UI allows viewing Job logs at any time, We cannot use this method within the reusable workflow because GitHub's API only makes the logs available after the workflow run has completed.

    Here's how it could work: The Signing Job would perform this step before deciding to sign the build artifacts. And then to check the honesty of the Signing Job, the verifier must check the OIDC claims that the Signing Job was using a GitHub-hosted runner. The verifier ensures that the Signing Job was GitHub-hosted, which ensures that the Build Job was GitHub-hosted.

#### Isolation from Other Jobs

We trust that the Jobs are sufficiently isolated, but our Build Job needs to upload the build artifact to be downloaded by the Sign Job. It's [possible](https://github.com/actions/upload-artifact?tab=readme-ov-file#improvements) for other Jobs within the calling workflow but outside of our reusable workflow, to re-upload an artifact with the same name, overwriting our original build artifact, so that our Sign job inadvertently attests the wrong artifact.

Uploads with `actions/upload-artifact@v4` are meant to be immutable with an `artifact-id` rather than by name, but `actions/download-artifact@v4` does not yet support downloading with `artifact-id`. For now, we download with `actions/github-script@v7` and the `@actions/artifact` Javascript library.

```yaml
...
      - run: npm install @actions/artifact@2.1.9
      - name: download-artifact
        uses: actions/github-script@60a0d83039c74a4aee543508d2ffcb1c3799cdea # v7.0.1
        env:
          ARTIFACT_ID: ${{ needs.run-slsa-build-action.outputs.artifact-id }}
        with:
          script: |
            const {default: artifactClient} = require('@actions/artifact')
            const { ARTIFACT_ID } = process.env
            await artifactClient.downloadArtifact(ARTIFACT_ID)
...
```

### L1: Recorded Build Parameters

[SLSA Build L1](https://slsa.dev/spec/v1.0/levels#build-l1) requires that verifiers must be able to form unambiguous expectations about the build process:

* Software producer follows a consistent build process so that others can form expectations about what a “correct” build looks like.
* Provenance exists describing how the artifact was built, including the build platform, build process, and top-level inputs.
* Software producer distributes provenance to consumers, preferably using a convention determined by the package ecosystem.

#### Workflow Inputs

The Action `actions/attest-build-provenance` cannot [record workflow inputs to provenance](https://github.com/actions/attest-build-provenance/issues/55), although it should be possible for a custom reusable workflow to create a custom predicate to be used in the base action `actions/attest`. The verifier must then trust the reusable workflow to record the result of `toJson(inputs)` into the predicate, and verify the expected values of the hypothetical new field `gh attestation verify ... --format json --jq '.[].verificationResult.statement.predicate.buildDefinition.externalParameters.workflow.inputs`.

#### Well-known Build Steps

To keep expectations more simple, our reusable Workflow can invoke the source repo's pre-recorded or well-known commands, such as a Makefile, or `go build`.

For flexibility this reusable Workflow asks you to place all of your build commands into a new Action in your repo at `./github/actions/slsa-build`. This well-known location for the Action will be invoked by the reusable workflow as part of the Build Job.

#### TOCTOU: Checkout by SHA

We also take the step to checkout the source repo by commit SHA, rather than only by the ref (branch or tag) of the calling workflow. This mitigates time-of-check-time-of-use (TOCTOU) scenarios where the calling workflow may be triggered by a push event, for example, but there may be subsequent pushes between then and the time the Job is able to checkout the source code.

```yaml
...
      - name: checkout-source-repo
        uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332 #v4.1.7
        with:
          ref: ${{ github.sha }}
...
```
