---
name: deploy-and-test
description: Deploy an application in a Docker container and verify it works using the browser. Builds, runs, waits for health, navigates the UI, checks for errors, and fixes issues in a loop.
---

## What I do

Autonomous loop to deploy and verify an application end-to-end.

## When to use me

Use this when you need to build, deploy, and verify an application works correctly in a container, including browser-based UI testing.

## Steps

### 1. Prepare

- Identify how the app starts: Dockerfile, docker-compose, package.json `start` script, etc.
- Determine the port the app exposes.
- Make sure the project code is available at the workspace path.

### 2. Build (if needed)

If there's a Dockerfile, tar the build context and send it:

```bash
tar -cf - -C /workspace . | \
  curl -s -X POST -H "Content-Type: application/tar" \
    --data-binary @- \
    "http://socket-proxy:2375/build?t=app-test"
```

Otherwise, use a base image and run the start command directly.

### 3. Create and start the container

Attach to `fleet-net` so the browser can reach it by container name:

```bash
curl -s -X POST -H "Content-Type: application/json" \
  -d "{
    \"Image\": \"app-test\",
    \"HostConfig\": {
      \"NetworkMode\": \"fleet-net\",
      \"Binds\": [\"/opt/opencode/projects/my-project:/workspace\"]
    }
  }" \
  http://socket-proxy:2375/containers/create?name=app-test
```

Start it:

```bash
curl -s -X POST http://socket-proxy:2375/containers/{id}/start
```

### 4. Wait for readiness

Poll the app via the container name on the shared network (retry for up to 60 seconds):

```bash
curl -s --retry 10 --retry-delay 5 http://app-test:3000/health
```

Adjust the port and health path to match the application.

### 5. Browser verification

Use the Playwright MCP tools:

1. `browser_navigate` to `http://app-test:3000` (container name + port on the shared network)
2. `browser_snapshot` to see the page structure
3. Walk through key user flows (login, navigation, forms)
4. `browser_console_messages` to check for JS errors
5. `browser_network_requests` to check for failed requests
6. `browser_take_screenshot` for visual confirmation if needed

### 6. Fix loop

If errors are found:
1. Read the error details
2. Fix the code in the workspace
3. Rebuild the image
4. Remove the old container: `curl -s -X DELETE http://socket-proxy:2375/containers/{id}?force=true`
5. Recreate and restart
6. Re-verify with the browser

### 7. Cleanup

When done, either leave the container running (if the user wants it deployed) or remove it:

```bash
curl -s -X DELETE http://socket-proxy:2375/containers/{id}?force=true
```

### 8. Commit

When all checks pass, commit the working code.
