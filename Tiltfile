# --- config you edit ---
NS = 'testkube'                       # target namespace
LOCAL_DIR = './sync_probe.txt'     # path on your laptop (repo-relative)
REMOTE_DIR = '/data/repo/sync_probe.txt'      # exact path in the pod/PVC
# ------------------------

# 0) Create the PVC once (idempotent)
cmd_pvc = """
bash -lc 'kubectl -n {ns} get pvc tilt-dev-pvc >/dev/null 2>&1 || kubectl apply -f k8s/pvc.yaml'
""".format(ns=NS)
local_resource('pvc-tilt-dev', cmd_pvc, trigger_mode=TRIGGER_MODE_AUTO)

# 1) Deploy syncer (depends on PVC)
k8s_resource('tilt-syncer', resource_deps=['pvc-tilt-dev'])

# 2) CLEAN: wipe the directory on the PVC after syncer is available
cmd_clean = """
set -euo pipefail;
kubectl -n {ns} wait --for=condition=available --timeout=60s deploy/tilt-syncer;

# Retry loop to gracefully handle the race condition where the pod is not
# immediately available after the deployment is ready.
for i in {{1..5}}; do
  POD=$(kubectl -n {ns} get pod -l app=tilt-syncer -o jsonpath="{{.items[0].metadata.name}}" 2>/dev/null || true);
  if [ -n "$POD" ]; then
    echo "Found pod: $POD";
    break;
  fi;
  echo "Waiting for pod to be discoverable... (attempt $i)";
  sleep 2;
done;

if [ -z "$POD" ]; then
  echo "ERROR: Could not find a running pod for app=tilt-syncer after multiple attempts." >&2;
  exit 1;
fi;

# If REMOTE_DST is a directory, wipe its contents; if it's a file, remove the file.
kubectl -n {ns} exec "$POD" -- sh -c '
  DST="{dst}";
  if [ -d "$DST" ]; then
    rm -rf "$DST"/* 2>/dev/null || true
  else
    rm -f "$DST" 2>/dev/null || true
    mkdir -p "$(dirname "$DST")"
  fi
'
""".format(ns=NS, dst=REMOTE_DIR)

local_resource(
    'pvc-clean',
    cmd_clean,
    trigger_mode=TRIGGER_MODE_AUTO,
    resource_deps=['tilt-syncer'],
)

# 3) SEED: copy local dir into PVC (runs AFTER clean)
cmd_seed = """
set -euo pipefail;
# Retry loop to find the pod
for i in {{1..5}}; do
  POD=$(kubectl -n {ns} get pod -l app=tilt-syncer -o jsonpath="{{.items[0].metadata.name}}" 2>/dev/null || true);
  if [ -n "$POD" ]; then
    echo "Found pod: $POD";
    break;
  fi;
  echo "Waiting for pod to be discoverable... (attempt $i)";
  sleep 2;
done;

if [ -z "$POD" ]; then
  echo "ERROR: Could not find a running pod for app=tilt-syncer after multiple attempts." >&2;
  exit 1;
fi;

# Ensure destination exists appropriately, then copy file or directory.
kubectl -n {ns} exec "$POD" -- sh -c '
  DST="{dst}";
  if [ -d "$DST" ];
  then
    true  # dir exists; fine
  else
    mkdir -p "$(dirname "$DST")"
  fi
'
# Decide cp mode based on local source type
if [ -d "{src}" ];
then
  kubectl -n {ns} cp "{src}/." "$POD:{dst}" -c syncer
else
  kubectl -n {ns} cp "{src}"     "$POD:{dst}" -c syncer
fi
""".format(ns=NS, src=LOCAL_DIR, dst=REMOTE_DIR)

local_resource(
    'seed-dir-into-pvc',
    cmd_seed,
    trigger_mode=TRIGGER_MODE_AUTO,
    resource_deps=['pvc-clean'],   # ensure clean runs first
)

# Build a tiny pod Tilt owns, so Live Update can copy files into it
docker_build(
    'tilt-syncer:dev',
    '.',
    dockerfile='Dockerfile.tilt-syncer',
    live_update=[
        # single-file sync: change locally -> auto-copied into the container
        sync(LOCAL_DIR, REMOTE_DIR),
    ],
)

# Deploy only the Deployment (PVC is NOT Tilt-managed)
k8s_yaml('k8s/syncer.yaml')
k8s_resource('tilt-syncer', resource_deps=['pvc-tilt-dev'])