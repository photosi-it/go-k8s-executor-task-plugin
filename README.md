# GoCD Kubernetes Executor Task Plugin

## Description

Task plugin for GoCD that enables execution of shell scripts within Kubernetes containers using Linux namespaces. The plugin leverages `nsenter` to execute commands inside target containers while maintaining GoCD's environment context and logging capabilities.

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

### System Requirements

- Linux operating system (Windows not supported)
- `nsenter` utility available in PATH
- Kubernetes cluster with appropriate pod security policies

### Environment Variables

The following environment variables must be set by the GoCD Kubernetes agent:

| Variable | Description |
|----------|-------------|
| `GO_K8S_CONTAINER_SHARED_ROOT` | Shared filesystem path accessible by both host and container |
| `GO_K8S_CONTAINER_PID` | Process ID of the target container's main process |
| `GO_K8S_CONTAINER_ROOT` | Root directory path within the container |

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

```xml
<task>
  <pluginConfiguration id="k8s-executor" version="1.0.0" />
  <configuration>
    <property>
      <key>shtype</key>
      <value>bash</value>
    </property>
    <property>
      <key>script</key>
      <value><![CDATA[
#!/bin/bash
set -e
echo "Starting deployment"
kubectl apply -f deployment.yaml
]]></value>
    </property>
  </configuration>
</task>
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

## License

Apache License 2.0

## Vendor

**PhotoSì**  
https://www.photosi.com

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
