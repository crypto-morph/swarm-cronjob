# Swarm-Cronjob Docker SDK v27 Fix Guide

## Problem

After upgrading to Docker SDK v27 and Go 1.23, the swarm-cronjob service fails to start in Docker Swarm with exit code 1, showing `0/1` replicas running. The service works fine when run directly with `docker run` but fails in Swarm.

**Root Cause:** The `command.NewDockerCli()` initialization in Docker SDK v27 applies default options that try to access Docker config files and set up terminal streams, which fail in the Swarm container environment.

## Fix

### Step 1: Update the Code

Edit `internal/docker/client.go`:

**Add the `os` import:**

```diff
 import (
 	"context"
+	"os"
 	"strings"

 	"github.com/crazy-max/swarm-cronjob/internal/model"
```

**Update the Docker CLI initialization (around line 47):**

```diff
-	dockerCli, err := command.NewDockerCli()
+	dockerCli, err := command.NewDockerCli(
+		command.WithCombinedStreams(os.Stderr),
+		command.WithContentTrustFromEnv(),
+	)
 	if err != nil {
 		return nil, errors.Wrap(err, "failed to create Docker cli")
 	}
```

### Step 2: Build the Fixed Docker Image

```bash
# Navigate to the swarm-cronjob directory
cd /path/to/swarm-cronjob

# Build the Docker image
docker build -t ghcr.io/voinetwork/swarm-cronjob:edge-fix .
```

### Step 3: Update the Swarm Service

```bash
# Update the service to use the fixed image
docker service update --image ghcr.io/voinetwork/swarm-cronjob:edge-fix voinetwork_scheduler

# Wait for the service to converge (about 30 seconds)
# You should see "Service voinetwork_scheduler converged"
```

### Step 4: Verify the Fix

```bash
# Check that the service is running (should show 1/1 replicas)
docker service ls

# Verify the task is running
docker service ps voinetwork_scheduler --filter "desired-state=running"

# Check the logs to confirm it started successfully
docker service logs voinetwork_scheduler --tail 5
```

You should see logs like:
```
[90mSun, 18 Jan 2026 22:44:11 UTC[0m [32mINF[0m [1mStarting swarm-cronjob 94813ba.m[0m
[90mSun, 18 Jan 2026 22:44:11 UTC[0m [32mINF[0m [1mAdd cronjob with schedule 4 */3 * * *[0m [36mservice=[0mvoinetwork_shepherd
```

## Quick One-Liner Test

To quickly test if this is the issue on another server:

```bash
# Check if the service is failing
docker service ps voinetwork_scheduler --no-trunc | grep -i failed
```

If you see multiple failed tasks with "task: non-zero exit (1)", this fix will resolve it.

## Alternative: Use Pre-built Image

If you don't want to build locally, you can push the image to a registry first:

```bash
# On the working server, tag and push the image
docker tag ghcr.io/voinetwork/swarm-cronjob:edge-fix ghcr.io/voinetwork/swarm-cronjob:fixed
docker push ghcr.io/voinetwork/swarm-cronjob:fixed

# On the other server, update the service
docker service update --image ghcr.io/voinetwork/swarm-cronjob:fixed voinetwork_scheduler
```

## Notes

- The fix removes filesystem dependencies from the Docker CLI initialization
- Only minimal options are provided: combined streams and content trust from environment
- This is sufficient for the authentication functionality needed by swarm-cronjob
- The fix works with Docker SDK v27 and Go 1.23
