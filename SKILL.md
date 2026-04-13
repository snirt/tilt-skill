---
name: tilt
description: Use when working with Tilt (tilt.dev) Tiltfiles - writing resources, configuring dependencies, live_update, helm_resource, debugging builds, or optimizing dev environments
---

# Tilt Reference

## Overview

Tilt orchestrates local dev environments via a `Tiltfile` (Starlark/Python dialect). It watches files, builds images, deploys to Kubernetes or Docker Compose, and hot-reloads changes.

## Key API Functions

### Image Building

| Function | Purpose |
|----------|---------|
| `docker_build(ref, context, dockerfile=, target=, build_args={}, platform=, only=[], ignore=[], live_update=[], ssh=)` | Build Docker image. `only` allowlists files; `ignore` blocklists. `target` selects a multi-stage build target. `platform` forces architecture (e.g., `linux/amd64`). |
| `custom_build(ref, command, deps, live_update=[])` | Non-Docker builds (Buildpacks, Bazel, etc.). |

### Deploying Resources

| Function | Purpose |
|----------|---------|
| `k8s_yaml(yaml)` | Apply raw K8s manifests (file path or Blob). |
| `helm(chart, name=, namespace=, release_name=, values=[], set=[])` | **Template only** - returns rendered YAML. Pass to `k8s_yaml()`. Tilt manages individual K8s objects. Does NOT run `helm install`. |
| `helm_resource(name, chart, release_name=, namespace=, flags=[], image_deps=[], image_keys=[], resource_deps=[], labels=[], deps=[], pod_readiness=, port_forwards=[], auto_init=True, links=[])` | **Full Helm release** - runs `helm install/upgrade`, tracks as single resource. Runs `helm uninstall` on `tilt down`. |
| `docker_compose(path, project_name=, env_file=)` | Load Docker Compose file(s). Each service becomes a Tilt resource. Accepts a single path or list of paths. |
| `dc_resource(name, new_name=, resource_deps=[], labels=[], trigger_mode=, auto_init=True, links=[])` | Configure individual Compose services. Use `new_name` to rename the resource in Tilt UI. |
| `k8s_custom_deploy(name, apply_cmd, delete_cmd, deps=[])` | Custom deploy tools (kustomize, cdk8s, etc.). |

**`helm()` vs `helm_resource()`:** Use `helm()` when you want Tilt to manage individual K8s objects from the chart (port-forward per pod, live_update per deployment). Use `helm_resource()` when you need Helm release lifecycle (hooks, rollback, `helm uninstall` on teardown) or when wiring image builds to chart values via `image_deps`/`image_keys`.

### Resource Configuration

| Function | Purpose |
|----------|---------|
| `k8s_resource(name, new_name=, port_forwards=[], resource_deps=[], labels=[], trigger_mode=, auto_init=True, objects=[], pod_readiness=, links=[])` | Configure a K8s resource. `objects` associates non-workload objects (Secrets, ConfigMaps) with this resource. `pod_readiness='ignore'` skips health checks. |
| `local_resource(name, cmd=, serve_cmd=, deps=[], resource_deps=[], allow_parallel=False, labels=[], trigger_mode=, auto_init=True, readiness_probe=, links=[])` | Run local commands as managed resources. `cmd` runs once; `serve_cmd` runs a long-lived process. **Must set `allow_parallel=True`** for parallel execution. |
| `local(command, quiet=False)` | Run command at **Tiltfile load time** (synchronous). Returns stdout as Blob. Use for setup, config values, git hashes. Not a managed resource. |

### Live Update (Hot Reload)

Steps must follow this strict order:
1. Zero or more `fall_back_on` (must be first)
2. Zero or more `sync`
3. Zero or more `run`
4. Optionally `restart_container()` (last, Docker Compose only)

```python
docker_build('myimage', '.',
  live_update=[
    fall_back_on(['requirements.txt']),    # Full rebuild if these change
    sync('./src', '/app/src'),             # Copy changed files into container
    run('pip install -r requirements.txt', # Run command in container
        trigger=['requirements.txt']),     # Only when this file changes
  ],
  ignore=['tests/', '*.md']
)
```

| Step | Purpose |
|------|---------|
| `fall_back_on(files)` | Force full image rebuild when these files change instead of live updating. Must be first. |
| `sync(local_path, container_path)` | Copy changed files into running container. Local path must be within `docker_build` context. |
| `run(cmd, trigger=[])` | Execute command in container. Trigger files must also be covered by a `sync()` step. Without trigger, runs on every live update. |
| `restart_container()` | Restart container after sync. **Docker Compose only** - does not work with K8s or custom_build. |

**Rebuild triggers:** If a changed file is in the build context but matches no `sync()` path, Tilt skips live update and does a full rebuild. Files outside the build context are ignored entirely.

### File & Data Operations

| Function | Purpose |
|----------|---------|
| `read_file(path, default=)` | Read file contents as Blob. **Registers the file as a Tiltfile dependency** - Tiltfile re-executes when it changes. |
| `read_json(path, default=)` | Parse JSON file into Starlark object. Also registers as dependency. |
| `read_yaml(path, default=)` | Parse YAML file into Starlark object. Also registers as dependency. |
| `watch_file(path)` | Explicitly watch a file/directory for Tiltfile re-execution. Use when `local()` reads files Tilt can't detect. |

**Important:** `local()` doesn't track file reads. If a `local()` command reads a file whose changes should re-trigger the Tiltfile, use `read_file()` or `watch_file()` to register that dependency.

### Health & UI

| Function | Purpose |
|----------|---------|
| `probe(http_get=, tcp_socket=, exec_action=, initial_delay_secs=0, period_secs=10, timeout_secs=1)` | Readiness probe for `local_resource` or `helm_resource`. |
| `http_get_action(port, path=, host='localhost')` | HTTP GET probe - succeeds on status 200-399. |
| `tcp_socket_action(port, host='localhost')` | TCP connection probe - succeeds if socket opens. |
| `link(url, name=)` | Associate a clickable URL with a resource in the Tilt UI. |

```python
local_resource('port-forward',
  serve_cmd='kubectl port-forward svc/myapp 8080:80',
  readiness_probe=probe(
    http_get=http_get_action(port=8080, path='/health/'),
    initial_delay_secs=15,
    period_secs=10,
  ),
  links=[link('http://localhost:8080', 'My App')],
)
```

### Configuration System

```python
# Define parameters (Tiltfile)
config.define_string('env-id')
config.define_bool('enable-feature')
config.define_string_list('services', args=True)
cfg = config.parse()

env_id = cfg.get('env-id', 'default')
```

```json
// tilt_config.json
{ "env-id": "dev", "enable-feature": true, "services": ["api", "web"] }
```

Also settable via CLI: `tilt up -- --env-id=staging`

### Settings

```python
update_settings(
  max_parallel_updates=3,                          # Parallel resource updates (default: 3)
  k8s_upsert_timeout_secs=120,                     # K8s apply timeout
  suppress_unused_image_warnings=['myimage*'],      # Silence unused image warnings
)
```

### Modular Tiltfiles

| Function | Purpose |
|----------|---------|
| `load('ext://name', 'symbol')` | Load from Tilt extension registry |
| `load('./lib/helpers.star', 'fn')` | Load from local file |
| `load_dynamic(path)` | Load at runtime (path can be computed) |

Common extensions:
```python
load('ext://dotenv', 'dotenv')              # Load .env files
load('ext://helm_resource', 'helm_resource') # Helm release management
load('ext://restart_process', 'docker_build_with_restart')  # Process restart on live update
```

## Resource Dependencies & Ordering

```python
k8s_resource('database', labels=['infra'])
k8s_resource('backend', resource_deps=['database'], labels=['app'])
local_resource('migrate', cmd='python migrate.py', resource_deps=['backend'])
```

**Readiness:**
- K8s resources: pods running + readiness probes passing (jobs: completed)
- Local resources: `cmd` exits 0, or `serve_cmd` process running + `readiness_probe` passing
- Docker Compose: container started

**Important:** Dependencies only gate **initial startup**. After first ready, subsequent updates proceed independently.

**Parallel execution:** `update_settings(max_parallel_updates=N)` controls global parallelism. `local_resource` also requires explicit `allow_parallel=True`.

## Resource Organization

```python
# Group resources with labels (shown in Tilt UI sidebar)
k8s_resource('postgres', labels=['database'])
k8s_resource('redis', labels=['database'])
k8s_resource('api', labels=['backend'])

# Selectively run groups via tilt_config.json or CLI args
config.define_string_list('to-run', args=True)
cfg = config.parse()
groups = cfg.get('to-run', ['backend'])
```

## Trigger Modes

| Mode | Behavior |
|------|----------|
| `TRIGGER_MODE_AUTO` (default) | Updates on file change or dependency change |
| `TRIGGER_MODE_MANUAL` | Only updates via UI button or `tilt trigger <name>` |

Combine `auto_init=False` with `TRIGGER_MODE_MANUAL` for resources that should never run automatically:
```python
local_resource('db-migrate',
  cmd='python manage.py migrate',
  resource_deps=['database'],
  auto_init=False,              # Don't run on tilt up
  trigger_mode=TRIGGER_MODE_MANUAL,  # Only via tilt trigger
)
```

## Safety

```python
# ALWAYS set this for remote clusters - prevents accidental prod deploys
allow_k8s_contexts('my-dev-cluster')

# Without it, Tilt refuses to run against non-local contexts
```

## Lifecycle Hooks

```python
# Run cleanup on tilt down
if config.tilt_subcommand == 'down':
    local('helm uninstall myrelease || true')
    local('kubectl delete namespace dev-ns --ignore-not-found || true')
    local('rm -f .session-id')

# Set env vars for Docker Compose interpolation
os.putenv('MY_VAR', 'value')
```

## CLI Discovery

**Do not memorize CLI flags.** Tilt's CLI changes across versions. Always discover at runtime:

```bash
tilt --help                    # List all subcommands
tilt <command> --help          # Flags for a specific command
tilt api-resources             # List all API resource types + shortnames
tilt version                   # Current version
```

### Key Commands to Know Exist

These are the commands worth discovering flags for. Run `tilt <cmd> --help` before using:

| Command | What it does |
|---------|-------------|
| `tilt up` | Start environment. Tiltfile args go after `--`. Has flags for host, port, streaming, context override. |
| `tilt down` | Tear down. Has flags for deleting namespaces and volumes. |
| `tilt ci` | CI/batch mode — exits on success or failure. Has timeout flag. |
| `tilt trigger <resource>` | Trigger manual resource or force rebuild. |
| `tilt enable` / `tilt disable` | Enable/disable resources by name, label, or `--all`. `enable --only` disables everything else. |
| `tilt args` | Change Tiltfile args on a running Tilt instance (re-evaluates Tiltfile). |
| `tilt logs` | View logs. Has flags for follow, since-duration, source (build/runtime), JSON output. |
| `tilt get <resource-type>` | kubectl-style query of Tilt internal state. Supports `-o json/yaml/wide`, `-w` watch, label selectors. |
| `tilt describe <type>/<name>` | Full detail on a resource. |
| `tilt wait` | Block until a condition is met — useful in scripts: `tilt wait --for=condition=Ready uiresource/<name>`. |
| `tilt doctor` | Print diagnostic info (for debugging Tilt itself). |

### Tilt API Resources

Tilt exposes internal state as API resources (like kubectl). Run `tilt api-resources` for the full list with shortnames. Common ones:

```bash
tilt get uiresources                    # All resources + status
tilt get uiresources -o json            # JSON for scripting
tilt describe uiresource/<name>         # Deep-dive on one resource
tilt get pf                             # Active port forwards (shortname)
tilt get dc                             # Docker Compose services (shortname)
tilt get fw                             # File watchers
tilt get liveupdates                    # Live update state
```

## Common Patterns

### Helm + Image Build Wiring

```python
docker_build('registry/myapp', '.')
helm_resource('myapp', './charts/myapp',
  image_deps=['registry/myapp'],
  image_keys=[('image.repository', 'image.tag')],
  flags=['--set', 'replicas=1'],
  resource_deps=['database'],
  pod_readiness='wait',
)
```

### Docker Compose with Overrides

```python
docker_compose(['docker-compose.yml', 'docker-compose.dev.yml'],
  env_file='.env.dev',
  project_name='myproject'
)
dc_resource('db', new_name='postgres', labels=['infra'])
dc_resource('app', labels=['backend'],
  resource_deps=['postgres'],
  links=[link('http://localhost:8080', 'App UI')]
)
```

### Long-Running Local Process with Health Check

```python
local_resource('port-forward',
  serve_cmd='while true; do kubectl port-forward svc/app 8080:80; sleep 2; done',
  readiness_probe=probe(
    http_get=http_get_action(port=8080, path='/health/'),
    initial_delay_secs=15,
  ),
  resource_deps=['app'],
  links=[link('http://localhost:8080', 'App')],
)
```

### Conditional Resources

```python
config.define_bool('enable-monitoring')
cfg = config.parse()
if cfg.get('enable-monitoring', False):
    k8s_yaml('monitoring.yaml')
    k8s_resource('prometheus', labels=['monitoring'])
```

### Session Persistence (Reuse Across Restarts)

```python
session_file = '.session/current'
existing = str(local('cat {} 2>/dev/null || true'.format(session_file), quiet=True)).strip()
if existing:
    session_id = existing
else:
    session_id = str(local('cat /dev/urandom | LC_ALL=C tr -dc a-z0-9 | head -c5', quiet=True)).strip()
    local("mkdir -p .session && echo '{}' > {}".format(session_id, session_file))

# Clean up on tilt down
if config.tilt_subcommand == 'down':
    local('rm -f {}'.format(session_file))
```

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Forgot `allow_k8s_contexts` for remote cluster | Tilt refuses to start. Add it before any `local()` calls. |
| `local_resource` runs serially even with no deps | Set `allow_parallel=True` |
| Live update doesn't trigger | Changed file must match a `sync()` path within the build context. Check `ignore`/`only` rules. |
| `run(trigger=)` never fires | Trigger files must also be covered by a `sync()` step. |
| Helm values not updating image | Use `image_deps` + `image_keys` on `helm_resource`. `image_deps` takes the image ref string, not a return value. |
| `local()` blocks Tiltfile load | Use `local_resource()` for slow/long-running commands. |
| Resources rebuild on every `tilt up` | Add `ignore`/`only` patterns; check for non-deterministic `local()` calls at load time. |
| `tilt down` doesn't clean up `helm()` resources | `helm()` is template-only. Use `helm_resource()` for release lifecycle, or add cleanup in `config.tilt_subcommand == 'down'` block. |
| `restart_container()` fails on K8s | It only works with Docker Compose. For K8s, use the `restart_process` extension or a file-watching entrypoint. |
| Tiltfile doesn't re-execute when config file changes | `local()` doesn't track file reads. Use `read_file()`/`read_json()` or `watch_file()` to register dependencies. |
