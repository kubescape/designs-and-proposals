# Proposal: Secret Scanning of Container Images in node-agent

- **Status:** Draft – discussion
- **Author:** Khuswant Rajpurohit ([@khuswant18](https://github.com/khuswant18))
- **Date:** 2026-10-03 (updated 2026-10-27 after review)
- **Scope:** [`node-agent`](https://github.com/kubescape/node-agent), [`storage`](https://github.com/kubescape/storage), [`kubescape`](https://github.com/kubescape/kubescape) CLI, [`regolibrary`](https://github.com/kubescape/regolibrary), [`helm-charts`](https://github.com/kubescape/helm-charts); later [`kubevuln`](https://github.com/kubescape/kubevuln)
- **Related:**
  - [kubescape/node-agent#987: Scan container images for misplaced secrets](https://github.com/kubescape/node-agent/issues/987) (open)
  - Roadmap: "Scanning for misplaced secrets in container images (Monitoring phase)", [ROADMAP.md](https://github.com/kubescape/project-governance/blob/main/ROADMAP.md)
  - [kubescape/kubevuln#206: Secret detection capability](https://github.com/kubescape/kubevuln/pull/206) (closed)
  - [kubescape/kubevuln#296: Dive and Trufflehog integration](https://github.com/kubescape/kubevuln/pull/296) (closed)
- **Related work:** [SecurityException CRD design](https://github.com/kubescape/kubevuln/blob/c68de0013294ea8ac46738b906d38feb8820e7c1/docs/security-exception-design.md), [`evidence-of-finding`](./evidence-of-finding.md), [`storage-grpc-fastpath-auth`](./storage-grpc-fastpath-auth.md)

Code links are pinned to the commits they were checked against (node-agent `44c4cb29`, storage `81ccbff2`, k8s-interface `e9b55e6c`, kubescape `6fd7b78d`, regolibrary `e67b7d42`, helm-charts `201914f3`, kubevuln `c68de001`). Things marked **tested** were tried on a real cluster; [Appendix A](#appendix-a-how-the-claims-were-tested) lists how.

### Changes after the first review (on `5861bce`)

- Kubescape matches a workload to findings by the **image digest its Pods run**, not by image name and tag ([section 7](#7-showing-it-in-kubescape)).
- A missing or `initializing` result is shown as **needs review**, never as passed. The full reporting contract is in [section 7](#reporting-contract).
- A node with overlay layers can **take over** a `not-scanned` result. A node without them can never overwrite a scan ([section 2](#taking-over-a-not-scanned-result)).
- The saved object is limited in **bytes**, not only in number of findings, and a failed write has a defined recovery ([section 6](#staying-under-the-size-limit)).
- The kubevuln#296 comparison is now presented as our own technical comparison, not as the reason it was closed.

### Changes after the second review (on `15d3a27`)

- Each reservation has an owner token. Every final write, fallback and retry is a PUT carrying the `resourceVersion` **and** `uid` of that reservation, and its response is checked. A scanner that lost its reservation to a takeover can no longer overwrite the new result ([section 2](#only-the-owner-of-the-reservation-can-write-the-result)).
- Fencing was checked against the real storage code. A merge patch without `resourceVersion`, and a PUT without `uid` after the object was deleted and created again, both still overwrite. So patches are not used after the reservation, and the `uid` is required.
- Added what was promised in review: a stable `findingKey`, counting unique (rule, path) pairs, and an `includeHiddenFindings` control input.
- A rules-version rescan only happens when the stored rules version is older than the node's own, so an old node-agent can't replace a newer scan during an upgrade.
- New tests: a delayed old writer after takeover, a rules-version rescan while an old scan is running, and a lost write reply.

## Summary

- Container images often contain passwords, tokens and keys by mistake. Kubescape does not find them today.
- node-agent already has every image layer on disk when it builds the SBOM. We scan those same files for secrets at the same time.
- We also scan files that were deleted or overwritten in a later layer, because they are still inside the image.
- We save the findings once per image, in a new object in Kubescape storage. **We never save the secret itself.**
- Kubescape reads that object and shows a new control, "Secrets in container image", on the workloads that use the image.
- The SBOM does not change. If the secret scan fails, the SBOM is not affected.

## Motivation

### The problem

Take this Dockerfile:

```dockerfile
FROM alpine:3.20
# 1. token in the image config
ENV API_TOKEN=ghp_xxxxxxxx
# 2. deleted, but still in the earlier layer
RUN mkdir -p /app && echo "AWS_SECRET_ACCESS_KEY=xxxx" > /app/.env
RUN rm /app/.env
# 3. overwritten, but the old value is still in the earlier layer
RUN echo "db_password: secret" > /app/config.yaml
RUN echo 'db_password: ${DB_PASSWORD}' > /app/config.yaml
# 4. private key
COPY id_rsa /root/.ssh/id_rsa
```

Each `RUN` or `COPY` creates a new **layer**. Deleting a file only adds a marker in a new layer; the old layer still has the file. Anyone who can pull the image can read all four secrets.

**Tested.** On a cluster with Kubescape installed:

| What we checked | Result |
|---|---|
| SBOM for this image | Created normally in 3 seconds |
| `kubescape scan control C-0012` | **Passed.** None of the 4 secrets reported |
| Same token in the pod's `env:` instead | **Failed** C-0012 |

C-0012 only reads Kubernetes YAML (pod env and ConfigMaps). Nothing in Kubescape looks inside images.

This is common: a study of 337,171 images found secrets in **8.5%** of them ([Dahlmanns et al., 2023](https://arxiv.org/abs/2307.03958)).

### Prior work

| | [kubevuln#206](https://github.com/kubescape/kubevuln/pull/206) (2024) | [kubevuln#296](https://github.com/kubescape/kubevuln/pull/296) (2025) | This proposal |
|---|---|---|---|
| Where it ran | kubevuln | kubevuln, pulled the image again | node-agent, files already on disk |
| Engine | 26 regex rules (taken from Trivy) | TruffleHog binary (AGPL license) | Shared Go module, Apache-2.0 rules |
| Deleted/overwritten files | Missed | Found | Found |
| What it stored | Nothing; **logged part of each secret** | **Full secret values** in a new CRD | Findings only, **no values** |
| Effect on SBOM | A secret-scan error failed the SBOM | Separate | Separate |

kubevuln#296 was closed without a public reason, and the visible maintainer comment on it was positive. The rows above are our own technical comparison, not the documented reasons it was closed.

Review comments on #206: "I'd set `SkipBinaryFiles` to `true` in the first release. We need to make the scanning more efficient before we turn this on" (@slashben), and "this will be implemented in node-agent" (@matthyx).

From the community meeting on #987:

> Makes sense to have it at the same time as the SBOM generation (either node-agent, kubescape or kubevuln - depending on use case) because all layers are already opened. Question: where to store the findings? Is there a SBOS (bill of secrets) or similar existing? Kubescape parses the CRD and reports the compliance issue.

## Goals

1. Find secrets in every layer of every running image, including deleted and overwritten files.
2. Also check the image config: environment variables, build history and labels.
3. Save the result once per image per cluster, with no secret values anywhere: not in storage, logs or metrics.
4. Show the result as a Kubescape control on the workloads that use the image.
5. Keep false alarms low.
6. Keep the detector reusable by kubevuln and the CLI later.
7. Keep the cost bounded, and never show a half-finished or missing scan as "clean".

## Non-goals

For the first version:

- Looking inside archives (`.tar.gz`, `.jar`, `.whl`) and binary files.
- Scanning the node's own filesystem.
- Checking whether a found secret still works, or rotating it.
- Checking whether the running container actually reads the file (possible later, using `ContainerProfile`).
- Scanning images that are not running.

## How it works: simple view

```mermaid
flowchart LR
  A[Container starts] --> B[node-agent finds<br/>the image layers]
  B --> C[Scan files and<br/>image config for secrets]
  C --> D[(Save findings<br/>once per image)]
  D --> E[Kubescape shows<br/>a failed control]
```

1. A container starts on a node.
2. node-agent already finds the image's layers on disk to build the SBOM. We reuse that.
3. A new scanner looks for secrets in those files and in the image config.
4. The findings (not the secrets) are saved in Kubescape storage, once per image.
5. Kubescape reads them and marks the workloads that use the image as failed.

## How it works: complete architecture

Read from top to bottom. The numbers 1 to 6 match the steps below.

```mermaid
flowchart TD
  subgraph NA[node-agent, on every node]
    S1[Step 1: Container starts]
    S1b[Step 1: ContainerCallback<br/>existing]
    S1c[Step 1: Image layers + image config<br/>found once, shared]
    S2[Step 2: SbomManager + Syft<br/>existing, unchanged]
    S3a{Step 3: Overlay layers found?}
    S3b[Step 3: Skip, save result not-scanned<br/>a capable node can take over]
    S3c[Step 3: SecretScanManager<br/>new]
    S3d[Step 3: Reserve, only one node<br/>scans each image]
    subgraph MOD[Step 4: secretscan module, new]
      S4a[Walker: goes through every layer,<br/>marks deleted/overwritten files as hidden]
      S4b[Detector: keywords → regex → allowlist]
      S4c[Config scan: ENV, history, labels]
    end
  end
  subgraph ST[Kubescape storage]
    SB[(SBOMSyft<br/>existing)]
    S5[(Step 5: ImageSecretScan<br/>new, per image, no values)]
    S5b[Step 5: Cleanup when the image<br/>is no longer running]
  end
  subgraph KS[Kubescape]
    S6a[Step 6: SecurityException]
    S6b[Step 6: kubescape scan matches<br/>Pod image digests to findings]
    S6c[Step 6: New control C-03xx:<br/>workload failed]
  end
  S1 --> S1b --> S1c
  S1c --> S2 --> SB
  S1c --> S3a
  S3a -- no --> S3b --> S5
  S3a -- yes --> S3c --> S3d
  S3d --> S4a --> S4b --> S5
  S3d --> S4c --> S5
  S5 --> S5b
  S5 --> S6b
  S6a --> S6b --> S6c
```

### Step by step, with the example image

**Step 1. A container starts (existing code).**
node-agent gets an event for every new container. It finds the image's layers on disk and reads the image config from the container runtime. The SBOM code already does this, and we reuse the result.

**Step 2. The SBOM is built as today (no change).**
The secret scan runs next to the SBOM, not inside it. They have separate status and error handling.

**Step 3. Decide whether to scan, and make sure only one node does it (new).**
- If the node does not use overlay layers, node-agent can only see the live container filesystem. That includes mounted Kubernetes Secrets, which are not part of the image. So we skip and save the scan result `not-scanned`. Such a node only ever creates the object and never updates it, so it can't overwrite a real scan. If a node with overlay layers sees the image later, it takes over and scans it.
- Otherwise, the first node to see the image creates its result object with status `initializing`. Other nodes see that it already exists and skip. SBOMs already work this way. The exceptions to this rule are in [Taking over a `not-scanned` result](#taking-over-a-not-scanned-result).

**Step 4. Find the secrets (new).**
- The **walker** reads every layer, from top to bottom. When a higher layer deletes or replaces a file, the walker marks the lower copy as **hidden**. Hidden files still count, because anyone who pulls the image can read them.
- The **detector** checks each file in three passes: a fast keyword check, then a regex, then an allowlist that drops known false alarms (test files, docs, example keys).
- The **config scan** checks `ENV`, build history and labels, because a token in `ENV` is not a file.

For the example image, this finds all four secrets: the key (visible), the AWS key (hidden, deleted), the password (hidden, overwritten), and the token (from the config).

**Step 5. Save the result once per image (new).**
One object per image, with the same name as the SBOM. Each finding has a rule, path, line, layer, visible or hidden, a `fingerprint` for the exact occurrence and a `findingKey` that stays the same after a rebuild. **The secret value is never saved.** The scan result is always set: `complete`, `incomplete` or `not-scanned`. The object is kept under the storage size limit in bytes. If some findings don't fit, the summary still counts them and the result is marked `incomplete`. When the image is no longer running, storage cleanup deletes the object, as it does for SBOMs.

**Step 6. Kubescape reports it (new).**
`kubescape scan` finds the image digests that each workload's Pods are running and reads the result for each digest. It applies any SecurityException, and a new control marks the workloads that run an image with findings as failed. A workload passes only when every digest it runs has a complete, clean result. A missing or unfinished result shows as **needs review**.

## What we reuse from node-agent today

These are already in node-agent and were checked in the code:

| What | Where | Used for |
|---|---|---|
| Event for every new container | [`ContainerCallback`](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/sbom_manager.go#L297) | Trigger |
| List of the image's layer folders on disk (overlay "lowerdirs", top layer first) | [`getMountedVolumes`](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/sbom_manager.go#L254-L294) | Files to scan |
| Image config: `Env`, `History`, `Labels`, layer IDs | [`getImageStatus`](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/sbom_manager.go#L247), parsed in [`NewSource`](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/syftutil/source.go#L39) | Config scan |
| Which layer ID belongs to which folder | [`toLayers`](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/syftutil/source.go#L168) | Layer of each finding |
| "One node per image": create with `status=initializing`, others get `AlreadyExists` | [L437–L460](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/sbom_manager.go#L437-L460) | Reservation |
| Rescan when the tool version changes | [`shouldRetryAtCurrentVersion`](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/sbom_manager.go#L963) | Rescan when rules change |

What we **cannot** reuse: the SBOM's file view ([`NodeResolver.FilesByPath`](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/syftutil/resolver.go#L62-L79)) keeps only the top copy of each file, so deleted and overwritten files never show up. That's why the scanner has its own walker.

## Design details

### 1. The new manager in node-agent

`SecretScanManager` gets the same container events and the same layers and config as `SbomManager`. It copies these SBOM habits:

| Habit | How |
|---|---|
| Object name | Same as the SBOM (`ImageInfoToSlug`), so the two objects pair by name |
| One node per image | Create with `status=initializing`; skip on `AlreadyExists`, except in the [takeover cases](#taking-over-a-not-scanned-result) |
| Rescan | When the stored `kubescape.io/secret-rules-version` is older than the node's own, or when a capable node takes over a `not-scanned` or stale result. Only the current owner of the reservation can write the result ([details](#only-the-owner-of-the-reservation-can-write-the-result)) |
| Object status (`kubescape.io/status`) | `initializing` while the scan runs, `ready` when the result is written. We don't use `too-large`, because storage stops accepting updates to an object in that state ([comment](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/sbom_manager.go#L450-L457)) |

There are two separate fields:
- The **object status** (`kubescape.io/status`) says whether node-agent has finished writing the object. It works the same way as for SBOMs.
- The **scan result** (`spec.scan.status`, also copied to the `kubescape.io/secret-scan-status` annotation) says how much of the image was checked: `complete`, `incomplete` or `not-scanned`.

For example, an image on a node without overlay layers has object status `ready` and scan result `not-scanned`.

- It runs **inside node-agent** first. The SBOM scanner sidecar is off by default in the chart, so sidecar support comes later.
- It has **its own single worker**, separate from the SBOM work, with the CPU cap node-agent already uses for Syft.
- node-agent already runs as root with `SYS_ADMIN`, which it needs to read the overlay markers on deleted files. No new permissions on the node.

### 2. When we don't scan

| Situation | What we save | Why |
|---|---|---|
| No overlay layers on the node | `not-scanned`, reason `no-overlay` | node-agent falls back to the live filesystem ([L271–L276](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/sbommanager/v1/sbom_manager.go#L271-L276)), which includes mounted Kubernetes Secrets |
| Number of layer folders ≠ number of layer IDs | `incomplete`, reason `layer-mismatch` | We can't say which layer a file came from |
| Time or size budget used up | `incomplete`, reason `budget-exceeded`, with what we found so far | A half-done scan must never look clean |
| Findings don't fit in the object | `incomplete`, reason `findings-truncated`, `truncated: true` | See [Staying under the size limit](#staying-under-the-size-limit) |

#### Taking over a `not-scanned` result

A cluster can mix nodes with and without overlay layers. Suppose a node without overlay layers sees an image first and saves `not-scanned` / `no-overlay`. A node that can read the layers must be able to replace that result. Otherwise the image stays unscanned for as long as it runs.

What a node does depends on whether it has overlay layers and on the object that already exists:

| Node | Existing object | What the node does |
|---|---|---|
| Without overlay | None | Create it with object status `ready`, scan result `not-scanned`, reason `no-overlay` |
| Without overlay | Any | Nothing. It only creates objects and never updates them, so it can't overwrite a scan |
| With overlay | None | Create it with `initializing`, scan, write the result |
| With overlay | `ready`, `not-scanned`, reason `no-overlay` | **Take over:** set it to `initializing` with a conditional update, then scan |
| With overlay | `initializing`, started less than `staleReservationAfter` ago | Nothing. Another node is scanning |
| With overlay | `initializing`, started more than `staleReservationAfter` ago | **Take over.** The other node probably stopped or couldn't write its result |
| With overlay | `ready`, `complete` or `incomplete`, rules version **older** than this node's | **Take over** and rescan |
| With overlay | `ready`, `complete` or `incomplete`, same or newer rules version | Nothing |

A node never rescans because the stored rules version is *newer* than its own. During a rolling upgrade of node-agent, an old node would otherwise replace a new scan with an older one.

How a takeover works:

- A takeover is a PUT that carries the `resourceVersion` and `uid` the node just read. If two nodes try at the same time, storage accepts one and returns `Conflict` to the other, which then skips. Why it must be a PUT and not a patch is explained in [the next section](#only-the-owner-of-the-reservation-can-write-the-result).
- When a node sets an object to `initializing`, by creating it or by taking it over, it also sets `kubescape.io/secret-scan-started` to the current time. This is how other nodes tell that a reservation is stale. `staleReservationAfter` defaults to 15 minutes, three times the scan timeout.

#### Only the owner of the reservation can write the result

A takeover hands the image to a new scanner, but the old scanner may still be running. For example: scanner A pauses for more than 15 minutes, scanner B takes over and finishes, and then A resumes. If A could still write, it would replace B's result with older or less complete evidence. A rules-version rescan raises the same problem.

So every reservation has an owner, and only the owner can write:

1. **Owner token.** Each time a node puts an object into `initializing` (create, takeover, or rules-version rescan), it sets `kubescape.io/secret-scan-owner` to a new random token. It keeps three things from the object storage returned: the token, the `resourceVersion` and the `uid`.
2. **Fenced writes.** Every write after that is a full **update (HTTP PUT, the clientset's `Update`)** whose body carries the kept `resourceVersion` **and** `uid`. This covers the final result, the summary-only fallbacks, and every retry. A takeover changes the `resourceVersion`, so a write from the old owner gets `Conflict`.
3. **Check the response.** A write counts as done only if the object storage returns has this node's token and the status it wrote. Anything else is handled like a `Conflict`.
4. **After a `Conflict`, read the object again:**

| What the node reads | What it means | What the node does |
|---|---|---|
| Owner is someone else's token | It lost the reservation | Drop the result. No retry, no fallback write. Counted as `write.total{status="fenced"}` |
| Owner is its own token, status `ready` | Its earlier write worked, but the reply was lost | Done |
| Owner is its own token, status `initializing` | Something else changed the object, but the node still owns it | Keep the new `resourceVersion` and retry the write |
| Object is gone, or has a different `uid` | Cleanup deleted it (the image stopped running), and maybe another node created it again | Drop the result |

Why it has to be exactly this. Each point below was checked with a test against storage `81ccbff2`: the real `genericregistry.Store` and file/SQLite backend, using `SBOMSyft` as a stand-in for the new kind. For the patch cases, the test copies the apiserver's merge-patch step.

| Write | Result |
|---|---|
| PUT with an old `resourceVersion` | `Conflict`. Already covered upstream by `TestDifferential_UpdateResourceVersion` |
| Merge patch with no `resourceVersion` (what node-agent's [`PatchSBOMAnnotations`](https://github.com/kubescape/node-agent/blob/44c4cb2951d5de2ce09231920d9b21c61329eaca/pkg/storage/v1/storage.go#L108-L118) sends) | **Applied.** Not fenced |
| Merge patch with an old `resourceVersion` | `Conflict` |
| PUT with an old `resourceVersion` but **no `uid`**, after the object was deleted and created again | **Applied.** `resourceVersion` is a counter per object that restarts at 1, so an old value can match the new object |
| PUT with an old `resourceVersion` and the old `uid`, after delete and re-create | `Conflict` (the UID precondition fails) |
| Merge patch with an old `resourceVersion` and the old `uid`, after delete and re-create | Rejected, but as `Invalid` ("uid is immutable"), not `Conflict` |
| PUT with an old `resourceVersion` to an object whose status is `too-large` | Returns **success** but writes nothing. Storage skips every check for `too-large` objects ([storage.go](https://github.com/kubescape/storage/blob/81ccbff2841c84010f50aa02bacb765e6e17eac9/pkg/registry/file/storage.go#L1937-L1944), and the same in [singlewriter.go](https://github.com/kubescape/storage/blob/81ccbff2841c84010f50aa02bacb765e6e17eac9/pkg/registry/file/singlewriter.go#L1079)) |

So the rules are:

- Every write after the reservation is a PUT with `resourceVersion` and `uid`. **No merge or JSON patch, including for annotation-only changes.** The SBOM path uses a patch, but it doesn't have to stop old writers.
- A takeover is also a PUT with `resourceVersion` and `uid`. It is based on the metadata-only read, which returns the real `resourceVersion` (checked). That PUT clears `spec`, so the findings from the earlier scan go away until the new scan writes. That's fine: an object in `initializing` shows as needs review either way.
- A response that doesn't carry this node's token and status counts as not written. In practice only the `too-large` case causes this. Nothing in this design sets `too-large`, and storage sets it only for ContainerProfiles, but the check keeps it safe if that ever changes.
- The only writes that aren't fenced are the two creates, the no-overlay `not-scanned` and the first `initializing`, and those fail with `AlreadyExists` instead of overwriting. Storage cleanup can delete the object at any time, which the table above handles.

Old owners can never write after a takeover, so these cases are safe:

- **Clock skew between nodes:** it can only make a takeover happen early. That repeats some work but can't lose or overwrite a newer result.
- **A rules-version rescan while an old scan is still running:** the rescan is a takeover, so the old scan's write is fenced.
- Only `no-overlay` can be taken over. That reason depends on the node, not on the image, and a node with overlay layers never produces it. A result like `budget-exceeded` or `layer-mismatch` would usually come out the same on the next node, so taking it over would only repeat the scan.

Known limitation: on Docker-runtime clusters (Docker Desktop, cri-dockerd with Docker 29), node-agent gets containers without image information and builds no SBOM. The secret scan has the same limitation.

### 3. Reading the layers

On the node, each layer is a folder. When a higher layer deletes a file, containerd puts a **marker** with the same name in that higher layer:

| Marker | Meaning |
|---|---|
| A character device `0/0` with the file's name | This file was deleted |
| A folder with the attribute `trusted.overlay.opaque=y` | Everything in this folder from lower layers is hidden |
| Same with `user.overlay.*` | Same, on rootless nodes |
| `.wh.<name>` / `.wh..wh..opq` files | Same, in image tar files (used later by kubevuln and the CLI) |

**Tested** on containerd: the example image's layers on disk looked like this (top first; the test image used longer fake values than the Dockerfile above):

```text
layer 1  /root/.ssh/id_rsa       normal file          → visible
layer 2  /app/config.yaml        normal file          → visible (clean value)
layer 3  /app/config.yaml        normal file          → hidden (overwritten)
layer 4  /app/.env               char device 0/0      → marker: deleted
layer 5  /app/.env               normal file          → hidden (deleted)
layer 6  alpine base files
```

How the walker decides "visible" or "hidden":

```text
go through layers from top to bottom:
  for each file in the layer:
    if it is a deletion marker  → remember: this path is deleted
    if it is a normal file:
      visible = not seen in a higher layer
                and not deleted by a higher layer
                and not inside a folder hidden by a higher layer
      scan the file, record visible or hidden
```

Only normal files are scanned. Links are not followed, so the walker never leaves the layer folder.

### 4. Finding secrets

```text
file → skip if too big or binary → keyword check → regex → allowlist → finding
```

1. **Skip** files over 10 MiB and binary files (a zero byte in the first 8 KiB).
2. **Keyword check:** each rule has keywords (for example `ghp_`, `AKIA`, `PRIVATE KEY`). One fast pass checks all keywords at once. Only rules whose keyword appears go on to the regex step.
3. **Regex** for those rules only.
4. **Allowlist:** drop matches in known safe places (test folders, docs, Go and Python standard libraries) and known example values like `AKIAIOSFODNN7EXAMPLE`.

The keyword check matters. Measured with the #206 rules:

| Image | Without keyword check | With keyword check | Findings |
|---|---|---|---|
| node:22-slim | 12.0 s | 3.0 s | same (3) |
| postgres:16 | 22.6 s | 4.7 s | same (1) |
| golang:1.23-alpine | 34.9 s | 5.2 s | same (4) |

Reading the files takes under 0.6 s; nearly all the time is spent in regex. Every hit on these public images was a false alarm (test keys, docs) or intentional (Debian's `ssl-cert-snakeoil.key`). That's why the allowlist is part of the first version.

**Rules:** we propose using Trivy's built-in rules and allowlist as data (Apache-2.0, with credit). The #206 rules were already a subset of Trivy's. The noisy `generic-api-key` rule stays off; with gitleaks' defaults it caused 57 of 61 hits on one image.

The detector doesn't care where files come from. It takes a list of files with their path, layer and visible/hidden flag. node-agent gives it overlay folders. Later, kubevuln and the CLI can give it layers from a registry image without changing the detector.

```go
// Illustrative
type FileEntry struct {
    Path    string // path inside the image
    LayerID string // "sha256:…"
    Visible bool   // false = deleted or overwritten
    Open    func() (io.ReadCloser, error)
}
func Scan(ctx context.Context, files Walker, config ImageConfig, limits Options) (Result, error)
```

### 5. The image config

`ENV`, build history (`History[].created_by`) and labels go through the same rules. These findings have `source: config.env`, `config.history` or `config.label`, and no layer.

### 6. The saved result

**Why a new object type?** The maintainers asked whether a standard "bill of secrets" exists. We looked:

| Format | Fits? |
|---|---|
| CycloneDX 1.6 / 1.7 | Closest. It can describe a token or key found at a path and line, without the value. But it has no field for layer, visible/hidden, or whether the scan finished |
| SARIF | Made for one report per scan run, not one object per image |
| SPDX | Nothing for secrets |
| "Bill of secrets" | No standard with this name |

So we propose a new Kubescape storage object, similar to `VulnerabilityManifest`, and we document how it maps to CycloneDX so it can be exported later.

```yaml
apiVersion: spdx.softwarecomposition.kubescape.io/v1beta1
kind: ImageSecretScan                 # name to be decided (Q3)
metadata:
  name: docker.io-library-myapp-1.2-3f9c1a        # same name as the SBOM
  namespace: kubescape
  annotations:
    kubescape.io/image-id: docker.io/library/myapp@sha256:…   # needed by cleanup
    kubescape.io/image-digest: sha256:…                        # digest part only, matched to Pods (section 7)
    kubescape.io/image-tag: docker.io/library/myapp:1.2
    kubescape.io/status: ready
    kubescape.io/secret-rules-version: "1"
    kubescape.io/secret-scan-started: "2026-10-03T10:00:00Z"   # finds stale reservations (section 2)
    kubescape.io/secret-scan-owner: 7d0f…                      # random token of the reservation owner (section 2)
    # copy of the summary, because a normal list only returns metadata (see section 7)
    # counts are unique (rule, path) pairs, and include any findings left out by the size limit
    kubescape.io/secret-scan-status: complete
    kubescape.io/secrets-critical: "2"
    kubescape.io/secrets-high: "1"
    kubescape.io/secrets-visible-critical: "1"   # pairs with at least one copy visible at runtime
    kubescape.io/secrets-visible-high: "1"
    kubescape.io/secrets-hidden: "2"             # pairs with no copy visible at runtime
    # medium and low counts follow the same pattern
spec:
  scan:
    status: complete          # complete | incomplete | not-scanned, always set
    reason: ""                # e.g. no-overlay, budget-exceeded, findings-truncated
    filesScanned: 4120
    filesSkipped: 313
    findingsTotal: 2
    findingsStored: 2
    truncated: false          # true if findings were left out (count or byte limit)
  findings:
  - ruleID: private-key
    severity: Critical
    source: file
    path: /root/.ssh/id_rsa
    line: 1
    layer: sha256:37a687…
    visibleAtRuntime: true
    fingerprint: 3f9c…        # this exact occurrence: rule + path + layer + line
    findingKey: b71e…         # stable across rebuilds: rule + path
  - ruleID: aws-secret-access-key
    severity: Critical
    source: file
    path: /app/.env
    line: 1
    layer: sha256:c3a091…
    visibleAtRuntime: false   # deleted in a later layer
    fingerprint: 8a21…
    findingKey: 0c4d…
```

Storage details:

- **Size:** limited in bytes, not only by number of findings. See [Staying under the size limit](#staying-under-the-size-limit).
- **Cleanup:** same rule as SBOMs ([`deleteByImageId`](https://github.com/kubescape/storage/blob/81ccbff2841c84010f50aa02bacb765e6e17eac9/pkg/registry/file/cleanup.go#L407)). The `kubescape.io/image-id` annotation must always be set, or cleanup deletes the object.
- **One finding per (rule, path, layer, line).** A secret repeated in three layers gives three findings. Each layer is a separate copy that someone can extract, and the fix depends on which layer it's in.
- **Two IDs per finding, neither made from the secret:**
  - `fingerprint = SHA-256(ruleID + path + layer + line)` identifies one exact occurrence in one image. It changes when the layer or the line changes.
  - `findingKey = SHA-256(ruleID + path)` stays the same when the image is rebuilt. It is used for counting and for exceptions.
- **Counting unique (rule, path) pairs.** The summary counts and the control count each `findingKey` once, not once per layer or line, so a secret copied into three layers is one secret. A pair counts as **visible** if at least one of its copies is visible at runtime, otherwise as **hidden**. Findings from the image config count as visible. Trade-off: two different secrets of the same type in the same file count as one. `findingsTotal` and `findingsStored` count list entries (occurrences), not pairs.
- **One object per image ID.** The name comes from [`ImageInfoToSlug`](https://github.com/kubescape/k8s-interface/blob/e9b55e6c1199cee6ab1dea756e1fdc29b04d8515/names/slugs.go#L180-L196), which adds the last 6 characters of the image ID to the tag, so two digests of `app:stable` get two separate objects. Kubescape matches on the `kubescape.io/image-digest` annotation, not on the name.

#### Staying under the size limit

New object types get `maxObjectSize` 400,000 bytes by default ([config](https://github.com/kubescape/storage/blob/81ccbff2841c84010f50aa02bacb765e6e17eac9/pkg/config/config.go#L208)). Storage rejects any request whose body is larger with HTTP 413 ([QueueHandler](https://github.com/kubescape/storage/blob/81ccbff2841c84010f50aa02bacb765e6e17eac9/pkg/queuemanager/manager.go#L143-L149)). Limiting the number of findings doesn't keep us under that: 1,000 findings with 1 KiB paths add up to more than 1 MB from the paths alone.

So node-agent limits the object in bytes as well as by count:

1. Count every finding first. The summary (annotations and `spec.scan`) always shows the full counts, even when some findings are not stored.
2. Sort the findings by severity, highest first, then by path and layer, so the same image always keeps the same findings.
3. Build the object without findings, encoded the same way the client sends it to storage. That gives the base size.
4. Add findings in that order while the encoded size stays at or under `maxObjectBytes` and the number stays at or under `maxFindings`. `maxObjectBytes` defaults to 360,000, which keeps 40,000 of the 400,000 bytes free for request overhead. It must be set below the storage `maxObjectSize` for this kind.
5. If any finding was left out, set `truncated: true` and `findingsStored` lower than `findingsTotal`. A `complete` scan result becomes `incomplete` with reason `findings-truncated`.

The counts still include the findings that were left out, so the control still fails the workload.

If the final write fails:

| Failure | What node-agent does |
|---|---|
| `Conflict` | Read the object again and follow the [owner rules](#only-the-owner-of-the-reservation-can-write-the-result). If another node owns it now, stop without writing |
| 413 (our size estimate was wrong) | Write again without findings: summary only, `truncated: true`, `incomplete`, reason `write-too-large`. Still a fenced write |
| Any other error (timeout, storage restarting) | Retry 3 times with backoff, then write the summary-only object with reason `write-failed`. Every retry and the fallback are fenced writes |
| The summary-only write fails too (not because of `Conflict`) | Give up. The object stays `initializing`, and Kubescape shows **needs review**. After `staleReservationAfter`, any node with overlay layers (this one included) [takes it over](#taking-over-a-not-scanned-result) and scans again the next time it sees the image |

### 7. Showing it in Kubescape

The new control is **separate from C-0012**, because C-0012 also runs on plain YAML files where there are no images.

We tested how Kubescape can read objects from storage. What we found:

1. **A normal list returns only names, labels and annotations, not the findings.** The full object comes back only with the special option `resourceVersion=fullSpec` ([storage code](https://github.com/kubescape/storage/blob/81ccbff2841c84010f50aa02bacb765e6e17eac9/pkg/registry/file/storage.go#L1683)). Tested: 0 CVEs in a normal list, 653 with `fullSpec`.
2. **A Rego rule can already read storage objects, but only their annotations.** We wrote a test rule that joins a Deployment's image with an SBOM's `kubescape.io/image-tag` annotation. It failed `Deployment/nginx` with no Kubescape code change.
3. **Rules see Deployments, not their Pods.** On its own, a rule can only match by image name and tag. That isn't safe (see [Matching by digest](#matching-a-workload-to-its-images-by-digest)), so Kubescape has to add the Pods' image digests.
4. **In the cluster, it silently passes.** The in-cluster scanner skips the `kubescape` namespace ([`excludeNamespaces`](https://github.com/kubescape/helm-charts/blob/201914f32051a53bdcae2aa181b617b911844cb9/charts/kubescape-operator/values.yaml#L54)), and all storage objects live there. With that setting, the same test rule passed nginx without any error.
5. **The in-cluster scanner can't read storage objects at all** (no RBAC permission).
6. **The old image-data input `ImageVulnerabilities` is not used.** C-0083 and C-0085 depend on it and return "irrelevant" even when an image has critical CVEs.

So whichever option we pick, Kubescape needs a change. Two options:

| | Option A: Kubescape fetches the full object | Option B: rule uses the annotations |
|---|---|---|
| How | The CLI loads the objects with `fullSpec` and gives them to the rule | The rule reads the summary counts from the annotations |
| Shows file paths | Yes | No, only counts |
| Matches image by | Digest that the Pods run | Digest that the Pods run (same lookup) |
| Kubescape change | Bigger: new input type, loading full objects, digest lookup, RBAC, namespace fix | Smaller: digest lookup, namespace fix, RBAC |

**Proposal:** option B first, option A later when we want file paths in the report. Option B still matches by digest; it only means the rule reads annotations instead of the full object.

#### Matching a workload to its images by digest

Matching by name and tag is not safe. Two workloads can both use `app:stable` but run different images, during a rollout or because nodes pulled the tag at different times. If the rule uses the clean image's report, the workload running the leaky image passes. If it combines every report for the tag, the clean workload fails.

So Kubescape matches by the digest each Pod actually runs:

1. For each workload, Kubescape finds its running Pods through owner references (Pod → ReplicaSet → Deployment, Pod → Job → CronJob, Pod → StatefulSet, and so on). It reads `imageID` from `status.containerStatuses[]` and `status.initContainerStatuses[]`.
2. It keeps only the digest part (`sha256:…`). The Pod may report `docker.io/library/app@sha256:abc…` or just `sha256:abc…` (images without a registry digest, see Q2), so only the digest part is compared.
3. node-agent writes the same digest part to the `kubescape.io/image-digest` annotation.
4. The rule gets each workload's list of digests and looks up the result for each one.

This needs Pods and their owners in Kubescape's input, not only the Deployment. Q10 asks whether the lookup should happen in Go before the rules run, or in Rego. Either way the rule only reads annotations. It doesn't need the full object (option A).

#### Reporting contract

The result for **one digest** comes from the objects whose `kubescape.io/image-digest` matches it:

| Objects for this digest | Result for this digest |
|---|---|
| None | needs review |
| Object status `initializing` | needs review |
| `ready`, any scan result, findings at or above the threshold | failed |
| `ready`, scan `complete`, no findings at or above the threshold | clean |
| `ready`, scan `incomplete` or `not-scanned`, no findings at or above the threshold | needs review |

If more than one object has the same digest (the same image under two tags), they describe the same bytes. Take the first result that applies in this order: failed, then clean, then needs review. So one `complete` scan is enough to make the digest clean.

The result for **a workload** comes from all the digests of its running Pods:

| Situation | Workload result |
|---|---|
| Any digest is failed | **failed** |
| Every container in every running Pod has a digest, and every digest is clean | **passed** |
| Anything else: a Pod without an `imageID`, an `imageID` with no digest part, or any digest needing review | **needs review** |
| The workload has no running Pods | not checked (nothing is running) |

"Needs review" is never shown as passed. The CLI scanning YAML files has no Pods, so the control doesn't apply there.

The control: `C-03xx` (the latest is C-0318), "Secrets in container image", category `Secrets`. "Findings at or above the threshold" means unique (rule, path) pairs, counted as in [section 6](#6-the-saved-result). It has two inputs:

| Input | Default | Meaning |
|---|---|---|
| `minSeverity` | `High` | Lowest severity that fails the control |
| `includeHiddenFindings` | `true` | When `true`, hidden pairs count the same as visible ones: anyone who can pull the image can read them. When `false`, only pairs with a copy visible at runtime count, using the `kubescape.io/secrets-visible-*` annotations |

Results appear at the next configuration scan (daily by default) unless continuous scanning is on.

### 8. Exceptions

**First version, no change needed:** a SecurityException with a `posture` entry for the new control ID turns the control off for the matched workloads.

**Later:** exceptions for single findings, for example to accept Debian's snakeoil key:

```yaml
apiVersion: kubescape.io/v1beta1
kind: SecurityException
metadata: {name: accept-snakeoil, namespace: web}
spec:
  reason: Debian ssl-cert package, not used
  match:
    images: ["docker.io/library/nginx:*"]
  secrets:
  - ruleID: private-key
    path: /etc/ssl/private/ssl-cert-snakeoil.key
    # layer: sha256:…   optional, to accept only one copy
```

An entry matches on `ruleID` + `path`, which is the `findingKey`, so it keeps working after the image is rebuilt and the layer digests change. `layer` is optional and narrows it to one copy.

This needs:
- a new `secrets` field. **Tested:** today the cluster rejects it with `unknown field "spec.secrets"`.
- the CRD rule changed so `secrets` alone is allowed ([CRD](https://github.com/kubescape/helm-charts/blob/201914f32051a53bdcae2aa181b617b911844cb9/charts/kubescape-operator/crds/security-exception.crd.yaml#L25-L28)).
- `match.images`, which today applies only to vulnerabilities, extended to secrets.

Exceptions only work when `capabilities.riskAcceptance` is enabled (off by default).

### 9. Keeping secret values out of everything

| Place | Rule |
|---|---|
| Storage | No value, no masked value, no surrounding lines |
| Logs | Never log what matched. Errors include only the rule ID and path |
| Metrics | Labels are only status, reason and severity. No paths, no image names |
| Fingerprint and `findingKey` | `SHA-256(rule + path + layer + line)` and `SHA-256(rule + path)`. No secret goes in, so neither can be reversed to a value |

A test puts known fake secrets in an image, scans it, then searches storage, logs and metrics for them. If any appear, the test fails.

## Configuration

```yaml
# Helm values
capabilities:
  imageSecretScan: disable        # needs nodeSbomGeneration: enable (the default)

nodeAgent:
  config:
    secretScan:
      maxFileSize: 10Mi
      maxBytesPerImage: 512Mi
      timeout: 5m
      maxFindings: 1000
      maxObjectBytes: 360000      # must be below storage maxObjectSize for this kind
      staleReservationAfter: 15m
      skipBaseLayers: false
      extraAllowPaths: []

storage:
  kindQueues:
    imagesecretscans: {queueLength: 50, workerCount: 1, maxObjectSize: 400000}
```

RBAC changes:

| Who | Needs |
|---|---|
| node-agent | create, get, update on the new object (no `patch`; see section 2) |
| kubescape | get, list on the new object |

## Metrics

| Metric | Labels |
|---|---|
| `node_agent.secretscan.scan.total` | status |
| `node_agent.secretscan.scan.duration` | status |
| `node_agent.secretscan.files.skipped.total` | reason (binary, size, allowlist) |
| `node_agent.secretscan.findings.total` | severity |
| `node_agent.secretscan.takeover.total` | reason (no-overlay, stale) |
| `node_agent.secretscan.write.total` | status (ok, truncated, too-large, failed, fenced) |

## Test plan

| Test | What it checks |
|---|---|
| Walker unit tests | Folders laid out like containerd layers: deleted files, hidden folders, overwritten files, links |
| Detector unit tests | Each rule with a real and a fake sample; keyword check gives the same results as no keyword check; allowlist; line numbers |
| Config scan tests | `ENV`, history, labels |
| False-alarm check | A fixed set of public images must keep the same number of findings |
| No-secret-values test | Fake secrets must not appear in storage, logs or metrics |
| Component test (kind) | Deploy the example image; check the saved object and the control result |
| Same tag, different digests | Two Deployments use `app:stable`; one runs digest A (with secrets), the other digest B (clean). A fails, B passes. A Deployment in the middle of a rollout, with Pods on both A and B, fails |
| Missing or unfinished report | No object, an `initializing` object, or a Pod without an `imageID`: the workload needs review and does not pass. An image with only a `sha256:` image ID still matches |
| Mixed nodes, both orders | Node without overlay first, then node with overlay: the result ends `complete`. Node with overlay first, then node without: the result stays `complete`. Two nodes with overlay take over at the same time: only one scans |
| Stale reservation | An `initializing` object older than `staleReservationAfter` is taken over; a newer one is not |
| Delayed old writer | Scanner A reserves and pauses, B takes over and writes `complete`, then A resumes. A's final write and its summary-only fallback both get `Conflict`, A sees another owner and stops, and B's result is unchanged |
| Rules-version rescan with an old scan running | A scans with rules v1; B rescans with v2 and finishes; A's write is fenced. A node with rules v1 that sees a v2 result does not rescan |
| Old writer after delete and re-create | A reserves (`resourceVersion` 1), cleanup deletes the object, C creates it again (`resourceVersion` 1 again). A's write gets `Conflict` because of the `uid`, A sees another `uid` and stops, and C's reservation is unchanged |
| Storage fencing (storage PR, through the generated clientset node-agent uses) | `Update` with an old `resourceVersion` gives `Conflict`. `Update` with the old `uid` after delete and re-create gives `Conflict`. The metadata-only `Get` returns the same `resourceVersion` as a full `Get`. A merge patch without `resourceVersion` is applied (this documents why node-agent must not use patches) |
| Counting and IDs | The same secret in three layers gives three findings, one `findingKey` and a count of 1. A pair with one visible and one hidden copy counts as visible. `findingKey` stays the same when the image is rebuilt with a changed layer; `fingerprint` changes |
| `includeHiddenFindings` | An image with only hidden High findings fails with `true` and passes with `false` (if nothing else needs review) |
| No patches after the reservation | node-agent unit test: the fake storage client fails the test on any `Patch` call to this kind |
| Response check | The fake storage returns success with another owner's token (like the `too-large` case). node-agent treats it as not written and does not report success |
| Lost reply | The final write succeeds but the reply is lost; the retry gets `Conflict`, the node reads its own token with status `ready`, and stops without writing again |
| Long paths | 1,000 findings with paths of about 1 KiB: the stored findings fit under `maxObjectBytes`, `truncated: true`, scan `incomplete`, storage accepts the write, and the annotation counts match `findingsTotal` |
| Byte boundary | An object exactly at `maxObjectBytes` keeps every finding; one byte more leaves out the last finding |
| Write failures | A 413 leads to a summary-only write. Storage errors are retried, then a summary-only write. If storage stays down, the object stays `initializing` and is taken over later |

## Rollout

The feature is off by default until the control and the false-alarm check are stable.

| # | Repo | What | Needs |
|---|---|---|---|
| 1 | designs-and-proposals | This proposal | — |
| 2 | node-agent | `secretscan` module: walker, detector, config scan, rules, tests | — |
| 3 | storage | New object type, cleanup, fencing tests (PUT with `resourceVersion` + `uid`) | 1 |
| 4 | node-agent | `SecretScanManager`, takeover, byte limit, config, metrics, component test | 2, 3 |
| 5 | helm-charts | Capability, config, RBAC, storage size | 3, 4 |
| 6 | kubescape | Pod digest lookup, reporting contract, namespace fix, RBAC (option B) | 3 |
| 7 | regolibrary | New control and rule | 6 |
| 8 | node-agent | Sidecar support, base-layer skip, per-layer cache | 4 |
| 9 | kubevuln, helm-charts, kubescape | `secrets` in SecurityException | 7 |
| 10 | docs | User docs | 7 |

Each PR stays under about 1,000 lines (not counting generated code and test data) and includes its tests.

Later: check whether the container really reads the file (using `ContainerProfile`); use the module in kubevuln and `kubescape scan image`; export to CycloneDX and SARIF.

## Alternatives considered

| Alternative | Why not |
|---|---|
| Scan in kubevuln | Needs a second image pull and registry credentials. node-agent already has the files. |
| Put findings in the SBOM | Different update cycle and error handling. Syft's own maintainers say secrets are out of scope for SBOMs ([syft#1954](https://github.com/anchore/syft/issues/1954)). |
| Use the SBOM's file view | Only sees the top copy of each file, so it misses deleted and overwritten ones. |
| TruffleHog | AGPL license. |
| gitleaks library | Adds a WebAssembly regex dependency; its default rules are tuned for git repos and are noisy on images. |
| Extend C-0012 | C-0012 also runs on plain YAML files, where there are no images. |
| Scan the live filesystem on non-overlay nodes | It includes mounted Kubernetes Secrets and would report them as image leaks. |

## Open questions

| # | Question | Our proposal |
|---|---|---|
| **Q1** | How should Kubescape read the findings: option B (annotations, small change) or option A (full object, file paths)? Is `ImageVulnerabilities` still used anywhere? | B first, A later. Both match by the digest the Pods run |
| **Q2** | For images without a registry digest (built locally, `kind load`, air-gapped), the Pod shows `sha256:abc…` but node-agent saves `docker.io/library/app@sha256:abc…`. Storage cleanup compares the two as text, finds no match, and deletes the SBOM and CVE report while the Pod is still running. **Tested.** This is a bug today, separate from this proposal. Is it already known? Should our new object wait for the fix? | I'll open a separate issue for it. Our object uses the same cleanup, so it gets the fix too, and doesn't need to wait. Until then, these images show as needs review, not passed |
| **Q3** | Is a new object type in Kubescape storage OK? What name? | Yes, `ImageSecretScan` |
| **Q4** | Where should the shared detector live? | Start in `node-agent/pkg/secretscan`; move to its own repo when kubevuln uses it |
| **Q5** | Can we use Trivy's rules as data (Apache-2.0)? | Yes, a chosen subset; `generic-api-key` off |
| **Q6** | Add `secrets` to SecurityException later, behind `riskAcceptance`? | Yes, in a later PR |
| **Q7** | Scan all layers, or skip the base image by default? | All layers with the allowlist; skipping is an option |
| **Q8** | Limits and default: separate worker, 5 min / 512 MiB / 1,000 findings per image, off by default? | Yes |
| **Q9** | Which control ID should we use? | The next one after C-0318 |
| **Q10** | Where should Kubescape match Pod digests to workloads: in Go before the rules run, or in Rego with Pods and their owners as rule input? | In Go, so every rule that needs image results can reuse it |
| **Q11** | Does storage reject an old writer, so that only one node can take over a result? | For PUT with `resourceVersion` + `uid`, yes. Checked at storage `81ccbff2` with the real backend (section 2). A merge patch without `resourceVersion`, or a PUT without `uid` after delete and re-create, is **not** rejected, so node-agent uses neither. The storage PR keeps a test for this through the clientset |

## Appendix A: how the claims were tested

Setup: Colima, k3s v1.35.0, containerd 2.3.1; Kubescape chart 1.40.4, node-agent v0.3.219, CLI 4.0.12.

| Claim | How | Result |
|---|---|---|
| Image secrets are not reported | Built the example image, deployed it, ran `kubescape scan control C-0012` | SBOM in 3 s; C-0012 passed; same token in pod `env` failed |
| How layers look on disk | Read the layer folders from `/proc/<pid>/mountinfo`; looked at names and sizes only | Deleted file shown as `0/0` character device; overwritten file in two layers |
| Normal list hides findings | `kubectl get --raw …/vulnerabilitymanifests` with and without `?resourceVersion=fullSpec` | 0 vs 653 CVEs |
| Rego can read annotations | Local test framework with a rule over Deployment + SBOMSyft | `Deployment/nginx` failed |
| In-cluster setting hides it | Same rule with the in-cluster namespace exclusions | `Deployment/nginx` passed |
| `ImageVulnerabilities` unused | `kubescape scan control C-0083,C-0085`; searched the code | "irrelevant", 0 resources; nothing fills it |
| Permissions | `kubectl auth can-i` as each service account | kubescape can't read storage objects; no one can read SecurityExceptions while `riskAcceptance` is off |
| `secrets` field rejected | `kubectl apply --dry-run=server` | `unknown field "spec.secrets"` |
| Cleanup deletes running images' SBOMs | Storage logs; Pod `imageID` vs saved annotation | SBOM and CVE report deleted at the 6-hour cleanup while the Pod was running |
