# GoCD Kubernetes Executor Task Plugin

## Description

The plugin enables a GoCD Kubernetes Elastic Agent to execute workloads inside a guest container within the same Pod. It relies on `nsenter` and leverages namespace sharing between containers, allowing the agent setup to remain agnostic and enabling the use of any base image without specific setup requirements.

## Technical Overview

### Architecture

The plugin operates as a bridge between GoCD agents and containerized workloads:

1. GoCD task configuration defines script content and shell type
2. Script is written to a shared filesystem accessible by both host and container
3. `nsenter` command is used to enter container namespaces and execute the script
4. Output streams are captured and forwarded to GoCD console logger
5. Temporary script files are cleaned up post-execution

### Execution Flow

```
GoCD Job → Plugin Handler → Script Generation → nsenter Invocation → Container Execution → Result Collection
```

### Key Components

- **Shell Support**: bash, zsh, sh
- **Namespace Entry**: Uses `nsenter` with `--target`, `--all`, `--wdns` flags
- **Environment Propagation**: All GoCD environment variables are exported to the container context
- **Working Directory**: Maintains correct working directory through `--wdns` parameter

## Prerequisites

### Required settings

The nsenter utility must be available in the agent container’s filesystem and accessible in the $PATH.

Before executing tasks, the agent must ensure that the target (guest) container is in a ready state and that all required environment variables are correctly configured.

The following environment variables must be set by the GoCD Kubernetes Elastic Agent to correctly interact with the guest container:

| Variable | Description |
|----------|-------------|
| `GO_K8S_CONTAINER_SHARED_ROOT` | Shared filesystem path accessible by both host and container |
| `GO_K8S_CONTAINER_PID` | Process ID of the target container's main process |
| `GO_K8S_CONTAINER_ROOT` | Root directory path within the container |


### Elastic agent configuration

The use of nsenter requires additional capabilities. The Pod must enable shareProcessNamespace: true, and the agent container must run with the SYS_ADMIN capability to allow namespace access.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gocd-build-pod
spec:
  shareProcessNamespace: true
  volumes:
  - name: shared-workspace
    emptyDir: {}
  containers:
    - name: gocd-agent
      image: custom/gocd-k8s-elastic-agent
      securityContext:
        allowPrivilegeEscalation: false
        capabilities:
          add:
            - SYS_ADMIN
      volumeMounts:
        - mountPath: /go
          name: shared-workspace
    
    - name: build-environment
      image: node:16.14.0  # Any base image from Docker Hub
      volumeMounts:
        - mountPath: /go
          name: shared-workspace
      command: ["/go/pause"]
```
## Installation

1. Build the plugin:
   ```bash
   ./gradlew clean build
   ```

2. Locate the generated JAR file in `build/libs/`

3. Deploy to GoCD server:
   - Copy JAR to `<gocd-server>/plugins/external/`
   - Restart GoCD server

## Configuration

### Task Configuration

In GoCD pipeline configuration, add a task of type "k8s Executor":

- **Shell**: Select shell interpreter (bash, zsh, or sh)
- **Script**: Multi-line script content to execute

### Example Pipeline Configuration (XML)

```yaml
            tasks:
            - plugin:
                configuration:
                  id: k8s-executor
                  version: 1.0.0
                options:
                  shtype: sh
                  script: |
                    set -e

                    echo "[INFO] Starting Node.js build pipeline..."

                    # Install dependencies
                    echo "[STEP] Installing dependencies..."
                    npm ci

                    # Run build
                    echo "[STEP] Building project..."
                    npm run build

                    # Run tests
                    echo "[STEP] Running tests..."
                    npm test

                    echo "[SUCCESS] Build and tests completed successfully."
```

## Implementation Details

### Script Execution Command

```bash
nsenter \
  --target <GO_K8S_CONTAINER_PID> \
  --all \
  --wdns=<working_directory> \
  -e \
  -- \
  <shell_type> \
  -c \
  "<environment_variables> <shell_type> <script_path>"
```

### Error Handling

- Exit code 0: Task success
- Non-zero exit code: Task failure with exit code logged
- Exception during execution: Task failure with error message
- Script cleanup: Always attempted, failures silently ignored

### Security Considerations

- Scripts are created with user-executable permissions only (`chmod u+x`)
- Temporary script files use UUID-based naming to prevent collisions
- Environment variables are properly escaped during export
- No Windows support reduces attack surface for that platform

## Build Configuration

- **Plugin ID**: `k8s-executor`
- **Version**: 1.0.0
- **Minimum GoCD Version**: 20.1.0
- **Java Version**: 11
- **Dependencies**:
  - GoCD Plugin API: 24.1.0
  - Gson: 2.11.0

## Limitations

- Linux-only operation
- Requires `nsenter` availability
- Depends on specific environment variables set by Kubernetes infrastructure
- No native Windows container support
- Shared filesystem requirement between host and container


## Development

### Running Tests

```bash
./gradlew test
```

### Building Release

```bash
export PRERELEASE=No
./gradlew clean build
```

## Troubleshooting

### Common Issues

**Script not found**: Verify `GO_K8S_CONTAINER_SHARED_ROOT` is correctly mounted in both host and container

**Permission denied**: Ensure GoCD agent has permissions to execute `nsenter` (typically requires elevated privileges)

**nsenter command not found**: Install `util-linux` package containing `nsenter`

**Environment variables not propagated**: Check that variable names do not contain special characters that break shell escaping
