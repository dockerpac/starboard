# [DRAFT] Caching Scan Results by Image Reference

## TL;DR;

To find vulnerabilities in container images Starboard creates asynchronous
Kubernetes (K8s) Jobs. Even though running a vulnerability scanner as a K8s
Job is expensive, Starboard does not reuse scan results in any way.
For example, if a workload refers to the image that has already been scanned,
Starboard will go ahead and create another (similar) K8s Job.

To some extent, the problem of wasteful and long-running K8s Jobs can be
mitigated by using Starboard with Trivy in the [ClientServer] mode instead of
the default [Standalone] mode. In this case a configured Trivy server will cache
results of scanning image layers. However, there is still unnecessary overhead
for managing K8s Jobs and communication between Trivy client and server.
(The only real difference is that some Jobs may complete faster for already scanned
images.)

To solve the above-mentioned problems, we could cache scan results by image
reference. For example, a CRD based implementation can store scan results as
instances of ClusterVulnerabilityReport object named after a hash of the
repo digest. An alternative implementation may cache vulnerability reports
in an AWS S3 bucket or a similar key-value store.

## Example

With the proposed cluster-scoped (or global) cache, Starboard can check if the
image with the specified reference has already been scanned. If yes, it will
just read the corresponding ClusterVulnerabilityReport, copy its payload, and
finally create an instance of a namespaced VulnerabilityReport.

Let's consider two `nginx:1.16` Deployments in two different namespaces `foo`
and `bar`. In the current implementation Starboard will spin up two K8s Jobs to
run a scanner and eventually create two VulnerabilityReports in `foo` and `bar`
namespaces respectively.

In a cluster where Starboard is installed for the first time, when we scan the
`nginx` Deployment in the `foo` namespace there's obviously no
ClusterVulnerabilityReport for `nginx:1.16`. Therefore, Starboard will spin up
a K8s Job and wait for its completion. On completion, it will create a
cluster-scoped ClusterVulnerabilityReport named after the hash of `nginx:1.16`.
It will also create a namespaced VulnerabilityReport named after the current
revision of the `nginx` Deployment.

> **NOTE** Because a repo digest is not a valid name for a K8s API object, we
> may, for example, calculate a (safe) hash of the repo digest and use is as
> name instead.

```console
$ kubectl get clustervulnerabilityreports
No resources found
```

```console
$ starboard scan vulnerabilityreports deploy/nginx -n foo -v 3
I1008 19:58:19.355462   62385 scanner.go:72] Getting Pod template for workload: {Deployment nginx foo}
I1008 19:58:19.358802   62385 scanner.go:89] Checking if images were already scanned
I1008 19:58:19.360411   62385 scanner.go:95] Cached scan reports: 0
I1008 19:58:19.360421   62385 scanner.go:101] Scanning with options: {ScanJobTimeout:0s DeleteScanJob:true}
I1008 19:58:19.365155   62385 runner.go:79] Running task and waiting forever
I1008 19:58:19.365190   62385 runnable_job.go:74] Creating job "starboard/scan-vulnerabilityreport-cbf8c9b99"
I1008 19:58:19.376902   62385 reflector.go:219] Starting reflector *v1.Event (30m0s) from pkg/mod/k8s.io/client-go@v0.22.2/tools/cache/reflector.go:167
I1008 19:58:19.376920   62385 reflector.go:255] Listing and watching *v1.Event from pkg/mod/k8s.io/client-go@v0.22.2/tools/cache/reflector.go:167
I1008 19:58:19.376902   62385 reflector.go:219] Starting reflector *v1.Job (30m0s) from pkg/mod/k8s.io/client-go@v0.22.2/tools/cache/reflector.go:167
I1008 19:58:19.376937   62385 reflector.go:255] Listing and watching *v1.Job from pkg/mod/k8s.io/client-go@v0.22.2/tools/cache/reflector.go:167
I1008 19:58:19.386049   62385 runnable_job.go:130] Event: Created pod: scan-vulnerabilityreport-cbf8c9b99-4nzkb (SuccessfulCreate)
I1008 19:58:51.243554   62385 runnable_job.go:130] Event: Job completed (Completed)
I1008 19:58:51.247251   62385 runnable_job.go:109] Stopping runnable job on task completion with status: Complete
I1008 19:58:51.247273   62385 runner.go:83] Stopping runner on task completion with error: <nil>
I1008 19:58:51.247278   62385 scanner.go:130] Scan job completed: starboard/scan-vulnerabilityreport-cbf8c9b99
I1008 19:58:51.247297   62385 scanner.go:262] Getting logs for nginx container in job: starboard/scan-vulnerabilityreport-cbf8c9b99
I1008 19:58:51.674449   62385 scanner.go:123] Deleting scan job: starboard/scan-vulnerabilityreport-cbf8c9b99
```

Now, if we scan the `nginx` Deployment in the `bar` namespace, Starboard will
see that there's already a ClusterVulnerabilityReport (`84bcb5cd46`) for the
same image reference `nginx:1.16` and will skip creation of a K8s Job. It will
just read and copy the report as VulnerabilityReport object to the `bar`
namespace.

```console
$ kubectl get clustervulnerabilityreports -o wide
NAME         REPOSITORY      TAG    DIGEST   SCANNER   AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN
84bcb5cd46   library/nginx   1.16            Trivy     17s   21         50     33       104   0
```

```console
$ starboard scan vulnerabilityreports deploy/nginx -n bar -v 3
I1008 19:59:23.891718   62478 scanner.go:72] Getting Pod template for workload: {Deployment nginx bar}
I1008 19:59:23.895310   62478 scanner.go:89] Checking if image nginx:1.16 was already scanned
I1008 19:59:23.903058   62478 scanner.go:95] Cache hit
I1008 19:59:23.903078   62478 scanner.go:97] Copying ClusterVulnerabilityReport to VulnerabilityReport
```

As you can see, Starboard eventually created two VulnerabilityReports by spinning
up only one K8s Job.

```console
$ kubectl get vulnerabilityreports -A
NAMESPACE   NAME                                REPOSITORY      TAG    SCANNER   AGE
bar         replicaset-nginx-6d4cf56db6-nginx   library/nginx   1.16   Trivy     5m38s
foo         replicaset-nginx-6d4cf56db6-nginx   library/nginx   1.16   Trivy     6m10s
```

## Summary

* This solution might be the first step towards more efficient vulnerability scanning.
* It's backward compatible and can be implemented as an experimental feature behind
  a gate.
* Both Starboard CLI and Starboard Operator can read and leverage ClusterVulnerabilityReports.

## TODO

* Specify the life-cycle of ClusterVulnerabilityReport objects, e.g. how they are invalidated.

[Standalone]: https://aquasecurity.github.io/starboard/v0.12.0/integrations/vulnerability-scanners/trivy/#standalone
[ClientServer]: https://aquasecurity.github.io/starboard/v0.12.0/integrations/vulnerability-scanners/trivy/#clientserver
