---
status: proposed
title: Verified Remote Resources
creation-date: '2021-10-11'
last-updated: '2021-12-03'
authors:
- '@squee1945'
---

# TEP-0091: Verified Remote Resources

<!--
**Note:** When your TEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary", and "Motivation" sections.
  These should be easy if you've preflighted the idea of the TEP with the
  appropriate Working Group.
- [ ] **Create a PR for this TEP.**
  Assign it to people in the SIG that are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the TEP clarified and merged quickly.  The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a TEP is merged does not mean it is complete or approved.  Any TEP
marked as a `proposed` is a working document and subject to change.  You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing TEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused.  If you disagree with what is already in a document, open a new PR
with suggested changes.

If there are new details that belong in the TEP, edit the TEP.  Once a
feature has become "implemented", major changes should get new TEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/teps/NNNN-TEP-template/README.md).

-->

<!--
This is the title of your TEP.  Keep it short, simple, and descriptive.  A good
title can help communicate what the TEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a TEP and for
highlighting any additional information provided beyond the standard TEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
  - [Use Cases (optional)](#use-cases-optional)
- [Requirements](#requirements)
- [Proposal](#proposal)
  - [Notes/Caveats (optional)](#notescaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
  - [User Experience (optional)](#user-experience-optional)
  - [Performance (optional)](#performance-optional)
- [Design Details](#design-details)
- [Test Plan](#test-plan)
- [Design Evaluation](#design-evaluation)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
- [Upgrade &amp; Migration Strategy (optional)](#upgrade--migration-strategy-optional)
- [Implementation Pull request(s)](#implementation-pull-request-s)
- [References (optional)](#references-optional)
<!-- /toc -->


## Summary

<!--
This section is incredibly important for producing high quality user-focused
documentation such as release notes or a development roadmap.  It should be
possible to collect this information before implementation begins in order to
avoid requiring implementors to split their attention between writing release
notes and implementing the feature itself.

A good summary is probably at least a paragraph in length.

Both in this section and below, follow the guidelines of the [documentation
style guide]. In particular, wrap lines to a reasonable length, to make it
easier for reviewers to cite specific portions, and to minimize diff churn on
updates.

[documentation style guide]: https://github.com/kubernetes/community/blob/master/contributors/guide/style-guide.md
-->

The proposed features advance Secure Software Supply Chain goals and allow
users of Tekton and Tekton Chains to implement more secure builds.

| This TEP is written based on TEP-0060 Remote Resource Resolution. |

- Provide an optional mechanism for Remote Resources to be verified.
- Provide an optional mechanism to fail the build if a Remote Resource cannot be
  verified.
- Adjust Chains to ignore outputs from Task Runs that explicitly fail
  verification.
- Adjust Chains to optionally ignore outputs from Task Runs with no verification
  requirement.


## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this TEP.  Describe why the change is important and the benefits to users.  The
motivation section can optionally provide links to [experience reports][] to
demonstrate the interest in a TEP within the wider Tekton community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

Tekton Chains is built to make attestations based on Task Run parameters and
results. For the results in particular, how do we know that some activity
actually occurred, for example, how do we know that an image was pushed to a
repository?

Remote Resource verification allows us to place some level of trust in the outputs
from a Task Run so that we can be confident in the build results and the
attestation claims that are made.


### Goals

<!--
List the specific goals of the TEP.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

- Provide solid building blocks to begin to establish formal chains of trust
  within the build pipeline itself.
- Leveraging the above, provide mechanisms to establish a verifiable corpus
  of community-provided Tasks and Task Bundles.
- Create an optional Tekton configuration for Remote Resource verification based
  on Sigstore Cosign (https://github.com/sigstore/cosign).
- Create an optional Tekton Chains configuration to skip attestation creation
  based on information from Remote Resources that could not be verified.


### Non-Goals

<!--
What is out of scope for this TEP?  Listing non-goals helps to focus discussion
and make progress.
-->

- Specification of a verification mechansim for arbitrary Tasks.
- Specification of a verification mechanism for the images within a Task
  Bundle.


### Use Cases (optional)

<!--
Describe the concrete improvement specific groups of users will see if the
Motivations in this doc result in a fix or feature.

Consider both the user's role (are they a Task author? Catalog Task user?
Cluster Admin? etc...) and experience (what workflows or actions are enhanced
if this problem is solved?).
-->

**Kaniko builds.** A Kaniko task exists in the Tekton Catalog
(https://github.com/tektoncd/catalog/blob/main/task/kaniko/0.5/kaniko.yaml).
This task emits
output that allows Chains to make attestations. The task uses the
`gcr.io/kaniko-project/executor` to perform the build and push the resulting
image. But how can I be sure this Task is not compromised
(e.g., a malicious image was pushed to the repository)
and is actually doing
what I expect? At a minimum, I would like to be able to verify it, for example,
using Sigstore's Cosign, to know that it was produced by the expected
third-party.

**Buildpack builds.** Similar to Kaniko, a Buildpacks task exists in the Tekton
Catalog with output that indicates the image that was built
(https://github.com/tektoncd/catalog/blob/main/task/buildpacks-phases/0.2/buildpacks-phases.yaml).
The motivation is
the same here: how can I trust the Task without some sort of verification?

**Really, any third-party task.** If we're going to build an ecosystem of tasks
in the Tekton Catalog, we need to think about how to ensure these tasks are
safe. Task verification can provide building blocks.

**First-party tasks.** In the spirit of zero-trust, I'd like to sign my own
Tasks and verify them on each use. This would be a best practice for
enterprises, for example.


## Requirements

<!--
Describe constraints on the solution that must be met. Examples might include
performance characteristics that must be met, specific edge cases that must
be handled, or user scenarios that will be affected and must be accomodated.
-->

- There is a process by which a Task can be marked as trusted meaning the Task
  has been explicitly signed off by an authenticated third party.
- Verification can ensure the contents of the Task has not been modified since
  being marked as trusted.
- It is not possible to modify the marked Task in such a way that it will still
  pass verification.
- Verification can be performed by Tekton Pipelines; Tekton Chains can see
  whether or not the verification occurred and if it was successful.
- A Pipeline Author can indicate that a Task must be verified in order
  for Chains to create a provenance attestation.
- A Pipeline Author can configure a build to fail if a Task cannot be verified.
- Ideally, the Tekton Catalog can formally support Tekton Bundles, and the
  Bundle registry will make it clear how to obtain the public key for
  community-provided, trusted bundles.


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

Sigstore Cosign (https://github.com/sigstore/cosign) has mechanisms to securely
sign a given OCI image. This provides
a solid footing to leverage Cosign to verify image-based Remote Resources.
Future extensions can incorporate other types of Remote Resources, like git
repository based Remote Resources.

This proposal will introduce new optional configuration to Tekton to indicate
that a Remote Resource verification must occur. The Pipeline Author specifying the
task will also have to provide the public key of the signature used in the
verification. The public key itself is explicitly obtained outside of Tekton,
for example, the Remote Resource author may publish their public key somewhere.

If a Remote Resource fails verification, the corresponding TaskRun will be
annotated as such. The Pipeline Author will have optional configuration to stop
and fail the build on verification failure.

**IMPORTANT:** A Task refers to other images. For the initial delivery of this
TEP, all static image references within a Task fetched as a Remote Resource
must be referenced by digest
(i.e., `...@sha256:abcdef`) in order for the verification to succeed. In later
iterations, this can be extended such that tag-referenced images can
_themselves_ be verified for the overall Task to pass verification. However,
this will be left out of the scope of this TEP.

Tekton Chains will be updated to skip processing results from any Task that
fails verification *by default*. Note that this does not change existing
behavior because no existing Remote Resource is subject to the verification proposed
here, and Remote Resource verification is optional and opt-in by the Pipeline
Author.

The Pipeline Author will be able to provide optional configuration to Tekton
Chains such that Chains will *only* consider results from Tasks that were
verifiable (and, following the previous rule, were successfully verified).


### Notes/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

This proposal is explicitly focussed on OCI image baesed Remote Resources
because they work well with Cosign. This does not preclude
extending the verification mechanism in the future to encompass an arbitrary
Task, nor does it preclude additional verification mechanisms other than
Cosign.


### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate. Think broadly.
For example, consider both security and how this will impact the larger
kubernetes ecosystem.

How will security be reviewed and by whom?

How will UX be reviewed and by whom?

Consider including folks that also work outside the WGs or subproject.
-->

When verifying a Remote Resource that identifies a Task, the Task still has
references to external, unverified resources - namely, 
the builder images. These builder images may have been compromised and the
Task would still be considered verified.

Some possible mitigations include the following:

1. (Within scope of this proposal) For a Task to be considered verified, all
   builder image references within it must be by-digest. The digest references
   would form a low-effort way for the Task Author to signal that they have
   verified or otherwise trust the builder images they are referencing.
   For example, they may have manually verified the builder images using Cosign
   or some other technique.
1. (Outside the scope of this proposal) Configuration is introduced into Tekton
   to facilitate verification of the builder images themselves. Configuration
   would also be required to cover off various combinations of bundle
   verification and image verification - for example, do I allow verified
   Task Bundles if the Task within the Bundle does not have verified images?


### User Experience (optional)

<!--
Consideration about the user experience. Depending on the area of change,
users may be task and pipeline editors, they may trigger task and pipeline
runs or they may be responsible for monitoring the execution of runs,
via CLI, dashboard or a monitoring system.

Consider including folks that also work on CLI and dashboard.
-->

**Task Author publishing a Task.**
A Task Author may want their task and its
outputs to be trusted so that provenance attestations can be confidently made.
The Task Author can publish their Task as a Task Bundle and sign the bundle
OCI image with the Cosign tool. The Task Author would also publish their
public key, for example, in the Tekton Catalog that lists their Task Bundle.

**Platform Operator providing source fetch and image push integration.**
A Platform Operator may wish to integrate Tekton tightly with their other
product offerings and they want to confidently make provenance attestations.
This operator could publish their *own* Tasks that fetch source and push
built artifacts as Task Bundles and sign them. When they construct a Pipeline
on behalf of their user, they could indicate that their own Task Bundles be 
verified with their public key and configure Chains to only consider outputs
from verified Tasks when making attestations.

**Pipeline Author composing a pipeline from third-party Tasks.**
A great advantage of Tekton is the ability to leverage an ecosystem of
third-party contributors creating Tasks. But those Tasks use builder images
(external binaries) and the Pipeline Author wants to trust their outputs.
The Pipeline Author could utilize only verifiable Task Bundles and fail the
build completely on verification failure, limiting the blast radius of a
possibly compromised component.


### Performance (optional)

<!--
Consideration about performance.
What impact does this change have on the start-up time and execution time
of task and pipeline runs? What impact does it have on the resource footprint
of Tekton controllers as well as task and pipeline runs?

Consider which use cases are impacted by this change and what are their
performance requirements.
-->

Task verification using Cosign requires fetching from a possibly external
OCI repository (the same repository as the builder image), so when a Remote
Resource is configured to be verified, some overhead is incurred on the order
of approximately 5 seconds per Remote Resource. Remote Resources that are not
configured to be verified will have no change in performance.

Tekton Chains will only need to make simple decisions over only locally
available data so there will be no change to performance.


## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.

If it's helpful to include workflow diagrams or any other related images,
add them under "/teps/images/". It's upto the TEP author to choose the name
of the file, but general guidance is to include at least TEP number in the
file name, for example, "/teps/images/NNNN-workflow.jpg".
-->

TaskRun's `taskRef.resource` will be updated with the following **optional**
configuration `verification`:

```yaml

apiVersion: tekton.dev/v1beta1
kind: TaskRun

spec:
  taskRef:
    resolver: bundle
    resource:
      image_url: gcr.io/kaniko-project/kaniko-task
      name: kaniko-build-task
      verification:
        signer: [cosign]
        key: [some-public-key]
        onError: [ stopAndFail | continue ]
```

`signer: cosign`. (Required) Additional verification providers can be added in
the future. Initially, this TEP will support only `cosign`.

`key: [some-public-key]`. (Required) Direct input to Cosign. Cosign supports
many types of inputs including plain text and various KMS systems.

`onError: [ stopAndFail | continue ]`. (Optional, default `stopAndFail`)
`stopAndFail` will cause the build to fail if image verification fails, in the
spirit of secure by default. `continue` will allow the build to continue even
if verification fails.

TEP-0060 introduces a new ResourceRequest reconsiler that will actually pull
the remote image and expand it to a Task. This is where the verification
logic will be introduced because (1) the image digest is required for 
signature validation and (2) in the initial delivery of the feature proposed
here, each builder image in the Task definition must be by referenced
by digest for the entire Task to be considered verified.

If the Remote Resource is configured for verification via Cosign (the only option
on the initial implementation of this TEP), the code will be adjusted to
use the Cosign libraries to verify the signature on the Remote Resource image
using the configuration-provided public key. Note that this TEP leaves room
for alternate signing mechanisms in the future; Cosign is proposed here in the
interest of minimizing scope.

**IMPORTANT:** For the initial delivery of this TEP, all static image
references in the Task must be referenced by digest (i.e., `...@sha256:abcdef`)
in order for the Task to be considered for verification.

If the Remote Resource image fails verification, the TaskRun will be annotated
with `verification-failed`
(**TODO:** please help provide the _actual_ full annotation that would be 
appropriate).

If `stopAndFail` is configured, the build will fail on image verification.

**Future design idea.** As a future development, a verification section could
be introduced to the `config-defaults` ConfigMap to allow Remote Resources to be
verifiable by default. The verification configuration could be based on 
image prefix to reduce configuration (longest image name would match first).
For example:

```
  default-remote-resource-verification: |
    bundle:
    - prefix: gcr.io/kaniko-project/
      signer: cosign
      key: [ some-public-key ]
    - prefix: gcr.io/my-enterprise/
      signer: cosign
      key: gcpkms://...
    - prefix: gcr.io/my-enterprise/specific-image
      signer: cosign
      key: [ some-specific-public-key ]
```

## Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).
-->

## Design Evaluation
<!--
How does this proposal affect the reusability, simplicity, flexibility 
and conformance of Tekton, as described in [design principles](https://github.com/tektoncd/community/blob/master/design-principles.md)
-->

## Drawbacks

<!--
Why should this TEP _not_ be implemented?
-->

N/A

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->


## Infrastructure Needed (optional)

<!--
Use this section if you need things from the project/SIG.  Examples include a
new subproject, repos requested, github details.  Listing these here allows a
SIG to get the process for these resources started right away.
-->

## Upgrade & Migration Strategy (optional)

<!--
Use this section to detail wether this feature needs an upgrade or
migration strategy. This is especially useful when we modify a
behavior or add a feature that may replace and deprecate a current one.
-->

## Implementation Pull request(s)

<!--
Once the TEP is ready to be marked as implemented, list down all the Github
Pull-request(s) merged.
Note: This section is exclusively for merged pull requests, for this TEP.
It will be a quick reference for those looking for implementation of this TEP.
-->

## References (optional)

<!--
Use this section to add links to GitHub issues, other TEPs, design docs in Tekton
shared drive, examples, etc. This is useful to refer back to any other related links
to get more details.
-->