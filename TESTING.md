# Testing Guide

Tetragon has a comprehensive testing framework with multiple test types to ensure code quality across different kernels and architectures.

## Test Types

### 1. Go Unit Tests

Unit tests for userspace Go code and BPF integration.

**Run all tests:**
```shell
make test
```

**Run with custom flags:**
```shell
make test EXTRA_TESTFLAGS="-v -run TestSpecificTest"
```

**Details:**
- Located throughout the codebase as `*_test.go` files
- Runs with sudo permissions
- Sequential execution (`-p 1 -parallel 1`)
- Default timeout: 20 minutes (configurable via `GO_TEST_TIMEOUT`)
- Coverage reporting enabled

### 2. BPF Unit Tests

Tests for specific BPF functions using the `cilium/ebpf` framework.

**Run BPF tests:**
```shell
make bpf-test
```

**Run with verbose output:**
```shell
make BPFGOTESTFLAGS="-v" bpf-test
```

**Details:**
- Located in `bpf/tests/`
- Uses Go tests with `cilium/ebpf` to run BPF code tests
- Test specific BPF functionality in isolation

### 3. End-to-End (E2E) Tests

End-to-end tests that install Tetragon in Kubernetes clusters and test complete features.

**Run E2E tests:**
```shell
make e2e-test
```

**Run without rebuilding images:**
```shell
make E2E_BUILD_IMAGES=0 e2e-test
```

**Run specific test:**
```shell
make E2E_TESTS=./tests/e2e/tests/skeleton e2e-test
```

**List available E2E test packages:**
```shell
make ls-e2e-test
```

**Details:**
- Located in `tests/e2e/tests/`
- Creates a kind cluster automatically
- Installs Tetragon using Helm
- Runs tests against the cluster
- Default timeout: 20 minutes (configurable via `E2E_TEST_TIMEOUT`)
- Custom images can be set via `E2E_AGENT` and `E2E_OPERATOR` variables

### 4. VM Tests

Tests across different kernel versions using [little-vm-helper](https://github.com/cilium/little-vm-helper).

**Details:**
- Located in `tests/vmtests/`
- See `tests/vmtests/README.md` for detailed documentation
- Used in CI to test against multiple kernel versions
- Kernel versions tested are defined in `.github/workflows/vmtests.yml`

## Test Utilities

### Test Compilation

Pre-compile all tests without running them:

```shell
make test-compile
```

This is useful for:
- Checking test compilation errors
- Parallel compilation of test binaries
- CI pipelines

### Tester Programs

Helper binaries used in tests:

```shell
make tester-progs
```

Located in `contrib/tester-progs/`, these are auxiliary programs used by the test suite.

### BPF Verification

Verify BPF programs:

```shell
make verify
```

### Benchmarks

Run Go benchmarks:

```shell
make bench
```

Customize with:
```shell
make bench EXTRA_TESTFLAGS="-bench=BenchmarkSpecific"
```

### Alignment Checker

Run alignment checker for BPF structures:

```shell
make alignchecker
```

## CI Testing

Tests run automatically in CI on:
- **Multiple kernel versions** - See `jobs.test.strategy.matrix.kernel` in `.github/workflows/vmtests.yml`
- **Multiple architectures** - amd64 and arm64

## Development Workflows

### Running Tests Locally

1. **Quick test iteration:**
   ```shell
   # Build BPF and test helpers once
   make tetragon-bpf tester-progs

   # Run specific test
   go test -exec sudo -v ./pkg/mypackage -run TestMyFunction
   ```

2. **Full test suite:**
   ```shell
   make test
   ```

3. **Test on specific kernel:**
   - See `tests/vmtests/README.md` for VM test setup

### Test Configuration

**Environment Variables:**
- `GO_TEST_TIMEOUT` - Timeout for Go tests (default: 20m)
- `E2E_TEST_TIMEOUT` - Timeout for E2E tests (default: 20m)
- `EXTRA_TESTFLAGS` - Additional flags for test binary
- `BPFGOTESTFLAGS` - Additional flags for BPF tests
- `E2E_BUILD_IMAGES` - Build images before E2E tests (default: 1)
- `E2E_AGENT` - Agent image for E2E tests
- `E2E_OPERATOR` - Operator image for E2E tests
- `E2E_BTF` - BTF file to use in E2E tests

## Code Quality

### Linting

Run Go linters:
```shell
make check
```

Uses `golangci-lint` with project-specific configuration.

### Formatting

Format all code:
```shell
make format
```

This runs:
- `go-format` - Go code formatting with `goimports`
- `clang-format` - BPF C code formatting

### Validation

Run all validation checks (linting, formatting, code generation):
```shell
make validate
```

This ensures:
- Code is properly formatted
- Generated code is up to date
- Linters pass
- Documentation is current

## Troubleshooting

### Common Issues

1. **Permission denied errors**
   - Tests require sudo permissions
   - Ensure your user can run sudo

2. **BPF compilation failures**
   - Ensure you have BPF development headers
   - Try using containerized build: `make tetragon-bpf` (default)
   - Or local clang: `make tetragon-bpf LOCAL_CLANG=1`

3. **E2E tests fail to create cluster**
   - Ensure Docker/Podman is running
   - Check that kind is installed
   - Verify network connectivity for pulling images

4. **Test timeouts**
   - Increase timeout: `make test GO_TEST_TIMEOUT=30m`
   - Run tests sequentially (already default)

### Debug Mode

Enable debug builds for better error messages:
```shell
make DEBUG=1 tetragon-bpf
make DEBUG=1 tetragon
```

This enables:
- No optimization (`NOOPT=1`)
- No symbol stripping (`NOSTRIP=1`)
- Debug flags for BPF programs

## Contributing

Before submitting a PR:

1. Run the full test suite:
   ```shell
   make test
   make bpf-test
   ```

2. Run validation:
   ```shell
   make validate
   ```

3. Ensure E2E tests pass (if touching core functionality):
   ```shell
   make e2e-test
   ```

For more information, see the [Contribution Guide](https://tetragon.io/docs/contribution-guide/).
