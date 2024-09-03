Github has an official Action for creating signed SLSA attestations. Since it is an Action, you can use it in the same Workflow Job as your build, adjacent to your other build steps. This presents the risk that a compromised build process, perhaps by malicious upstream dependencies, can have access to the secret signing material used to create your SLSA attestations.

While the signed attestations are a great step towards securing the distribution of your built artifacts, Github's documentation suggests using Reusable Workflows to elevate your builds to SLSA Build Level 3.

This post discusses and demonstrates how to meet [slsa.dev's requirements](https://slsa.dev/spec/v1.0/levels#build-track) using Github's `actions/attest-build-artifacts` Action with Reusable Workflows.

## Hardening

The simplest initial step is to isolate signing into a separate Job. We must also make efforts to verify that the Jobs are done on github's own hardened runners, and not any "self-hosted" runners. We then go a bit further to make the reusbale workflow fit more build styles by abstracting your build commands into a new Action at a well-known location in your repo.

### L3: Isolate Secret Signing Material

[SLSA Build L3](https://slsa.dev/spec/v1.0/levels#build-l3-hardened-builds) requires that the the secret signing material
and other credentials (such as package registry credectials), be inaccesible by the build steps.

The `slsa3-build-rw.yml` will run your repo's named `slsa-build` action in a low-privilege `Build` Job, and then 
attest the artifacts in a separate `Sign` Job.
### L2: Trusted Build Platforms

[SLSA Build L2](https://slsa.dev/spec/v1.0/levels#build-l2-hosted-build-platform) requires that the build happens on a "trusted build platform". For Github Actions,
we interpret this to mean that artifacts are built using Github's hosted runners, and not a user's self-hosted runners.

Why not use self-hosted runners? Becuase They [can be maliciously modified](https://docs.github.com/en/enterprise-cloud@latest/actions/hosting-your-own-runners/managing-self-hosted-runners/about-self-hosted-runners#self-hosted-runner-security) by their host, but we can rely on Github to provide safe github-hosted runners. We use github-hosted runners to help protect the integrity of the build.

The `slsa3-build-rw.yml` will run your repo's named `slsa-build` action with a runner-label that you may supply as input.
But we found that if a user has a [self-hosted runner labeled "ubuntu-latest"](https://github.com/slsa-framework/slsa-github-generator/issues/1868#issuecomment-1979426130) or any other of Github's default runner label, then
Github Actions may still queue the Job on their self-hosted runners.

#### Verifying the Runner Environment

We use a combination of methods to to detect and prevent the use of self-hosted runners, but ultimately the verifying user must
supply the option `--deny-self-hosted-runners` to the Gtihub CLI when verifying.

The first method is effective for preventing you from accidentally using self-hosted runers. 
The second method is for detecting truly malicious self-hosted runners that try to conceal their status.

1. 
   The Github context `runner.environment` has values either `"github-hosted"` or `"self-hosted"`. But since this context is not available at
`jobs.<job_id>.if`, but at `jobs.<job_id>.steps.if`, we must force the Job to cancel by setting the Step's `run: exit 1`, based on the Step's `if: ${{ runner.environment != 'github-hosted' }}`.

    We are not yet sure if the decision to execute a Step happens locally on the runner or is more-directly controlled by Github Actions's remote orchestrator. In either case, `run: exit 1` has to execute on locally on the runner, so this is essentially a self-certification. 
    Speculating on the potential of the former case, we try to prevent the two Jobs from producing any usable outputs by adding the same condition to their core respective steps for building and signing artifacts.

    ```
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

    The second method 
We verify that the that the `Build` Job was run on a github-hosted runner by passing a mock signed attestation (not signing the user's target artifacts) from the `Build` Job to the `Sign` Job, which tries to verify the runner environment, 
by inspecting the OIDC claims by way of the [Fulcio signing certificate](https://github.com/sigstore/fulcio/blob/main/docs/oid-info.md#mapping-oidc-token-claims-to-fulcio-oids).

Finally, outside of the Workflow, the end user of the artifact and attestation will have to veirfy that a github-hosted runner was used by the `sign` Job:

```
$ gh attestation verify my-artifact --bundle attestation.jsonl --repo my-org/my-repo  --signer-workflow "my-signer-org/my-signer-repo/.github/workflows/my-signer-workflow.yml" --format json | jq --exit-status '.[].verificationResult.signature.certificate.runnerEnvironment | select(.=="github-hosted")'
"github-hosted"
```

### L1: Recorded Build Parameters

[SLSA Build L1](https://slsa.dev/spec/v1.0/levels#build-l1) requires that verifier must be able to form unambiguous expectations about the build process.

This can mean that our Reusable Workflow invokes pre-recorded commands, such as a Makefile, or `go build`.
For this reusable workfow, you have some more flexibility to fit more build styles: 

Instead, we can place all of your build commands into a new Action that this post instructs you to author into your repo
at `./github/actions/slsa-build`. This well-known location for the Action will be invoked by the reusable workflow
as part of the `Build` Job.
