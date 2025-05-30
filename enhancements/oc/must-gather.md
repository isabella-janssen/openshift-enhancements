---
title: must-gather
authors:
- "@deads2k"
reviewers:
- "@derekwaynecarr"
- "@soltysh"
- "@mfojtik"
approvers:
- "@derekwaynecarr"
creation-date: 2019-09-09
last-updated: 2024-01-10
status: implemented
see-also:
replaces:
superseded-by:
---

# Must-Gather

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

To debug something broken in the cluster, it is important to have a single command for an unskilled customer to run that
gathers all the information we may need to solve the problem. If you're familiar with `sosreport`, the idea is to have
that, but focused on a kuberentes cluster instead of a host. We need to avoid the versioning skew complexity and the
scaling problems inherent in a shared repo with input from multiple products.

## Motivation

Software breaks and you aren't smart enough to know exactly what you need to debug the problem ahead of time. You figure
that out *after* you've debugged the problem. This tool is about the first shotgun gathering so you only have to ask the
customer once.

It must be simple. You're gathering because your software is buggy and hard to use. The more complex your gathering software, the more likely
it is that the gathering software fails too. Simplify your gathering software by only using a matching version of the gathering tool.
This simplifies code and test matrices so that your tool always works. For instance, OpenShift payloads include the exact
level of gathering used to debug that payload.

Own your destiny. You own shipping your own software. If you can be trusted to ship your software, you can be trusted
to ship your gathering tool to match it, don't let your gathering be gated by another product. It may seems easier to start,
but ultimately you'll end up constrained by different motivations, styles, and cadences. If you can ship one image, you can
ship a second one.

### Goals

1. Gathering should exactly match a single version of the product it is inspecting.
2. Different products should be responsible for gathering for their own components.
3. Introspection for clusteroperators should be largely automatic, covering a broad range of generic use-cases.
4. A single, low-arg client command for users.
5. In a failing cluster, gathering should be maximized to collect everything it can even when part of it fails.
6. CEE should own the gather script itself, since they are the first consumers.

### Non-Goals

## Proposal

`must-gather` for openshift is a combination of three tools:

1. A client-side `inspect` command that works like a super-get. It has semantic understanding of some resources and traverses
   links to get interesting information beyond the current. Pods in namespaces and logs for those pods, for instance.
   Currently, this is `openshift-must-gather inspect`, but we are porting this to `oc adm` as experimental in 4.3. We may
   change and extend arguments over time, but the intent of the command will remain.
2. The openshift-must-gather image, produced from https://github.com/openshift/must-gather. The entry point is a
   [/gather bash script](https://github.com/openshift/must-gather/blob/master/collection-scripts/gather) owned by CEE
   (not the developers) that describes what to gather. It is tightly coupled to the OpenShift payload
   and only contains logic to gather information from that payload. We have e2e tests that make sure this functions.
3. `oc adm must-gather --image` which is a client-side tool that runs any must-gather compatible image by creating a pod,
   running the `/usr/bin/gather` binary, and then rsyncing the `/must-gather` and includes the logs of the pod. To 
   avoid overwhelming the cluster by passing numerous `--image` flags or an `--all-images` flag (covered in next point),
   we are capping this to spin up four concurrent pods. If required, it could be exposed via a flag in future.
4. `oc adm must-gather --all-images` allows the user to perform the same task as `oc adm must-gather --image`, but 
   instead of specifying multiple must-gather compatible images using multiple `--image` flags, it finds such images 
   from the cluster and runs `/usr/bin/gather`. The way it finds such images is by looking up the Operators 
   (ClusterServiceVersions/CSVs) and [ClusterOperators]
   (https://github.com/openshift/enhancements/blob/master/dev-guide/operators.md#what-is-an-openshift-clusteroperator)
   with the annotation `operators.openshift.io/must-gather-image` on the cluster where it's running. Among the 
   various flags provided by `oc adm must-gather`, only `--dest-dir`, `--node-selector` and `--timeout` can be used 
   in conjunction with the `--all-images` flag.
5. `oc adm must-gather` allows users to limit the collected logs via `--since` and `--since-time` flags. `--since`
    accepts a duration (eg. 2h) and `--since-time` accepts an RFC3339 compatible date time

### `inspect`

See the [inspect enhancement](inspect.md) for details on the `inspect` command.

### must-gather Images

To provide your own must-gather image, it must....

1. Must have a zero-arg, executable file at `/usr/bin/gather` that does your default gathering
2. Must produce data to be copied back at `/must-gather`. The data must not contain any sensitive data. We don't string PII information, only secret information.
3. Must produce a text `/must-gather/version` that indicates the product (first line) and the version (second line, `major.minor.micro-qualifier`),
   so that programmatic analysis can be developed.
4. Should honor the user-provided values for `--since` and `--since-time`, which are passed to plugins via
   environment variables named `MUST_GATHER_SINCE` and `MUST_GATHER_SINCE_TIME`, respectively.

### local fall-back

If the `oc adm must-gather` tool's pod cannot be scheduled or run on the cluster, the `oc adm must-gather` tool will, after a timeout, fall-back to running `oc adm inspect clusteroperators` locally.

### User Stories

#### Story 1

#### Story 2

### Implementation Details/Notes/Constraints

During the implementation of `--since` and `--since-time`, `--until` and
`--until-time` were also considered.
Supporting `until` flags considerably increase the complexity of the task, so more
research is required before pursuing it.
The `--rotated-pod-logs` in `oc adm inspect` did not initially support `--since`
and `--since-time`. This was changed to take into considerations the time
constraint provided by the new flags by comparing them against the date in the
log file name. See https://github.com/openshift/oc/pull/1653 for implementation
details.

### Risks and Mitigations

What are the risks of this proposal and how do we mitigate? Think broadly. For
example, consider both security and how this will impact the larger OKD
ecosystem.

How will security be reviewed and by whom? How will UX be reviewed and by whom?

Consider including folks that also work outside your immediate sub-project.

When using either multiple `--images` flag or the `--all-images` flag, only four concurrent pods are started to 
prevent overwhelming the cluster. Also, when using the `--all-images` flag, one can't pass arguments other than 
`--dest-dir`, `--node-selector`, `--source-dir` and `--timeout`.

## Design Details

This is subject to change, but today we do this by running the must-gather image in an init container and then we have
a container that sleeps forever. We download the result and then delete the namespace to cleanup.

### Output Format

The output of a must-gather image is up the component producing the image, but this is how the openshift/must-gather is
currently organized.

```text
├── audit_logs
│   ├── kube-apiserver
│   │   ├── zipped audit files from each master here
│   ├── openshift-apiserver
│   │   ├── zipped audit files from each master here
├── host_service_logs
│   └── masters
│       ├── crio_service.log
│       └── kubelet_service.log
└── <inspect cmd output>
...
```

### Test Plan

There is an e2e test that makes sure the command always exits successfully and that certain apsects of the content
are always present.

### Graduation Criteria

### Upgrade / Downgrade Strategy

The image is included in the payload, but has no content running in a cluster to upgrade.

### Version Skew Strategy

The `oc` command must skew +/- one like normal commands.

## Implementation History

## Drawbacks

## Alternatives (Not Implemented)

## Infrastructure Needed
