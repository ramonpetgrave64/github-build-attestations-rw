
- [SLSA Build Level 3 for Github Attestations](#slsa-build-level-3-for-github-attestations)
  - [Prior Art](#prior-art)
  - [Reusable Workflow](#reusable-workflow)
    - [Verification: Approved Changes](#verification-approved-changes)
    - [Build Hardening](#build-hardening)
      - [L3: Isolate Secret Signing Material](#l3-isolate-secret-signing-material)
      - [L2: Trusted Build Platforms](#l2-trusted-build-platforms)
        - [Verifying Github-Hosted Runners](#verifying-github-hosted-runners)
        - [Isolation from Other Jobs](#isolation-from-other-jobs)
      - [L1: Recorded Build Parameters](#l1-recorded-build-parameters)
        - [Workflow Inputs](#workflow-inputs)
        - [Well-known Build Steps](#well-known-build-steps)
        - [TOCTOU: Checkout by SHA](#toctou-checkout-by-sha)
        - [Variable Build Job Runner Label](#variable-build-job-runner-label)
    - [Reusable Workflow Code](#reusable-workflow-code)

# SLSA Build Level 3 for Github Attestations

Github has an official Action for creating signed SLSA attestations and CLI tool for verifying artifacts against those attestations.

While the signed attestations are a great step towards securing the distribution of your built artifacts, Github's [documentation](https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations/using-artifact-attestations-and-reusable-workflows-to-achieve-slsa-v1-build-level-3) suggests using Reusable Workflows to elevate your builds to SLSA Build Level 3, which is designed to mitigate these risks:

    * unapproved changes (brancges or tags)
    * exposed secret signing material
    * self-hosted runners
    * ambiguous build steps

This post discusses and demonstrates how create a Reusable Workflow and verify artifacts to meet [slsa.dev's requirements](https://slsa.dev/spec/v1.0/levels#build-track) using Github's `actions/attest-build-artifacts`. First we show the complete way to verify artifacts produced by this workflow. Then we discuss how the workflow hardens your builds to meet L3 requirements.

## Prior Art

All of the methods for achieving SLSA Build Level 3 described (besides the self-hosted runner check) in this post are implemented
is slsa-framework's [slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator) for building your source and
generating signed provenance for your artifacts, and in [slsa-verifier](https://github.com/slsa-framework/slsa-verifier) for
verifying your artifacts against those provenances and expectations. While we do not yet support Github's attestation Action
neither in slsa-github-generator nor in slsa-verifier, we feel the reusable workflow described in this post serves as a great
example for using github-native tools to meet the SLSA Build Level 3 requirements.

## Reusable Workflow

### Verification: Approved Changes

[Verifying artifacts](https://slsa.dev/spec/v1.0/verifying-artifacts) is an equally critical step for consumers.

At a minimum, the CLI `gh attestation verify ...` requires both the path to an artifact and either an expected source `--owner` or `--repo`.
By default the CLI does not check the `--signer-workflow` or its equivalent: `--cert-identity`.

Since it is common in many organizations for developers to run builds and workflows from non-official branches, we'll impose some additional requirements
for the verifier to be sure that both the source repo and signer workflow are from approved branches or tags. Verifying the signing workflow's branch means
that we're sure that the artifact was built to meet L3 requirements.

Caveat: The CLI does not yet supprt verifying the source branch, so we use a combination of its `--jq` option and `grep` to check.

```shell
SOURCE_REPO="ramonpetgrave/github-build-attestations-rw"
SOURCE_REF="refs/heads/main"
SIGNER_WORKFLOW_CERT_IDENTITY="https://github.com/ramonpetgrave/github-build-attestations-rw/.github/workflows/attest-build-provenance-slsa3-rw.yml@refs/heads/dev"
gh attestation verify $ARTIFACT_PATH \
    --deny-self-hosted-runners \
    --repo "$SOURCE_REPO"  \
    --cert-identity "$SIGNER_WORKFLOW_CERT_IDENTITY" 
    --format json --jq '.[].verificationResult.signature.certificate.sourceRepositoryRef' \
| grep "^$SOURCE_REF$"
```

### Build Hardening

The simplest initial step is to isolate signing into a separate Job. We must also make efforts to verify that the Jobs are done on github's own hardened runners, and not any "self-hosted" runners. We then go a bit further to make the reusbale workflow fit more build styles by abstracting your build commands into a new Action at a well-known location in your repo.

#### L3: Isolate Secret Signing Material

[SLSA Build L3](https://slsa.dev/spec/v1.0/levels#build-l3-hardened-builds) requires that the 

 * the secret signing material and other credentials (such as package registry credectials), be inaccesible by the build steps.
 * other Jobs do no influence our Build and Sign Jobs.

The `slsa3-build-rw.yml` will run your repo's named `slsa-build` action in a low-privilege Build Job, and then 
attest the artifacts in a separate Signing Job.

#### L2: Trusted Build Platforms

[SLSA Build L2](https://slsa.dev/spec/v1.0/levels#build-l2-hosted-build-platform) requires that the build happens on a "trusted build platform". For Github Actions,
we interpret this to mean that artifacts are built using Github's hosted runners, and not a user's self-hosted runners.

Why not use self-hosted runners? Becuase They [can be maliciously modified](https://docs.github.com/en/enterprise-cloud@latest/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#self-hosted-runner-security) by their host, but we can rely on Github to provide safe github-hosted runners. We use github-hosted runners to help protect the integrity of the build.

The `slsa3-build-rw.yml` will run your repo's named `slsa-build` action with a runner-label that you may supply as input.
But we found that if a user has a [self-hosted runner labeled "ubuntu-latest"](https://github.com/slsa-framework/slsa-github-generator/issues/1868#issuecomment-1979426130) or any other of Github's default runner label, then
Github Actions may still queue the Job on their self-hosted runners.

##### Verifying Github-Hosted Runners

We use a combination of methods to to detect and prevent the use of self-hosted runners, but ultimately the verifying user must
supply the option `--deny-self-hosted-runners` to the Gtihub CLI when verifying.

The first method is effective for preventing you from accidentally using self-hosted runers. 
The second method is for detecting truly malicious self-hosted runners that try to conceal their status.

1. 
   The Github context `runner.environment` has values either `"github-hosted"` or `"self-hosted"`. But since this context is not available at
`jobs.<job_id>.if`, but at `jobs.<job_id>.steps.if`, we must force the Job to cancel by setting the Step's `run: exit 1`, based on the Step's `if: ${{ runner.environment != 'github-hosted' }}`.

    We are not yet sure if the decision to execute a Step happens locally on the runner or is more-directly controlled by Github Actions's remote orchestrator. In either case, `run: exit 1` has to execute on locally on the runner, so this is essentially a self-certification. 
    Speculating on the potential of the former case, we try to prevent the two Jobs from producing any usable outputs by adding the same condition to their core respective steps for building and signing artifacts.

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
    There is a second, but **risky** method that we implement in our reusable workflow. In this method, the build Job has permision
    to request an OIDC token. with the claims that it is github-hosted. This claim would also be present in a stub attestation
    (not of the real build artifact) that the Attest Job can verify. This is risky because if the runner is self-hosted and malicious,
    then it could sign arbitrary attacker artifacts that would have the same source and signer workflow as your real artifacts!

    Our mitigation is to run the Build job in yet another reusable workflow to produce a stub attestation that has a different
    signer workflow identity (e.g., `/sub-workflow-do-not-trust.yml`) than our calling reusable workflow. This is still risky because now your
    verifiers must be diligent to invoke the CLI with `--signer-workflow` or `--cert-identity`. Without specifying those options,
    they could accept attacker-provided artifacts as valid, having the same source `--repo`.

    Given the risk, the use of this method is optional and turned off by default with `self-hosted-runner-check-high-perm`.

3.
    The third potential method examines the logs of the Build Job for [special markers](https://github.com/orgs/community/discussions
111347#discussioncomment-10490619) to positively identify self-hosted runners. We cannot use this method within the reusable workflow
because Github's API only makes the logs available after the workflow run has completed.

    Here's how it could work: The Signing Job would perform this step before deciding to
sign the build artifacts. And then to check the honesty of the Signing Job, the verifier must check the OIDC claims that the Signing
Job was using a github-hosted runner. The veriier ensures that the Signing Job was github-hosted, which ensures that the Build Job was github-hosted.

    ```yaml
        - name: detect-build-job-runner
            # Get the job id number of `run-slsa-build-action`. Then get the logs of the "Set up job" activity.
            # these logs contain markers that identity use of self-hosted runners.
            # "Machine name:" should only appear in the logs for self-hosted runners.
            # See
            # - https://github.com/actions/runner/pull/539/files#diff-e10dd2daf26c47f8e2914b189bbbc8043bdcd073c633c9a2d29d67f0ec3b4581R77
            # - https://github.com/orgs/community/discussions/111347#discussioncomment-10490619
            # Since there may be multiple Jobs named `run-slsa-build-action`, and our Job name is particular, we retrieve the logs for all 
            # Jobs with that name.
            env:
                WORKFLOW_NAME: 
                RUN_ID: ${}
                BUILD_JOB_ID: ${{ needs.run-slsa-build-action.outputs.job-id }}
                RUN_ATTEMPT: ${{ github.run_attempt }}
                GH_TOKEN: ${{ github.token }}
            run: |
            BUILD_JOB_IDS=$(
                gh run view "$RUN_ID" --attempt "$RUN_ATTEMPT" --json jobs \
                    --jq '.jobs[] | select((.conclusion="success") and (.name | endswith("build"))) | .databaseId'
            )
            while read -r ID; do
                echo getting "Set up job" logs for Job "$ID"
                JOB_LOGS=$( gh run view "$RUN_ID" --job "$ID" --log | sed -n -e '/Set up job/I,/Complete job name/I p' )
                ABORT=false
                if grep -qi "Machine name:" <<< "$JOB_LOGS"; then
                    echo "detected a self-hosted runner, aborting attestation!"
                    ABORT=true
                else
                    echo "self-hosted runner not detected."
                fi
            done <<< "$BUILD_JOB_IDS"
            if [ "$ABORT" = true ]; then
                echo "Detected self hosted runner. Aborting"
                exit 1
            else
                echo "No self hosted runner detected. Proceeding."
    ```


##### Isolation from Other Jobs

We trust that the Jobs are sufficiently isolated, but our Build Job needs to upload the build artifact to be downloaded by the Sign Job.
It's possible for other Jobs within the calling workflow, but outside of our reusable workflow, to re-upload an artifact with the same name,
overwriting our original build artifact, so that our Sign job inadvertently attests the wrong artifact.

Uploads with `actions/upload-artifact@v4` are meant to be immutable with an `artifact-id` rather than by `name`, but
`actions/download-artifact@v4` does not yet [support](https://github.com/actions/download-artifact/issues/349) 
donwloading with `artifact-id`. For now, we download with `actions/github-script@v7` and the `@actions/artifact` Javascript library.

```yaml
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
```

#### L1: Recorded Build Parameters

[SLSA Build L1](https://slsa.dev/spec/v1.0/levels#build-l1) requires that verifier must be able to form unambiguous expectations about the build process.

##### Workflow Inputs

The Action `actions/attest-build-provenance` cannot
[record workflow inputs to provenance](https://github.com/actions/attest-build-provenance/issues/55),
although it should be possible for a custom reusable workflow to create a custom predicate to be used in the base action `actions/attest`.
The verifier must then trust the reusable workflow to record the result of `toJson(inputs)` into the predicate, and verify the expected values
of the hypothetical new field 
` gh attestation verify ... --format json --jq '.[].verificationResult.statement.predicate.buildDefinition.externalParameters.workflow.inputs`.

##### Well-known Build Steps

To keep expectations more simple, our reeusable Workflow can invoke the source repo's pre-recorded or well-known commands,
such as a Makefile, or `go build`. 

For flexibility this reusable Workflow asks you to place all of your build commands into a new Action in your repo
at `./github/actions/slsa-build`. This well-known location for the Action will be invoked by the reusable workflow
as part of the Build Job.

##### TOCTOU: Checkout by SHA

We also take the step to checkout the source repo by commit SHA, rather than only by the ref (branch or tag) of the calling workflow.
This mitigates time-of-check-time-of-use (TOCTOU) scenarios where the calling workflow may by triggered by a `push` event,
for example, but there may be subsequent pushes between then and the time the Job is able to checkout the source code.

```yaml
      - name: checkout-source-repo
        uses: actions/checkout@692973e3d937129bcbf40652eb9f2f61becf3332 #v4.1.7
        with:
          ref: ${{ github.sha }}
```

##### Variable Build Job Runner Label

Their is one caveat that we feel is acceptable: We allow the workflow input `build-runner-label` to determine the runner-label
for the  Build Job's `runs-on: ${{ inputs.build-runner-label }}`. Technically, this is an external parameter that could potentially
not be recored neither in source code nor in provenance, but the differences in runner operating systems
(`ubuntu-latest` vs `windows-latest`) are clear enough to be unambiguous. 

### Reusable Workflow Code

Here is the full code of the reusable workflow we describe, including instructions.

```yaml
...
```
