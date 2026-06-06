# Minikube Repository - Agent Guidelines

## Project Overview

Minikube is a lightweight Kubernetes development environment that runs on single-node clusters across various platforms (Docker, K3s, libvirt/KVM/QEMU/HyperKit/hyperv). This repo is located at `/root/minikube` and is the actual minikube repository.

**Note:** The path name suggests it's "minikube" but this contains Go code for a CLI tool (main.go files in cmd/), build workflows, and ISO generation infrastructure — not Kubernetes itself despite any confusing git history notes elsewhere.

## Key Directories

```
cmd/minikube/main.go           # Main entry point for minikube binary
deploy/iso/*.go               # Code to generate the minikube Linux ISO (23k+ lines)
deploy/addons/*              # Addon implementations (e.g., ingress, metrics-server)
hack/codegen/k8sio           # Kubernetes API code generation stub files
test/integration             # Integration tests for functional testing
cmd/auto-pause/             # Binary to automatically pause/resume virtual machines on power loss events.
```

## Common Commands & Workflows

### Building the minikube binary:
- `make build` → cross-platform builds; see cmd/minikube/main.go and deploy/kicbase for details.

### Running tests locally (requires preinstalled deps):
- Go test requires external packages like Gomega, Gobdd, Kubectl stubs, etc., installed via make vendoring or the repo's vendor.sh script.  
  Use `make test` to run unit tests after dependencies are in place; some integration tests may need a real VM/hardware environment before being able to connect.
- `/test/unit-test.yml`: Unit test workflow (no extra deps needed if vendored).

### Building/minifying the ISO:
```bash
hack/build.sh           # Build minikube binary and generate ISO image for current architecture  
make build-minikubeiso  # Same as above, via Makefile target. 
./minikube start        # Start a VM using that generated ISO
```

### Development workflow after making changes:
1. Save your code → `hack/build.sh` or just go ahead and run it directly if modifying testable logic  
2. Run tests with `/test/unit-test.yml` to confirm regressions (no need for external infrastructure unless integrating)  
3. For integration testing, either spin up a fresh VM from the updated ISO (`./minikube delete; ./minikube start`) or reuse an existing cluster if it's already running locally before making changes

### Code organization notes:
- `deploy/iso` (~25k lines of Go) handles minikube boot images including cloud-init, containerd binaries (containerd, runc), CNI plugins for network setup during VM creation.  
  - The ISO builder generates a Linux image with `/etc/container/config.toml`, `/var/lib/minikube/iso-version` tracking releases (`20547b9138e6...`).
- `deploy/addons`: Contains implementations like ingress, metrics-server for kubectl add-ons. Addon install logic is in deploy/kicbase/.
- Each subdirectory under cmd/* provides a self-contained Go module (main.go entry point).

## Tool Usage Patterns

### Bash commands you'll frequently need:
```bash
make build          # Cross-platform binary builds  
./minikube start    # Launch an existing VM cluster or create from the ISO file in deploy/iso  
hack/build.sh       # Same as ./minikube, used by CI jobs. 
# For cross-compilation to other archs (Docker, KVM/QEMU): see hack/crosscompile-<arch>.sh scripts
```

### Code generation:
When modifying Kubernetes API structures or adding new CRDs: update stub files under `hack/codegen/k8sio/` and re-run code gen (`./scripts/run-code-gen.sh`). Generated types live in vendor/github.com/kubernetes-client-go.

## Recommended MCP Tool Usage Patterns for Agents

1. **Bash**: Build binaries, run tests with vendored deps installed (check Makefile first), cross-compile to new archs  
2. **Read** (and Edit): Modify go.mod/vendor when adding Go packages; edit deploy/iso/*.go for ISO content changes  
3. **Write/Edit**: Update stub files in hack/codegen/k8sio after modifying Kubernetes API structures or generating CRDs  
4. **Agent/**general-purpose: Multi-step tasks like updating addon implementations under addons/, running integration tests, or regenerating crosscompile scripts

## Build & Test Infrastructure (CI)

- `.github/workflows/build.yml`: Cross-compilation matrix targeting Docker/K3s/VM drivers
- `.github/workflows/smoke-test.yml`: Minimal functional smoke checks against the built binary  
- `.github/workflows/functional_test.yml`: Full integration suite with k8s cluster validation steps.  

## Common Pitfalls

1. **ISO version tracking**: After deploying ISO updates, remember to bump deploy/iso/minikube-iso/board/*/rootfs-overlay/etc/VERSION or run hack/build.sh for new release versions
2. **Test dependencies**: Go tests need external vendored packages (Gomega/Gobdd/kubectl stubs) installed via `make vendor` before running test commands—don't skip this step if you're seeing "missing dependency" errors
3. **Cross-compilation scripts**: hack/crosscompile-<arch>.sh files are maintained in parallel; updating one requires syncing others for consistency

## Additional Notes

- The `.claude/` directory is excluded from the Git repository — agents should save artifacts there rather than polluting working directories  
- This repo has extensive integration testing with external dependencies (VM drivers, Docker/K3s clusters) that may require preinstallation on your local environment before running certain test suites