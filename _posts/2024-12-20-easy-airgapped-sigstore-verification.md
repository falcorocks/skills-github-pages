# Introducing TrustRoot Assembler: Simplifying Airgapped Sigstore Verification

## Cluster Image Policies & Custom Sigstore Instances

Sigstore allows us to sign container images using the "keyless" method. The Sigstore policy controller enables us to enforce admission control of images in a Kubernetes cluster based on the validity of these keyless signatures. For an image signature to be valid, it must be:
* Integer: The image was not tampered with, and the signature matches it.
* Authentic: The signature was produced by a specific identity (e.g., a particular GitHub Workflow from a specific repository).

The `ClusterImagePolicy` Custom Resource allows us to define an admission policy that evaluates signature integrity (automatically) and authenticity (by verifying that the OIDC issuer in the signature certificate matches the issuer in `spec.keyless[x].identities[y].issuer`). Additionally, it allows us to specify which signer identities are authorized (since anyone can sign with Sigstore!) by ensuring that the signature certificate identity matches `spec.keyless[x].identities[y].subject` or `spec.keyless[x].identities[y].subjectRegExp`. By default, the controller assumes that a signature was produced using the Public Good instance of Sigstore (the one we interact with when using `cosign sign ...`).

```yaml
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
    name: image-is-signed-by-github-actions-public-sigstore
spec:
    images:
    - glob: "**"
    authorities:
    - keyless:
            # Signed by the public Fulcio certificate authority
            url: https://fulcio.sigstore.dev
            identities:
            - issuer: https://token.actions.githubusercontent.com
                subjectRegExp: "https://github.com/falcorocks/.*/.github/workflows/.*@refs/heads/main"
        ctlog:
            # Signature stored in the public Rekor transparency log
            url: https://rekor.sigstore.dev
```

A field `spec.keyless[x].trustRootRef` can be used to inform the policy controller that the signature was produced using another Sigstore instance.

```yaml
apiVersion: policy.sigstore.dev/v1alpha1
kind: ClusterImagePolicy
metadata:
    name: image-is-signed-by-github-actions-my-sigstore
spec:
    images:
    - glob: "**"
    authorities:
    - keyless:
            # Signed by my private Fulcio certificate authority
            url: https://fulcio.my-sigstore.dev
            # Sigstore relies on my private Trust Root
            trustRootRef: my-sigstore
            identities:
            - issuer: https://token.actions.githubusercontent.com
                subjectRegExp:  "https://github.com/falcorocks/.*/.github/workflows/.*@refs/heads/main"
        ctlog:
            # Signature stored in my private Rekor transparency log
            url: https://rekor.my-sigstore.dev
```

When the default Sigstore is used, the policy controller relies on the TrustRoot embedded in its source code. However, if the `ClusterImagePolicy` specifies a `trustRootRef: my-sigstore`, the policy controller will need to check for a `TrustRoot` Custom Resource named `my-sigstore` in the cluster. But what exactly is a `TrustRoot`, and why does it matter?

## TUF TrustRoots

Sigstore follows a zero-trust philosophy. How does `cosign` know it is communicating with the correct instance of `Fulcio`? How do we ensure we are looking for a signature in the correct `Rekor` instance? To establish trust between these components, we need a PKI Root CA. The Update Framework (TUF) allows us to create a central root of trust for these components to rely on when making trust decisions. A TUF TrustRoot contains, among other things, the identities of the Sigstore components (Fulcio, Rekor, ct_log), a timestamp, and the public keys of the TrustRoot "managers." This information is then signed by the private keys of the managers in a signing ceremony. There is much more to TUF and TrustRoot, so you should check [todo add link].

In practice, the easiest way to use TUF is to set up a TUF repository with TUF-ci, which is what Sigstore maintainers do. Cosign is shipped with an embedded TrustRoot, so it does not need to reach the TUF repository to get it. However, the repository is eventually rotated, and the Sigstore components must regularly check for updates by querying the online repository.

The same applies to the policy controller. TrustRoots must be updated, and the process must be online. This process can be fully automated in clusters that allow internet connectivity to the TrustRoot remote. But what about airgapped clusters? In that case, the process must be semi-automated: first, we need to assemble an "offline" TrustRoot custom resource by reaching the mirror, then we deploy that custom resource in the airgapped cluster, where verification can happen fully offline.

The Sigstore documentation explains how to set up a `repository` TrustRoot, but there are no clearly reproducible examples. Inspired by prezha/TrustRoot, I've created falcorocks/trustroot-assembler to easily assemble `repository` TrustRoots for any TUF instance.