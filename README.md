# insights-operator

This cluster operator gathers anonymized system configuration and reports it to Red Hat Insights. It is a part of the standard OpenShift distribution. The data collected allows for debugging in the event of cluster failures or unanticipated errors.

## Reported data

* ClusterVersion
* ClusterOperator objects
* All non-secret global config (hostnames and URLs anonymized)

The list of all collected data with description, location in produced archive and link to Api and some examples is at [docs/gathered-data.md](docs/gathered-data.md)

The resulting data is packed in .tar.gz archive with folder structure indicated in the document. Example of such archive is at [docs/insights-archive-sample](docs/insights-archive-sample).

## Building

To build the operator, install Go 1.11 or above and run:

    make build

To test the operator against a remote cluster, run:

    bin/insights-operator start --config=config/local.yaml --kubeconfig=$KUBECONFIG

where `$KUBECONFIG` has sufficiently high permissions against the target cluster.

## Roadmap

The current operator only collects global configuration. Future revisions will expand the set of config that can be gathered as well as add on-demand capture.

## Contributing

Please make sure to run `make test-unit` to check all changes made in the source code.

### Sample IO archive
There is a sample IO archive maintained in this repo to use as a quick reference. (can be found at [docs/insights-archive-sample](https://github.com/openshift/insights-operator/tree/master/docs/insights-archive-sample))

To keep it up-to-date it is **required** to update this manually when developing a new data enhancement.

Make sure the `.json` files are in a humanly readable format in the sample archive.
By doing this its easier to review a data enhancement PR, and rule developers can easily check what data it collects. 

#### Generating a sample archive
Run the insights-operator on a test cluster.  (from `cluster-bot` or `Quicklab` or etc.) 

#### Formatting archive json files
This formats .json files from folder with extracted archive.
```
find . -type f -name '*.json' -print | while read line; do cat "$line" | jq > "$line.tmp" && mv "$line.tmp" "$line"; done
```


## Testing

Unit tests can be started by the following command:

    make test-unit

It is also possible to specify CLI options for Go test. For example, if you need to disable test results caching, use the following command:

    make test-unit TEST_OPTIONS=-count=1

## Issue Tracking

Insights Operator is part of Red Hat OpenShift Container Platform. For product-related issues, please
file a ticket [in Red Hat Bugzilla](https://bugzilla.redhat.com/enter_bug.cgi?product=OpenShift%20Container%20Platform&component=Insights%20Operator) for "Insights Operator" component.

## Generating the document with gathered data
The document docs/gathered-data contains list of collected data, the Api and some examples. The document is generated from package sources by looking for Gather... methods.
If for any GatherXXX method exists its method which returns example with name ExampleXXX, the generated example is added to document with the size in bytes.


To start generating the document run:
```
make gen-doc
```

## Accessing Prometheus metrics provided by Insights Operator

It is possible to read Prometheus metrics provided by Insights Operator. For example if the IO runs locally, the following command migth be used:

``
curl --cert k8s.crt --key k8s.key -k https://localhost:8443/metrics
``

## Accessing prometheus metrics with K8s Token
```
# Get token
oc whoami -t
# Read metrics from Pod
oc exec -it deployment/insights-operator -n openshift-insights -- curl -k -H "Authorization: Bearer 8NCUZTV3mvigpHxhdIKer6AyBLce14uehzg9b2R4dPY" 'https://localhost:8443/metrics'
```
Example of metrics exposed by Insights Operator can be found at [metrics.txt](docs/metrics.txt)

### Certificate and key needed to access Prometheus metrics

Certificate and key are required to access Prometheus metrics (instead 404 Forbidden is returned). It is possible to generate these two files from Kubernetes config file. Certificate is stored in `users/admin/client-cerfificate-data` and key in `users/admin/client-key-data`. Please note that these values are encoded by using Base64 encoding, so it is needed to decode them, for example by `base64 -d`.

There's a tool named `gen_cert_key.py` that can be used to automatically generate both files. It is stored in `tools` subdirectory.

#### Usage:

```
gen_cert_file.py kubeconfig.yaml
```

### Fetching metrics from Prometheus endpoint

```
sudo kubefwd svc -n openshift-monitoring -d openshift-monitoring.svc -l prometheus=k8s
curl --cert k8s.crt --key k8s.key  -k 'https://prometheus-k8s.openshift-monitoring.svc:9091/metrics'
```

### Debugging prometheus metrics without valid CA

Get a bearer token
```
oc sa get-token prometheus-k8s -n openshift-monitoring
```
Change in pkg/controller/operator.go after creating metricsGatherKubeConfig (about line 86)
```
metricsGatherKubeConfig.Insecure = true
metricsGatherKubeConfig.BearerToken = "paste your token here"
metricsGatherKubeConfig.CAFile = "" // by default it is "/var/run/secrets/kubernetes.io/serviceaccount/service-ca.crt"
metricsGatherKubeConfig.CAData = []byte{}
```

### Using the profiler

#### Starting IO with the profiler
IO starts a profiler if given the correct environment.
Set the OPENSHIFT_PROFILE env variable to "web".
```
export OPENSHIFT_PROFILE=web
```

#### Collect profiling data
After IO starts the profiling can be accessed at `http://localhost:6060`, you can use the `pprof` tool to connect to it.

Some profiling examples:
```
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30  // CPU profiling for 30 seconds
```
```
go tool pprof http://localhost:6060/debug/pprof/heap      // heap profiling
```
These commands will create a compressed file that can be visualized using a variety of tools, one of them is the `pprof` tool.

#### Analyzing profiling data
Starting a web ui at `localhost:8080` to visualize/analyze the profiling data:
```
go tool pprof -http=:8080 /path/to/profiling.out
```
For extra info:
[link](https://jvns.ca/blog/2017/09/24/profiling-go-with-pprof/)


### Creating a changelog
At `./cmd/changelog/main.go` there is a script that can generate a changelog for you.

#### Prerequisites
It uses both the local git and GitHub`s API to generate the file so:
- To get info from GitHub you will need to set the `GITHUB_TOKEN` envvar to a GitHub access-token.
- Make sure that you have a local, up-to-date copy of each release-branch that might be in the changelog.

#### Use
It can be used 2 ways:
1. Providing no command line arguments the script will update the current CHANGELOG.md with the latest changes according to the local git state. (IMPORTANT: It will only work with changelogs created with this script)
    - Example: `go run cmd/changelog/main.go`
2. Providing 2 command line arguments, `AFTER` and `UNTIL` dates the script will generate a new CHANGELOG.md within the provided time frame.
    - Example: `go run cmd/changelog/main.go 2021-01-10 2021-01-20`