## Crane: handling CRD v1beta1 removal

In OCP 4.9, the v1beta1 CRD API will be removed. The Crane maintainers considered multiple options for handling this API removal.  

### Option 1: Maintain Current and Legacy Versions of Crane (selected for Crane 1.6)

_See `crd-v1beta1-removal-dual-stream` for details_

 - OCP 4.6+ uses Crane 1.6+ with v1 Mig/Velero CRDs
 - OCP 4.5- and 3.7+ use Crane 1.5 with v1beta1 Velero CRDs

 We will be taking this approach in the Crane 1.6 timeframe to ensure that we will have a working version of Crane shipped in time for OCP 4.9 to avoid blocking user upgrades.


### Option 2: Implement Remote Dependency Install for Crane (selected for Crane 1.7)

_See `crd-v1beta1-removal-remote-install` for details_

 - OCP 4.6+ uses Crane 1.6+ with v1 Mig/Velero CRDs
 - OCP 4.5- and 3.7+ get dependencies remotely installed with v1beta1 Velero CRDs

This approach should reduce the matrix of versions we have to ship bugfixes, CVE patches, etc to, reducing overall work in the long term.

Given that this approach represents a greater up-front time investment, we will strongly reconsider this path in the Crane 1.7 timeframe to reduce long term complexity in the dev/QE/rel-eng version matrix.


### Option 3: Generate v1beta1 and v1 CRDs, make mig-controller read both (not selected)

_This option is not documented as heavily, but was considered and dismissed_

 - OCP 4.6+ uses Crane 1.6+ with v1 Mig/Velero CRDs
 - OCP 4.5- and 3.7+ use Crane 1.6+ with v1beta1 Mig/Velero CRDs

 This approach keeps the ability to run mig-controller on OCP 3.x, but potentially causes serious problems in the future since we would be trying to maintain a k8s controller compatible across an unreasonably wide set of API versions as time goes on. Option 1 and 2 are seen as less risky.