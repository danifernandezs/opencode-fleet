---
name: debug-container
description: Diagnose and troubleshoot a Docker container that is failing or misbehaving. Inspects logs, state, configuration, and allows exec into the container to investigate.
---

## What I do

Investigate a broken container: logs, state, config, and interactive debugging.

## When to use me

Use this when a container is crashing, returning errors, or behaving unexpectedly.

## Steps

### 1. Find the container

```bash
curl -s http://socket-proxy:2375/containers/json?all=true
```

Identify the container by name or image. Note its status and state.

### 2. Inspect state and config

```bash
curl -s http://socket-proxy:2375/containers/{id}/json | jq '{State: .State, Config: .Config, HostConfig: .HostConfig}'
```

Check:
- `State.Status` (running, exited, restarting)
- `State.ExitCode` (non-zero means crash)
- `State.Error` (error message)
- `HostConfig.Binds` (are volumes correct?)
- `NetworkSettings.Ports` (are ports mapped?)

### 3. Read logs

```bash
curl -s "http://socket-proxy:2375/containers/{id}/logs?stdout=true&stderr=true&tail=100"
```

Look for error messages, stack traces, or startup failures.

### 4. Exec into the container

If the container is running but misbehaving:

```bash
# Create exec instance
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"AttachStdout": true, "AttachStderr": true, "Cmd": ["sh", "-c", "env && ls -la /"]}' \
  http://socket-proxy:2375/containers/{id}/exec
```

Then start it:

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"Detach": false, "Tty": false}' \
  http://socket-proxy:2375/exec/{exec_id}/start
```

### 5. Common issues to check

- **Exited immediately**: wrong command, missing dependencies, wrong entrypoint
- **Can't bind port**: port already in use by another container
- **No network access**: missing network config, DNS resolution failing
- **File not found**: volume mount path incorrect on the host side
- **Permission denied**: user in container can't read mounted files

### 6. Remediate

Based on findings:
- Fix configuration and recreate the container
- Fix application code and rebuild
- Adjust volume mounts or network settings
- Restart the container after changes
