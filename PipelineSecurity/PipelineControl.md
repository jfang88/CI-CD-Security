<img src="https://r2cdn.perplexity.ai/pplx-full-logo-primary-dark%402x.png" style="height:64px;margin-right:32px"/>

# Would you like this turned into a concrete control matrix mapped to preventive, detective, and responsive controls for a GitLab-style on-prem pipeline?

Assume that we are primarily on premise and can control the download/updates to be stored on local self hosted repositories or mirrors.  Add in appropriate approval.
Add in delay (if appropriate) and processes to continuously validate what we have.

If possible, describe processes to validate / audit what is actually running in production.

Yes — below is a concrete GitLab-style on-prem control matrix for a pipeline that uses internal mirrors, Nexus, controlled approvals, deliberate delay where it adds value, and continuous validation from intake through production.
The design assumption is that runners execute untrusted code, protected environments can block deployments pending approvals, and self-managed runners need isolation, segmentation, and ongoing review of both platform configuration and job behavior.

## Control matrix

| Stage | Preventive controls | Detective controls | Responsive controls |
| :-- | :-- | :-- | :-- |
| External intake into local mirrors | Only a central ingestion service can download upstream actions, scanner binaries, container images, and databases; everything lands first in a quarantined Nexus repository, not directly in a runnable repo. | Log source URL, checksum, signer, maintainer, retrieval time, and diff from the previous approved version; alert on mutable tags, changed checksums, or unexpected publisher changes. | Freeze promotion of the item, block downstream use, and trigger review plus retroactive impact analysis of any pipelines that consumed the prior version. |
| Internal approval and promotion | Require two-person approval for high-risk components such as CI actions, runner images, security scanners, base images, and deployment tooling; approve by immutable SHA or digest only. | Record who approved, what evidence was reviewed, and which exact digest or commit was promoted into the approved repo. | Revoke approval, unlist the version from approved repositories, and force pipelines back to the last known good version. |
| Build and scan jobs | Use protected runners for protected branches and tags, keep runners ephemeral where possible, avoid shell executor for mixed-trust workloads, avoid privileged containers, and do not use reusable shared workspaces for untrusted jobs. [^1] | Monitor runner egress, filesystem access to credential paths, container privilege use, and attempts to reuse cached repos or local images in unsafe ways. | Destroy the runner after suspicious activity, rotate all job-visible credentials, and rebuild the runner image rather than cleaning in place. |
| Release and deployment | Deploy only from protected environments with deployment approvals, restricted deployers, and manual execution after approvals are granted. [^2] | Review blocked deployments, approval history, and environment deployment records to confirm the right approvers, timing, and artifact version. | Reject or cancel blocked or suspicious deployments, reopen validation, and require a fresh promotion decision. |
| Production runtime | Admit only approved images or packages whose digest matches the promoted artifact and whose metadata matches the release record stored in GitLab and Nexus. | Continuously compare running digests, package versions, configs, and signer metadata against the approved release manifest; alert on drift. | Quarantine the workload, roll back to the previous approved artifact, and open an incident to determine how the drift entered production. |

## Approval and delay

Use approval at two separate points, because they solve different risks: first at external intake into the local mirror, and again before deployment to a protected production environment.
In GitLab, protected environments can require approvals before deployment, deployments remain blocked until approvals are granted, and after approval the deployment job still must be run manually, which makes them suitable for production release gates.

Delay is appropriate for high-risk upstream updates, especially scanner wrappers, runner images, and pipeline components, because a short soak period helps catch compromised or silently altered upstream content before it reaches production use.
A practical model is: low-risk signature or database updates may have short automated quarantine plus validation, while pipeline-executing components require a longer hold window, review of upstream changes, integrity verification, and successful execution in a non-production validation lane before promotion.

## Continuous validation

Continuously validate the approved mirror, not just the pipeline, because self-managed CI risk comes from both the runner platform and the code being executed on it.
GitLab’s runner security guidance explicitly recommends a systematic approach to the full stack and ongoing rigorous reviews of platform configuration and use, which fits a recurring validation program rather than one-time hardening.

Use a recurring control loop for every approved dependency or tool:

- Re-verify digest, signature, and upstream provenance on a schedule, and flag any previously approved item whose upstream tag now points somewhere else or whose metadata changes unexpectedly.
- Re-scan approved artifacts and runner images on a cadence, because new intelligence can turn yesterday’s approved item into today’s risk.
- Re-test execution in an isolated validation runner pool that has the same proxy, mirrors, and policy settings as production runners, so you detect hidden network calls or new privilege assumptions before rollout.
- Re-certify approvals after major upstream changes, maintainer transfers, repo ownership changes, or unusual release behavior, even if the version number looks normal.


## Production verification

To validate what is actually running in production, make the promoted artifact digest or package checksum the primary identity of the release, then store that identity in the change record, GitLab deployment record, Nexus promotion record, and the runtime manifest used for deployment.
That lets you audit production by comparing four things: what GitLab says was approved, what Nexus says was promoted, what the deployment system says it applied, and what hosts or clusters are actually running.

A useful production audit process is:

- Export the approved production manifest for each release, including image digests, package versions, Helm chart version, config bundle hash, SBOM reference, approvers, and deployment ticket.
- Query production hosts or Kubernetes clusters for actual running image digests, package inventories, mounted configs, and startup command lines, then compare them to the approved manifest and fail the audit on any drift.
- Reconcile GitLab audit events and deployment histories with runtime state, because GitLab provides audit events for user actions and environment deployment approval history that can show who approved or changed deployment behavior.[^3]
- Run scheduled “attested state” checks daily and after every deployment window, and treat any untracked digest, unsigned image, or manually changed package as a production incident.


## Operating process

A strong operating model for your environment is a five-lane process: ingest, quarantine, approve, promote, verify.
Ingest pulls upstream content into a quarantined internal mirror; quarantine runs integrity checks, scanning, metadata capture, and sandbox validation; approve requires named reviewers and evidence; promote copies only immutable approved versions into the production mirror; verify confirms production is still running only those approved versions.

For GitLab administration, also separate who can change pipelines from who can approve deployments and who can administer runners, because developers can influence code executed on runners and protected runners are intended to reduce exposure of sensitive jobs.[^1]
That separation, combined with protected environments, protected runners, network segmentation, and mirror-based dependency control, gives you both prevention against supply-chain compromise and a reliable audit trail of what was approved, what was deployed, and what is truly running.

Would you like this next as a RACI-style operating procedure plus a sample `.gitlab-ci.yml` control pattern for intake, approval, deploy, and post-deploy verification?
<span style="display:none">[^10][^11][^12][^13][^14][^15][^4][^5][^6][^7][^8][^9]</span>

<div align="center">⁂</div>

[^1]: https://docs.gitlab.com/ci/runners/configure_runners/

[^2]: https://docs.gitlab.com/ci/environments/deployment_approvals/

[^3]: https://docs.gitlab.com/administration/compliance/audit_event_reports/

[^4]: https://docs.gitlab.com/runner/security/

[^5]: https://docs.gitlab.co.jp/ee/ci/runners/configure_runners.html

[^6]: https://gitlab.cn/docs/14.0/ee/ci/runners/configure_runners.html

[^7]: https://sels.tecnico.ulisboa.pt/gitlab/help/ci/runners/configure_runners.md

[^8]: https://oneuptime.com/blog/post/2025-12-21-gitlab-deployment-approvals/view

[^9]: https://gitlab.cn/docs/en/ee/administration/audit_event_reports.html

[^10]: https://gitlab.fullon.co.jp/gitlab/help/ci/runners/configure_runners.md

[^11]: https://software.rcc.uchicago.edu/git/help/ci/runners/README.md

[^12]: https://repository.prace-ri.eu/git/help/ci/environments/deployment_approvals.md

[^13]: https://docs.gitlab.com/user/compliance/audit_events/

[^14]: https://gitlab-docs.creationline.com/ee/ci/runners/configure_runners.html

[^15]: https://genboree.org/gitlab/help/ci/environments/deployment_approvals.md

