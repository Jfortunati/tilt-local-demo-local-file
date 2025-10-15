# Tilt File Sync to Persistent Volume Demo 🔄

This project is a simple demonstration of how to use **Tilt** to live-sync a local file into a Kubernetes pod that is backed by a **PersistentVolumeClaim (PVC)**.

This pattern is useful for development environments where you need to rapidly iterate on code or configuration files that must exist on a persistent volume, without rebuilding a container image for every change.

## Prerequisites

Before you begin, make sure you have the following tools installed and running:

* **Tilt**: For orchestrating the development environment.
* **A local Kubernetes cluster**: Such as [minikube](https://minikube.sigs.k8s.io/docs/start/), Docker Desktop, or k3d.
* **Docker**: Or another container runtime compatible with your Kubernetes cluster.
* **kubectl**: The Kubernetes command-line tool.

---

## How to Run the Demo

1.  **Clone the repository:**
    ```bash
    git clone [https://github.com/Jfortunati/tilt-local-demo-local-file.git](https://github.com/Jfortunati/tilt-local-demo-local-file.git)
    cd tilt-local-demo-local-file
    ```

2.  **Start your Kubernetes cluster:**
    If you're using minikube, simply run:
    ```bash
    minikube start
    ```

3.  **Launch Tilt:**
    Run `tilt up` from the root of the project directory. Tilt will open in your web browser and begin setting up the resources.
    ```bash
    tilt up
    ```

---

## Verifying the Sync

Once `tilt up` is running and all resources in the UI are green, you can test the file synchronization.

1.  **Check the initial file contents:**
    The `Tiltfile` first copies the local `sync_probe.txt` into the pod. You can verify its contents with `kubectl exec`:
    ```bash
    kubectl exec -n testkube -it deploy/tilt-syncer -- cat /data/repo/sync_probe.txt
    ```

2.  **Modify the local file:**
    Open the local `sync_probe.txt` file in your text editor, change the text, and save it.

3.  **Verify the change in the pod:**
    Almost instantly, Tilt will sync the change. Run the `kubectl exec` command again, and you'll see the updated content inside the pod. This happens **without a pod restart**.

4.  **Test for Persistence:**
    To prove the file is on the persistent volume, delete the pod directly. The Kubernetes Deployment will automatically create a new one to replace it.
    ```bash
    # Delete the running pod
    kubectl delete pod -l app=tilt-syncer -n testkube
    
    # Wait a few seconds for the new pod to start, then check the file again
    kubectl exec -n testkube -it deploy/tilt-syncer -- cat /data/repo/sync_probe.txt
    ```
    The file and its latest content will still be there, proving it was persisted on the volume. ✨

---

## How It Works

* `Tiltfile`: This is the main script that orchestrates the entire process. It creates the PVC, builds the Docker image, deploys the resources to Kubernetes, and sets up the live sync.
* `k8s/pvc.yaml`: A manifest that defines the `PersistentVolumeClaim` to provide stable storage.
* `k8s/syncer.yaml`: A Kubernetes `Deployment` that runs a simple pod. This pod mounts the PVC at the `/data/repo` path.
* `Dockerfile.tilt-syncer`: A minimal Dockerfile that creates the `/data/repo` directory where the volume will be mounted.
* `sync_probe.txt`: The sample file on your local machine that gets synced into the pod.

---

## Cleanup

To stop the live-sync and remove the Kubernetes resources created by Tilt, press `(Ctrl + C)` in the terminal where `tilt up` is running, or run:

```bash
tilt down
