# E2E Testing Guide

This guide explains how Tetragon's end-to-end (E2E) tests work and how to write new tests.

## Overview

Tetragon's E2E tests use the [Kubernetes E2E framework](https://pkg.go.dev/sigs.k8s.io/e2e-framework) with a custom runner system that:
- Creates temporary Kubernetes clusters (kind)
- Installs Tetragon via Helm
- Runs workloads and validates Tetragon events
- Exports test artifacts for debugging

## Quick Start

### Running E2E Tests

**Run all E2E tests:**
```shell
make e2e-test
```

**Run without rebuilding images:**
```shell
make E2E_BUILD_IMAGES=0 e2e-test
```

**Run specific test package:**
```shell
make E2E_TESTS=./tests/e2e/tests/skeleton e2e-test
```

**List available test packages:**
```shell
make ls-e2e-test
```

### Using Existing Cluster

**Run tests on your own cluster:**
```shell
go test ./tests/e2e/tests/... -kubeconfig ~/.kube/config
```

## Test Architecture

### 1. Test Runner Setup

Every E2E test starts with a `TestMain` function that initializes the runner:

```go
var runner *runners.Runner

func TestMain(m *testing.M) {
    runner = runners.NewRunner().Init()

    // Optional: custom setup
    runner.Setup(func(ctx context.Context, c *envconf.Config) (context.Context, error) {
        // Your setup code
        return ctx, nil
    })

    runner.Run(m)
}
```

**The runner automatically:**
1. Creates a temporary kind cluster (if no kubeconfig provided)
2. Detects minimum kernel version across all nodes
3. Installs Tetragon via Helm
4. Sets up port forwarding for gRPC and metrics
5. Creates export directories for test artifacts

### 2. Test Structure

A typical E2E test follows this pattern:

```go
func TestExample(t *testing.T) {
    // 1. Get kernel version
    kversion := helpers.GetMinKernelVersion(t, runner.Environment)

    // 2. Create event checker with limits
    checker := createEventChecker(kversion).
        WithEventLimit(100).
        WithTimeLimit(30 * time.Second)

    // 3. Define test features (steps)
    runEventChecker := features.New("Run Event Checks").
        Assess("Run Event Checks", checker.CheckInNamespace(1*time.Minute, namespace)).
        Feature()

    runWorkload := features.New("Run Workload").
        Assess("Wait for Checker", checker.Wait(30*time.Second)).
        Assess("Deploy Workload", deployWorkload).
        Feature()

    // 4. Run features in parallel
    runner.TestInParallel(t, runEventChecker, runWorkload)
}
```

### 3. Event Checking System

The `RPCChecker` connects to Tetragon's gRPC stream and validates events:

```go
func createEventChecker(kernelVersion string) *checker.RPCChecker {
    eventChecker := ec.NewUnorderedEventChecker(
        ec.NewProcessExecChecker("curl-checker").
            WithProcess(
                ec.NewProcessChecker().
                    WithPod(ec.NewPodChecker().
                        WithNamespace(sm.Full("my-namespace"))).
                    WithBinary(sm.Suffix("curl")),
            ),
    )

    return checker.NewRPCChecker(eventChecker, "curlEventChecker")
}
```

**Event Checker Features:**
- **Event Limits**: Stop after N events with `WithEventLimit(N)`
- **Time Limits**: Stop after duration with `WithTimeLimit(duration)`
- **Namespace Filtering**: `CheckInNamespace(timeout, namespace...)`
- **Custom Filters**: `CheckWithFilters(timeout, allowList, denyList)`
- **Kernel-Specific Checks**: Conditionally add checks based on kernel version

**Event Checker Lifecycle:**
1. Start listening to gRPC stream from all Tetragon pods
2. Signal ready via `Wait()` mechanism
3. Match incoming events against expected patterns
4. Stop when event/time limit reached or all checks pass
5. Export matched events and unmatched checks to JSON

### 4. Features & Parallel Execution

Tests use **features** for composable test steps:

```go
// Define a feature with multiple assessments
feature := features.New("Feature Name").
    Assess("Step 1", func(ctx context.Context, t *testing.T, c *envconf.Config) context.Context {
        // Do something
        return ctx
    }).
    Assess("Step 2", func(ctx context.Context, t *testing.T, c *envconf.Config) context.Context {
        // Do another thing
        return ctx
    }).
    Feature()

// Run features sequentially
runner.Test(t, feature1, feature2)

// Run features in parallel
runner.TestInParallel(t, eventChecker, workload)
```

**Typical parallel pattern:**
- Feature 1: Event checker (runs in background)
- Feature 2: Wait for checker, then deploy workload

## Writing a New E2E Test

### Step 1: Copy the Skeleton Test

```shell
cp -r tests/e2e/tests/skeleton tests/e2e/tests/mytest
```

### Step 2: Update Package and Namespace

```go
package mytest_test

const namespace = "mytest"
```

### Step 3: Define Your Event Checker

```go
func myEventChecker(kernelVersion string) *checker.RPCChecker {
    checker := ec.NewUnorderedEventChecker(
        // Define expected events here
        ec.NewProcessExecChecker("my-check").
            WithProcess(
                ec.NewProcessChecker().
                    WithBinary(sm.Suffix("mybinary")),
            ),
    )

    // Add kernel-specific checks if needed
    if kernels.KernelStringToNumeric(kernelVersion) >= kernels.KernelStringToNumeric("5.10.0") {
        checker.AddChecks(
            // Additional checks for newer kernels
        )
    }

    return checker.NewRPCChecker(checker, "myEventChecker")
}
```

### Step 4: Define Your Workload

```go
const myWorkload = `
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mycontainer
    image: myimage:latest
    command: ["mybinary"]
    args: ["arg1", "arg2"]
  restartPolicy: Never
`
```

### Step 5: Write Your Test

```go
func TestMyFeature(t *testing.T) {
    kversion := helpers.GetMinKernelVersion(t, runner.Environment)
    checker := myEventChecker(kversion).
        WithEventLimit(50).
        WithTimeLimit(30 * time.Second)

    runChecker := features.New("Run Event Checks").
        Assess("Check Events", checker.CheckInNamespace(1*time.Minute, namespace)).
        Feature()

    runWorkload := features.New("Deploy Workload").
        Assess("Wait for Checker", checker.Wait(30*time.Second)).
        Assess("Deploy Pod", func(ctx context.Context, _ *testing.T, c *envconf.Config) context.Context {
            ctx, err := helpers.LoadCRDString(namespace, myWorkload, true)(ctx, c)
            if err != nil {
                t.Fail()
            }
            return ctx
        }).
        Feature()

    runner.TestInParallel(t, runChecker, runWorkload)
}
```

## Test Lifecycle

### Setup Phase (runs once)

1. **Cluster Setup**: Creates kind cluster if no kubeconfig
   ```go
   runner.WithSetupClusterFn(helpers.MaybeCreateTempKindCluster)
   ```

2. **Kernel Detection**: Queries minimum kernel version
   ```go
   runner.Setup(helpers.SetMinKernelVersion())
   ```

3. **Tetragon Installation**: Installs via Helm
   ```go
   runner.WithInstallTetragon(
       tetragon.WithHelmOptions(map[string]string{
           "tetragon.exportAllowList": "",
           "tetragon.enablePolicyFilter": "true",
       }),
   )
   ```

4. **Port Forwarding**: Forwards Tetragon pods' gRPC/metrics ports
   ```go
   runner.Setup(helpers.PortForwardTetragonPods(runner.Environment))
   ```

### BeforeEach Test

1. Create export directory: `/tmp/tetragon-e2e-export/<test-name>/`
2. Start metrics dumper (every 30 seconds)
3. Start gops dumper (every 30 seconds)

### AfterEach Test

1. **If test failed**: Dump cluster info to export directory
2. **If test passed**: Remove export directory (unless `-tetragon.keep-export=true`)

### Finish Phase

1. Dump setup info if Tetragon installation failed
2. Uninstall Tetragon (if `-tetragon.uninstall=true`)
3. Delete temporary kind cluster (if created)

## Kind Cluster Configuration

When no kubeconfig is provided, the runner creates a kind cluster with:

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
  - hostPath: "/proc"
    containerPath: "/procRoot"        # Real host proc (not kind node)
  - hostPath: "/tetragonExport"
    containerPath: "/tetragonExport"  # Test artifacts export
  - hostPath: "/sys/fs/bpf"
    containerPath: "/sys/fs/bpf"
    propagation: Bidirectional
```

**Special Tetragon Configuration for Kind:**
- `/procRoot` → `/procRootReal` in Tetragon pod
- `tetragon.procfs=/procRootReal` ensures Tetragon sees real host processes
- `/tetragonExport` → Direct access to host filesystem for test artifacts

## Command Line Flags

### Common Flags

```shell
# Use existing cluster
go test ./tests/e2e/tests/... -kubeconfig ~/.kube/config

# Specify Helm chart
go test ./tests/e2e/tests/... -tetragon.helm.chart=/path/to/chart

# Set Helm values
go test ./tests/e2e/tests/... -tetragon.helm.set key=value

# Keep test artifacts on success
go test ./tests/e2e/tests/... -tetragon.keep-export=true

# Uninstall Tetragon after tests
go test ./tests/e2e/tests/... -tetragon.uninstall=true

# Use custom BTF file
go test ./tests/e2e/tests/... -tetragon.btf=/path/to/btf

# Custom cluster name
go test ./tests/e2e/tests/... -cluster-name=my-test-cluster

# Custom kind image
go test ./tests/e2e/tests/... -cluster-image=kindest/node:v1.33.2
```

### Environment Variables

```shell
# Skip image rebuild
E2E_BUILD_IMAGES=0 make e2e-test

# Specify custom images
E2E_AGENT=myregistry/tetragon:latest make e2e-test
E2E_OPERATOR=myregistry/tetragon-operator:latest make e2e-test

# Run specific tests
E2E_TESTS=./tests/e2e/tests/mytest make e2e-test
```

## Debugging Failed Tests

### Export Directory Structure

When tests fail, artifacts are saved to `/tmp/tetragon-e2e-export/<test-name>/`:

```
/tmp/tetragon-e2e-export/TestMyFeature/
├── myEventChecker.eventchecker.events.json      # All events received
├── myEventChecker.eventchecker.unmatched.json   # Checks that didn't match
├── metrics.0.txt                                 # Metrics dump (t=0s)
├── metrics.30.txt                                # Metrics dump (t=30s)
├── gops.0.txt                                    # Gops dump (t=0s)
├── gops.30.txt                                   # Gops dump (t=30s)
└── cluster-dump/                                 # Cluster state
    ├── pods.txt
    ├── daemonsets.txt
    └── logs/
```

### Event Checker Logs

The event checker logs show real-time event matching:

```
ProcessExecEvent:1 => MATCH, continuing
ProcessExecEvent:2 => no match: binary mismatch, continuing
ProcessExecEvent:3 => MATCH, continuing
ProcessExecEvent:4 => FINAL MATCH
DONE!
```

### Keep Export Files

```shell
# Keep artifacts even on success
go test ./tests/e2e/tests/... -tetragon.keep-export=true
```

### Verbose Logging

```shell
# Increase verbosity
go test ./tests/e2e/tests/... -v -args -v=4
```

## Runner Customization

### Custom Runner Configuration

```go
func TestMain(m *testing.M) {
    runner = runners.NewRunner().
        WithKeepExportFiles(true).                    // Keep artifacts
        NoPortForward().                               // Skip port forwarding
        WithInstallTetragon(                          // Custom Tetragon config
            tetragon.WithHelmOptions(map[string]string{
                "tetragon.grpc.enabled": "true",
                "tetragon.exportAllowList": "",
            }),
        ).
        Init()

    runner.Run(m)
}
```

### Custom Setup/Cleanup

```go
func TestMain(m *testing.M) {
    runner = runners.NewRunner().Init()

    // Custom setup
    runner.Setup(func(ctx context.Context, c *envconf.Config) (context.Context, error) {
        // Create custom resources
        return ctx, nil
    })

    // Custom cleanup
    runner.Finish(func(ctx context.Context, c *envconf.Config) (context.Context, error) {
        // Clean up custom resources
        return ctx, nil
    })

    runner.Run(m)
}
```

## Helper Functions

### Namespace Management

```go
// Create namespace
ctx, err := helpers.CreateNamespace("my-namespace", true)(ctx, c)

// Delete namespace
ctx, err := helpers.DeleteNamespace("my-namespace", true)(ctx, c)
```

### Resource Loading

```go
// Load YAML/JSON string
ctx, err := helpers.LoadCRDString(namespace, yamlString, true)(ctx, c)

// Unload resource
ctx, err := helpers.UnloadCRDString(namespace, yamlString, true)(ctx, c)
```

### Executing Commands in Pods

```go
// Execute command in pod
output, err := helpers.ExecInPod(ctx, c, namespace, podName, containerName, []string{"ls", "-la"})
```

## Best Practices

1. **Use Unique Test Namespaces**: Each test should use its own namespace
   ```go
   const namespace = "mytest"
   ```

2. **Set Appropriate Limits**: Balance between test reliability and speed
   ```go
   checker.WithEventLimit(100).WithTimeLimit(30 * time.Second)
   ```

3. **Clean Up Resources**: Remove test resources in teardown
   ```go
   runner.Finish(helpers.DeleteNamespace(namespace, true))
   ```

4. **Use Parallel Features**: Run event checkers and workloads concurrently
   ```go
   runner.TestInParallel(t, runChecker, runWorkload)
   ```

5. **Export Events for Debugging**: Always use unique checker names
   ```go
   checker.NewRPCChecker(eventChecker, "uniqueCheckerName")
   ```

6. **Handle Kernel Differences**: Check kernel version before adding checks
   ```go
   if kernels.KernelStringToNumeric(kversion) >= kernels.KernelStringToNumeric("5.10.0") {
       // Add kernel-specific checks
   }
   ```

7. **Wait for Checker**: Always wait for event checker to start
   ```go
   Assess("Wait for Checker", checker.Wait(30*time.Second))
   ```

## Common Patterns

### Testing TracingPolicy

```go
const tracingPolicy = `
apiVersion: cilium.io/v1alpha1
kind: TracingPolicy
metadata:
  name: my-policy
spec:
  kprobes:
  - call: "sys_open"
    syscall: true
    args:
    - index: 0
      type: "string"
`

// In test
Assess("Load Policy", func(ctx context.Context, _ *testing.T, c *envconf.Config) context.Context {
    ctx, err := helpers.LoadCRDString(namespace, tracingPolicy, true)(ctx, c)
    if err != nil {
        t.Fail()
    }
    return ctx
})
```

### Testing Process Execution

```go
checker := ec.NewOrderedEventChecker(
    ec.NewProcessExecChecker("exec").
        WithProcess(ec.NewProcessChecker().
            WithBinary(sm.Full("/usr/bin/curl"))),
    ec.NewProcessExitChecker("exit").
        WithProcess(ec.NewProcessChecker().
            WithBinary(sm.Full("/usr/bin/curl"))),
)
```

### Testing File Access

```go
checker := ec.NewUnorderedEventChecker(
    ec.NewProcessKprobeChecker("open").
        WithFunctionName(sm.Full("__x64_sys_open")).
        WithArgs(ec.NewKprobeArgumentListMatcher().
            WithValues(
                ec.NewKprobeArgumentChecker().WithFileArg(
                    ec.NewKprobeFileChecker().
                        WithPath(sm.Full("/etc/passwd")),
                ),
            ),
        ),
)
```

## Troubleshooting

### Tests Hang

- Check if parallel tests are enabled (automatic in runner)
- Verify event checker has appropriate limits
- Ensure workload actually generates expected events

### Events Not Matching

1. Check `*.eventchecker.events.json` - Are events being received?
2. Check `*.eventchecker.unmatched.json` - What checks didn't match?
3. Verify namespace filtering
4. Check binary paths and arguments

### Cluster Creation Fails

- Ensure Docker is running
- Check kind is installed: `kind version`
- Verify network connectivity

### Image Loading Fails

- Build images first: `make images`
- Check image exists: `docker images | grep tetragon`
- Use `E2E_BUILD_IMAGES=1` to force rebuild

## Additional Resources

- [E2E Framework Documentation](https://pkg.go.dev/sigs.k8s.io/e2e-framework)
- [Skeleton Test](tests/e2e/tests/skeleton/skeleton_test.go) - Template for new tests
- [Event Checker Documentation](https://pkg.go.dev/github.com/cilium/tetragon/api/v1/tetragon/codegen/eventchecker)
- [Contribution Guide](https://tetragon.io/docs/contribution-guide/)
