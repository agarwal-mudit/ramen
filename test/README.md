<!--
SPDX-FileCopyrightText: The RamenDR authors
SPDX-License-Identifier: Apache-2.0
-->

# Ramen test environment

This directory provides tools and configuration for creating Ramen test
environment.

## Setup

1. Add yourself to the libvirt group (required for minikube kvm2 driver).

   ```
   sudo usermod -a -G libvirt $(whoami)
   ```

   Logout and login again for the change above to be in effect.

1. Install minikube, for example on RHEL/CentOS/Fedora:

   ```
   sudo dnf install https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
   ```

   You need `minikube` version supporting the `--extra-disks` option.
   The tool was tested with `minikube` v1.26.1.

1. Install the kubectl tool. See
   [Install and Set Up kubectl on Linux](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

1. Install `clusteradm` tool. See
   [Open Cluster Management Quick Start guide](https://open-cluster-management.io/getting-started/quick-start/#install-clusteradm-cli-tool)
   for the details. Version 0.5.0 or later is required.

1. Install `podman`

   ```
   sudo dnf install podman
   ```

   Allow yourself to run podman as root without a password by adding
   this file as root:

   ```
   # cat /etc/sudoers.d/podman
   # Allow passwordless podman
   myusername ALL=(ALL) NOPASSWD: /usr/bin/podman
   ```

1. Install the `drenv` package in a virtual environment:

   Run this once in the root of the source tree:

   ```
   python3 -m venv ~/.venv/ramen
   source ~/.venv/ramen/bin/activate
   pip install --upgrade pip
   pip install -e ./test
   ```

   You can create the virtual environment anywhere, but keeping the
   environment outside of the source tree is good practice.

   This installs a link file in the virtual environment:

   ```
   $ cat /home/nsoffer/.venv/ramen/lib/python3.10/site-packages/drenv.egg-link
   /home/nsoffer/src/ramen/test
   ```

   So changes in the `test/drenv` package are available immediately
   without installing the package again.

## Using the drenv tool

Before running the `drenv` tool you need to activate the virtual
environment:

```
source ~/.venv/ramen/bin/activate
```

The shell prompt will change to reflect that the `ramen` virtual
environment is active:

```
(ramen) [user@host test]$
```

Change directory to the test directory where the environment yamls and
scripts are:

```
cd test
```

To start the environment:

```
drenv start example.yaml
```

To stop the environment:

```
drenv stop example.yaml
```

To delete the environment:

```
drenv delete example.yaml
```

To inspect a processed environment file:

```
drenv dump example.yaml
```

Dumping the file shows how drenv binds templates, expands scripts
arguments, name workers, and applies default values. This can be useful
to debugging drenv or when writing a new environment file.

Useful options:

- `-v`, `--verbose`: Show verbose logs
- `-h`, `--help`: Show online help
- `--name-prefix`: Add prefix to profiles names

When you are done you can deactivate the virtual environment:

```
deactivate
```

## The environment file

To create an environment you need an yaml file describing the
clusters and how to deploy them.

### Example environment file

```
name: example
templates:
  - name: "example-cluster"
    driver: podman
    container_runtime: cri-o
    workers:
      - scripts:
          - name: example
profiles:
  - name: ex1
    template: example-cluster
  - name: ex2
    template: example-cluster
```

### Experimenting with the example environment

You can play with the example environment to understand how the `drenv`
tool works and how to write scripts.

#### Starting the example environment

Starting the environment create 2 minikube clusters, deploy example
deployment on every clusters, and finally run a self test verifying that
the deployment is available on both clusters.

```
$ drenv start example.yaml
2023-01-03 23:20:17,822 INFO    [example] Starting environment
2023-01-03 23:20:17,823 INFO    [ex1] Starting cluster
2023-01-03 23:20:18,824 INFO    [ex2] Starting cluster
2023-01-03 23:20:41,037 INFO    [ex1] Cluster started in 23.21 seconds
2023-01-03 23:20:41,038 INFO    [ex1/0] Running example/start
2023-01-03 23:20:41,200 INFO    [ex1/0] example/start completed in 0.16 seconds
2023-01-03 23:20:41,200 INFO    [ex1/0] Running example/test
2023-01-03 23:20:42,212 INFO    [ex2] Cluster started in 23.39 seconds
2023-01-03 23:20:42,212 INFO    [ex2/0] Running example/start
2023-01-03 23:20:42,387 INFO    [ex2/0] example/start completed in 0.17 seconds
2023-01-03 23:20:42,387 INFO    [ex2/0] Running example/test
2023-01-03 23:20:59,249 INFO    [ex1/0] example/test completed in 18.05 seconds
2023-01-03 23:21:01,474 INFO    [ex2/0] example/test completed in 19.09 seconds
2023-01-03 23:21:01,474 INFO    [example] Environment started in 43.65 seconds
```

#### Inspecting the clusters with minikube

We can use minikube to inspect or access the clusters:

```
$ minikube profile list
|---------|-----------|---------|--------------|------|---------|---------|-------|--------|
| Profile | VM Driver | Runtime |      IP      | Port | Version | Status  | Nodes | Active |
|---------|-----------|---------|--------------|------|---------|---------|-------|--------|
| ex1     | podman    | crio    | 192.168.49.2 | 8443 | v1.25.3 | Running |     1 |        |
| ex2     | podman    | crio    | 10.88.0.166  | 8443 | v1.25.3 | Running |     1 |        |
|---------|-----------|---------|--------------|------|---------|---------|-------|--------|
```

#### Inspecting the clusters with kubectl

We can use kubectl to access the clusters:

```
$ kubectl logs deploy/example-deployment --context ex1
Tue Jan  3 21:20:58 UTC 2023
Tue Jan  3 21:21:08 UTC 2023
Tue Jan  3 21:21:18 UTC 2023

$ kubectl logs deploy/example-deployment --context ex2
Tue Jan  3 21:21:00 UTC 2023
Tue Jan  3 21:21:10 UTC 2023
Tue Jan  3 21:21:20 UTC 2023
```

#### Isolating environments with --name-prefix

To run multiple instances of the same environment, or multiple
environments using the same profile names, use unique `--name-prefix`
for each run.

Start first instance:

```
$ drenv start --name-prefix test1- example.yaml
2023-01-03 23:35:38,328 INFO    [test1-example] Starting environment
2023-01-03 23:35:38,330 INFO    [test1-ex1] Starting cluster
2023-01-03 23:35:39,330 INFO    [test1-ex2] Starting cluster
2023-01-03 23:36:01,923 INFO    [test1-ex1] Cluster started in 23.59 seconds
2023-01-03 23:36:01,924 INFO    [test1-ex1/0] Running example/start
2023-01-03 23:36:02,153 INFO    [test1-ex1/0] example/start completed in 0.23 seconds
2023-01-03 23:36:02,153 INFO    [test1-ex1/0] Running example/test
2023-01-03 23:36:02,428 INFO    [test1-ex2] Cluster started in 23.10 seconds
2023-01-03 23:36:02,429 INFO    [test1-ex2/0] Running example/start
2023-01-03 23:36:02,608 INFO    [test1-ex2/0] example/start completed in 0.18 seconds
2023-01-03 23:36:02,608 INFO    [test1-ex2/0] Running example/test
2023-01-03 23:36:21,114 INFO    [test1-ex1/0] example/test completed in 18.96 seconds
2023-01-03 23:36:22,616 INFO    [test1-ex2/0] example/test completed in 20.01 seconds
2023-01-03 23:36:22,616 INFO    [test1-example] Environment started in 44.29 seconds
```

This creates:

```
$ minikube profile list
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
|  Profile  | VM Driver | Runtime |      IP      | Port | Version | Status  | Nodes | Active |
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
| test1-ex1 | podman    | crio    | 192.168.49.2 | 8443 | v1.25.3 | Running |     1 |        |
| test1-ex2 | podman    | crio    | 10.88.0.196  | 8443 | v1.25.3 | Running |     1 |        |
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
```

Start second instance:

```
$ drenv start --name-prefix test2- example.yaml
2023-01-03 23:36:44,181 INFO    [test2-example] Starting environment
2023-01-03 23:36:44,182 INFO    [test2-ex1] Starting cluster
2023-01-03 23:36:45,183 INFO    [test2-ex2] Starting cluster
2023-01-03 23:37:08,685 INFO    [test2-ex2] Cluster started in 23.50 seconds
2023-01-03 23:37:08,686 INFO    [test2-ex2/0] Running example/start
2023-01-03 23:37:08,901 INFO    [test2-ex2/0] example/start completed in 0.22 seconds
2023-01-03 23:37:08,901 INFO    [test2-ex2/0] Running example/test
2023-01-03 23:37:08,969 INFO    [test2-ex1] Cluster started in 24.79 seconds
2023-01-03 23:37:08,969 INFO    [test2-ex1/0] Running example/start
2023-01-03 23:37:09,132 INFO    [test2-ex1/0] example/start completed in 0.16 seconds
2023-01-03 23:37:09,132 INFO    [test2-ex1/0] Running example/test
2023-01-03 23:37:26,811 INFO    [test2-ex2/0] example/test completed in 17.91 seconds
2023-01-03 23:37:27,119 INFO    [test2-ex1/0] example/test completed in 17.99 seconds
2023-01-03 23:37:27,119 INFO    [test2-example] Environment started in 42.94 seconds
```

This adds new profiles:

```
$ minikube profile list
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
|  Profile  | VM Driver | Runtime |      IP      | Port | Version | Status  | Nodes | Active |
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
| test1-ex1 | podman    | crio    | 192.168.49.2 | 8443 | v1.25.3 | Running |     1 |        |
| test1-ex2 | podman    | crio    | 10.88.0.196  | 8443 | v1.25.3 | Running |     1 |        |
| test2-ex1 | podman    | crio    | 192.168.58.2 | 8443 | v1.25.3 | Running |     1 |        |
| test2-ex2 | podman    | crio    | 10.88.0.201  | 8443 | v1.25.3 | Running |     1 |        |
|-----------|-----------|---------|--------------|------|---------|---------|-------|--------|
```

You must use the same `--name-prefix` when stopping or deleting the
environments.

#### Running scripts manually

When debugging scripts it is useful to run them manually:

```
$ example/start ex1
* Deploying example
  deployment.apps/example-deployment unchanged

$ example/test ex1
* Testing example deployment
  deployment "example-deployment" successfully rolled out
```

#### Starting a started environment

If something failed while starting, or we change the scripts, we can run
start again. This can be faster then creating the environment from
scratch.

```
$ drenv start example.yaml
2023-01-03 23:40:25,451 INFO    [example] Starting environment
2023-01-03 23:40:25,452 INFO    [ex1] Starting cluster
2023-01-03 23:40:26,453 INFO    [ex2] Starting cluster
2023-01-03 23:40:29,972 INFO    [ex1] Cluster started in 4.52 seconds
2023-01-03 23:40:29,972 INFO    [ex1] Waiting until all deployments are available
2023-01-03 23:40:30,658 INFO    [ex2] Cluster started in 4.20 seconds
2023-01-03 23:40:30,658 INFO    [ex2] Waiting until all deployments are available
2023-01-03 23:41:00,224 INFO    [ex1] Deployments are available in 30.25 seconds
2023-01-03 23:41:00,225 INFO    [ex1/0] Running example/start
2023-01-03 23:41:00,381 INFO    [ex1/0] example/start completed in 0.16 seconds
2023-01-03 23:41:00,381 INFO    [ex1/0] Running example/test
2023-01-03 23:41:00,467 INFO    [ex1/0] example/test completed in 0.09 seconds
2023-01-03 23:41:00,925 INFO    [ex2] Deployments are available in 30.27 seconds
2023-01-03 23:41:00,925 INFO    [ex2/0] Running example/start
2023-01-03 23:41:01,080 INFO    [ex2/0] example/start completed in 0.15 seconds
2023-01-03 23:41:01,080 INFO    [ex2/0] Running example/test
2023-01-03 23:41:01,166 INFO    [ex2/0] example/test completed in 0.09 seconds
2023-01-03 23:41:01,166 INFO    [example] Environment started in 35.71 seconds
```

#### Using --verbose option

While debugging it is useful to use the `--verbose` option to see much
more details:

```
$ drenv start example.yaml -v
2023-01-03 23:41:53,414 INFO    [example] Starting environment
2023-01-03 23:41:53,416 INFO    [ex1] Starting cluster
2023-01-03 23:41:53,539 DEBUG   [ex1] * [ex1] minikube v1.28.0 on Fedora 37
2023-01-03 23:41:53,540 DEBUG   [ex1]   - MINIKUBE_HOME=/data/minikube
2023-01-03 23:41:53,582 DEBUG   [ex1] * Using the podman driver based on user configuration
2023-01-03 23:41:53,664 DEBUG   [ex1] * Using Podman driver with root privileges
2023-01-03 23:41:53,666 DEBUG   [ex1] * Starting control plane node ex1 in cluster ex1
2023-01-03 23:41:53,669 DEBUG   [ex1] * Pulling base image ...
2023-01-03 23:41:53,672 DEBUG   [ex1] * Creating podman container (CPUs=2, Memory=4096MB) ...
2023-01-03 23:41:54,416 INFO    [ex2] Starting cluster
2023-01-03 23:41:54,614 DEBUG   [ex2] * [ex2] minikube v1.28.0 on Fedora 37
2023-01-03 23:41:54,617 DEBUG   [ex2]   - MINIKUBE_HOME=/data/minikube
2023-01-03 23:41:54,665 DEBUG   [ex2] * Using the podman driver based on user configuration
2023-01-03 23:41:54,768 DEBUG   [ex2] * Using Podman driver with root privileges
2023-01-03 23:41:54,771 DEBUG   [ex2] * Starting control plane node ex2 in cluster ex2
2023-01-03 23:41:54,774 DEBUG   [ex2] * Pulling base image ...
2023-01-03 23:41:54,777 DEBUG   [ex2] * Creating podman container (CPUs=2, Memory=4096MB) ...
2023-01-03 23:42:00,763 DEBUG   [ex1] * Preparing Kubernetes v1.25.3 on CRI-O 1.24.3 ...
2023-01-03 23:42:01,814 DEBUG   [ex1]   - Generating certificates and keys ...
2023-01-03 23:42:01,921 DEBUG   [ex2] * Preparing Kubernetes v1.25.3 on CRI-O 1.24.3 ...
2023-01-03 23:42:02,808 DEBUG   [ex2]   - Generating certificates and keys ...
2023-01-03 23:42:03,656 DEBUG   [ex1]   - Booting up control plane ...
2023-01-03 23:42:05,617 DEBUG   [ex2]   - Booting up control plane ...
2023-01-03 23:42:13,684 DEBUG   [ex1]   - Configuring RBAC rules ...
2023-01-03 23:42:14,095 DEBUG   [ex1] * Configuring CNI (Container Networking Interface) ...
2023-01-03 23:42:15,219 DEBUG   [ex1] * Verifying Kubernetes components...
2023-01-03 23:42:15,380 DEBUG   [ex1]   - Using image gcr.io/k8s-minikube/storage-provisioner:v5
2023-01-03 23:42:15,653 DEBUG   [ex2]   - Configuring RBAC rules ...
2023-01-03 23:42:15,752 DEBUG   [ex1] * Enabled addons: storage-provisioner, default-storageclass
2023-01-03 23:42:15,797 DEBUG   [ex1] * Done! kubectl is now configured to use "ex1" cluster and "default" namespace by default
2023-01-03 23:42:15,809 INFO    [ex1] Cluster started in 22.39 seconds
2023-01-03 23:42:15,809 INFO    [ex1/0] Running example/start
2023-01-03 23:42:15,843 DEBUG   [ex1/0] * Deploying example
2023-01-03 23:42:15,984 DEBUG   [ex1/0]   deployment.apps/example-deployment created
2023-01-03 23:42:15,992 INFO    [ex1/0] example/start completed in 0.18 seconds
2023-01-03 23:42:15,992 INFO    [ex1/0] Running example/test
2023-01-03 23:42:16,026 DEBUG   [ex1/0] * Testing example deployment
2023-01-03 23:42:16,067 DEBUG   [ex2] * Configuring CNI (Container Networking Interface) ...
2023-01-03 23:42:16,083 DEBUG   [ex1/0]   Waiting for deployment spec update to be observed...
2023-01-03 23:42:17,161 DEBUG   [ex2] * Verifying Kubernetes components...
2023-01-03 23:42:17,216 DEBUG   [ex2]   - Using image gcr.io/k8s-minikube/storage-provisioner:v5
2023-01-03 23:42:17,626 DEBUG   [ex2] * Enabled addons: storage-provisioner, default-storageclass
2023-01-03 23:42:17,675 DEBUG   [ex2] * Done! kubectl is now configured to use "ex2" cluster and "default" namespace by default
2023-01-03 23:42:17,688 INFO    [ex2] Cluster started in 23.27 seconds
2023-01-03 23:42:17,688 INFO    [ex2/0] Running example/start
2023-01-03 23:42:17,721 DEBUG   [ex2/0] * Deploying example
2023-01-03 23:42:17,858 DEBUG   [ex2/0]   deployment.apps/example-deployment created
2023-01-03 23:42:17,866 INFO    [ex2/0] example/start completed in 0.18 seconds
2023-01-03 23:42:17,866 INFO    [ex2/0] Running example/test
2023-01-03 23:42:17,900 DEBUG   [ex2/0] * Testing example deployment
2023-01-03 23:42:17,954 DEBUG   [ex2/0]   Waiting for deployment spec update to be observed...
2023-01-03 23:42:27,903 DEBUG   [ex1/0]   Waiting for deployment spec update to be observed...
2023-01-03 23:42:27,909 DEBUG   [ex1/0]   Waiting for deployment "example-deployment" rollout to finish: 0 out of 1 new replicas have been updated...
2023-01-03 23:42:28,021 DEBUG   [ex1/0]   Waiting for deployment "example-deployment" rollout to finish: 0 of 1 updated replicas are available...
2023-01-03 23:42:28,992 DEBUG   [ex2/0]   Waiting for deployment spec update to be observed...
2023-01-03 23:42:28,997 DEBUG   [ex2/0]   Waiting for deployment "example-deployment" rollout to finish: 0 out of 1 new replicas have been updated...
2023-01-03 23:42:29,046 DEBUG   [ex2/0]   Waiting for deployment "example-deployment" rollout to finish: 0 of 1 updated replicas are available...
2023-01-03 23:42:34,960 DEBUG   [ex1/0]   deployment "example-deployment" successfully rolled out
2023-01-03 23:42:34,967 INFO    [ex1/0] example/test completed in 18.98 seconds
2023-01-03 23:42:35,980 DEBUG   [ex2/0]   deployment "example-deployment" successfully rolled out
2023-01-03 23:42:35,987 INFO    [ex2/0] example/test completed in 18.12 seconds
2023-01-03 23:42:35,987 INFO    [example] Environment started in 42.57 seconds
```

#### Stopping the environment

We can stop the environment, for example if we need to reboot the host,
or don't have enough resources to run multiple environment at the same
time.

```
$ drenv stop example.yaml
2023-01-03 23:43:09,169 INFO    [example] Stopping environment
2023-01-03 23:43:09,171 INFO    [ex1] Stopping cluster
2023-01-03 23:43:09,172 INFO    [ex2] Stopping cluster
2023-01-03 23:43:13,829 INFO    [ex1] Cluster stopped in 4.66 seconds
2023-01-03 23:43:14,032 INFO    [ex2] Cluster stopped in 4.86 seconds
2023-01-03 23:43:14,033 INFO    [example] Environment stopped in 4.86 seconds
```

We can start the environment later. This can be faster than recreating
it from scratch.

#### Deleting the environment

To delete the environment including the VM disks and dropping all
changes made to the environment:

```
$ drenv delete example.yaml
2023-01-03 23:43:36,601 INFO    [example] Deleting environment
2023-01-03 23:43:36,602 INFO    [ex1] Deleting cluster
2023-01-03 23:43:36,603 INFO    [ex2] Deleting cluster
2023-01-03 23:43:43,645 INFO    [ex2] Cluster deleted in 7.04 seconds
2023-01-03 23:43:43,897 INFO    [ex1] Cluster deleted in 7.29 seconds
2023-01-03 23:43:43,897 INFO    [example] Environment deleted in 7.30 seconds
```

### The environment file format

- `templates`: templates for creating new profiles.
    - `name`: profile name.
    - `driver`: The minikube driver. Tested with "kvm2" and "podman"
      (default "kvm2")
    - `container_runtime`: The container runtime to be used. Valid
      options: "docker", "cri-o", "containerd" (default: "containerd")
    - `network`: The network to run minikube with. If left empty,
      minikube will create a new isolated network.
    - `extra_disks`: Number of extra disks (default 0)
    - `disk_size`: Disk size string (default "50g")
    - `nodes`: Number of cluster nodes (default 1)
    - `cni`: Network plugin (default "auto")
    - `cpus`: Number of CPUs per VM (default 2)
    - `memory`: Memory per VM (default 4g)
    - `addons`: List of minikube addons to install
    - `workers`: Optional list of workers to run when starting a
      profile. Use multiple workers to run scripts in parallel.
        - `name`: Optional worker name
        - `scripts`: Scripts to run by this worker.
            - `name`: Scripts directory
            - `args`: Optional argument to the script. If not specified the
              script is run with one argument, the profile name.

- `profiles`: List of profile managed by the environment. Any template
   key is valid in the profile, overriding the same key from the template.
    - `template`: The template to create this profile from.

- `workers`: Optional list of workers for running scripts after all
  profile are started.
    - `name`: Optional worker name
    - `scripts`: Scripts to run by this worker
        - `name`: Scripts directory
        - `args`: Optional argument to the script. If not specified the
          script is run without any arguments.

#### Scripts hooks

The script direcotry may contain scripts to be run on certain events,
based on the hook file name.

| Event        | Scripts       | Comment                             |
|--------------|---------------|-------------------------------------|
| start        | start, test   | after cluster was started           |
| stop         | stop          | before cluster is stopped           |
| delete       | -             |                                     |

The `start` and `test` scripts are not allowed to fail. If a script
fail, execution stops and the entire command will fail.

The `stop` script is allowed to fail. The failure is logged but the
`stop` command will not fail.

#### Script arguments

When specifying script `args`, you can use the special variable `$name`.
This will be replaced with the profile name.

Example yaml:

```
profiles:
  - name: cluster1
    workers:
      - scripts:
          - name: script
            args: [$name, arg2]
```

The `drenv` tool will run the script hooks as:

```
script/start cluster1 arg2
script/test cluster1 arg2
```

## The regional-dr environment

This is a configuration for testing regional DR using a hub cluster and
2 managed clusters.

## Testing drenv

### Installing development tools

```
pip install -r requirements.txt
```

### Enabling full test coverage

To enable test coverage for child processes, copy the `coverage.pth`
file to the python installation:

```
cp coverage.pth ~/.venv/ramen/lib/python*/site-packages
```

### Running the tests

Run all linters and tests and report test coverage:

```
make
```

Create an html report and open the report in a browser:

```
make coverage-html
```

Checking that code is formatted according to project style:

```
make black
```

Reformatting code to be compatible with project style:

```
make black-reformat
```
