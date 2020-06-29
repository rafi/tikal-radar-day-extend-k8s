# Extend Kubernetes with Go & Operators

## Introduction

Kubernetes is proving great abilities in orchestrating container clusters over
large number of nodes with abstract definitions for resources. It allows you to
define and manage intricate infrastructure as code.

The goal of the workshop is learning how to natively extend Kubernetes by
creating a new custom resource and introducing new abilities to your Kubernetes
cluster.

## What You Will Build

In the workshop we will create a new Custom Resource Definition (CRD) and define
its spec. Together with the operator-sdk and Golang we'll write a Kubernetes
controller that watches cluster resource events and create new Job resources
that will run Ansible playbooks dynamically when a new custom resource is
created.

## What You Will Learn

* How to bootstrap a new project using the operator-sdk tool
* How to create and spec a new Kubernetes CRD
* How to use operator-sdk’s development server
* Write a Kubernetes controller that creates new Jobs when new CR's are created

## Prerequisites

* Experience with Minikube/Kubernetes
* Basic Golang knowledge

1. git
1. go version v1.12+
1. mercurial version 3.9+
1. docker version 17.03+
1. kubectl version v1.11.3+
1. Access to a Kubernetes v1.11.3+ cluster

Ensure you have Minikube and Go installed on your laptop.

### Setup

I've tried to summarize installation here:

### macOS Setup

For macOS make sure to install and upgrade packages:

```bash
brew install go kubernetes-cli minikube
minikube start
minikube stop
```

### Ubuntu Setup

For Ubuntu, first install Go:

```bash
sudo add-apt-repository ppa:longsleep/golang-backports
sudo apt-get update
sudo apt-get install golang-go
```

Then install Minikube:

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
chmod +x minikube
sudo cp minikube /usr/local/bin && rm minikube
```

### Windows Setup

For Windows, first install latest Golang from [here](https://golang.org/dl/).
Then install Minikube:

```bash
choco install minikube
```

[Minikube]: https://kubernetes.io/docs/tasks/tools/install-minikube/

## Getting Started: Operator-SDK

The [Operator SDK] has a CLI tool that helps the developer to create, build,
and deploy a new operator project.

Install the SDK CLI tool:

```bash
$ brew install operator-sdk
```

### (1) Operator-SDK: Initialize new project

Use the CLI to create a new ansible-operator project:
(🚨 Change ‘`--repo`’ to your Git repository)

```bash
$ mkdir $HOME/projects
$ cd $HOME/projects
$ operator-sdk new ansible-operator --repo=github.com/tikalk/ansible-operator
$ cd ansible-operator
```

To learn about the project directory structure, see [project layout] doc.

### (2) Operator-SDK: Add new Custom Resource Definition

Add a new Custom Resource Definition(CRD) API called AnsibleJob, with APIVersion
`k8s.tikalk.com/v1alpha1` and Kind `AnsibleJob`.

```bash
$ operator-sdk add api --api-version=k8s.tikalk.com/v1alpha1 --kind=AnsibleJob
```

This will scaffold the `AnsibleJob` resource API under
`pkg/apis/cache/v1alpha1/...`.

```bash
$ git init
$ git status
$ git add .
$ git commit -m 'Introduce new CRD'
```

### (3) Operator-SDK: Define the spec and status

Modify the spec and status of the `AnsibleJob` CR at
`pkg/apis/k8s/v1alpha1/ansiblejob_types.go`:

```go
type AnsibleJobSpec struct {
	Repo      string `json:"repo,required"`
	Playbook  string `json:"playbook,required"`
	Inventory string `json:"inventory,required"`
	Limit     string `json:"limit"`
	Forks     int32  `json:"forks"`
	Check     bool   `json:"check"`
}

type AnsibleJobStatus struct {
	Status      string `json:"status"`
	BuildNumber int32  `json:"build_number"`
}
```

### (4) Operator-SDK: Generate after schema change

After modifying the `*_types.go` file always run the following command to
update the generated code for that resource type:

```bash
$ operator-sdk generate k8s
$ operator-sdk generate crds
$ git status
$ git commit -am 'Add spec'
```

### (5) Operator-SDK: Add a new controller

Add a new Controller to the project that will watch and reconcile the
`AnsibleJob` resource:

```bash
$ operator-sdk add controller --api-version=k8s.tikalk.com/v1alpha1 --kind=AnsibleJob
```

This will scaffold a new Controller implementation under
`pkg/controller/ansiblejob/...`.

```bash
$ git status
$ git add .
$ git commit -m 'Introduce controller'
```

## Run the Operator

Before running the operator, the CRD must be registered with the Kubernetes
apiserver:

```bash
$ kubectl create -f deploy/crds/k8s.tikalk.com_ansiblejobs_crd.yaml
```

Once this is done, there are two ways to run the operator:

* As a Deployment inside a Kubernetes cluster
* As Go program outside a cluster (⚡ preferred during development cycle!)

### Run Operator During Development

Run locally outside the cluster:

```bash
$ export OPERATOR_NAME=ansible-operator
$ operator-sdk run local --watch-namespace=default

INFO[0000] Running the operator locally; watching namespace "default"
{ "msg": "Operator Version: 0.0.1" }
{ "msg": "Go Version: go1.14.4" }
{ "msg": "Go OS/Arch: darwin/amd64" }
{ "msg": "Version of operator-sdk: v0.18.2" }
```

### Create New Custom Resource (CR)

Modify `deploy/crds/k8s.tikalk.com_v1alpha1_ansiblejob_cr.yaml`

```yaml
apiVersion: k8s.tikalk.com/v1alpha1
kind: AnsibleJob
metadata:
  name: playbook-debug
spec:
  repo: https://github.com/tikalk/ansible-operator-test.git
  playbook: playbooks/debug.yml
  inventory: ./inventory
  limit: local
  check: false
  forks: 1
```

Now let’s deploy it!

```bash
$ kubectl create -f deploy/crds/k8s.tikalk.com_v1alpha1_ansiblejob_cr.yaml
```

:warning: If your operator-sdk is running locally, you will notice it
reconciled the new state and created a new pod. STOP the operator-sdk server
now.

## Controller Implementation

Will work on `pkg/controller/ansiblejob/ansiblejob_controller.go` where
our controller lives.

### Controller: Watch Job

Add batchv1 "k8s.io/api/batch/v1" to imports:
```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go

 import (
        "context"
+       batchv1 "k8s.io/api/batch/v1"
 )
```

And watch Jobs (Not pods) which we will create:

```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go
@@ -54,7 +56,7 @@ func add(mgr manager.Manager, r reconcile.Reconciler) error {

        // TODO(user): Modify this to be the types you create that are owned by the primary resource
        // Watch for changes to secondary resource Pods and requeue the owner AnsibleJob
-       err = c.Watch(&source.Kind{Type: &corev1.Pod{}}, &handler.EnqueueRequestForOwner{
+       err = c.Watch(&source.Kind{Type: &batchv1.Job{}}, &handler.EnqueueRequestForOwner{
                IsController: true,
                OwnerType:    &k8sv1alpha1.AnsibleJob{},
        })
```

### Controller: Create new function

Let’s start changing the “Reconcile” logic which we’ll use to create our Ansible job:

```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go

-       // Define a new Pod object
-       pod := newPodForCR(instance)
+       // Define a new Job object
+       pod := newJobForCR(instance)
```

```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go

-// newPodForCR returns a busybox pod with the same name/namespace as the cr
-func newPodForCR(cr *k8sv1alpha1.AnsibleJob) *corev1.Pod {
+// newJobForCR returns a pod with the same name/namespace as the cr
+func newJobForCR(cr *k8sv1alpha1.AnsibleJob) *batchv1.Job {
```

Now replace the whole function `newJobForCR` with:

```go
// newJobForCR returns a pod with the same name/namespace as the cr
func newJobForCR(cr *k8sv1alpha1.AnsibleJob) *batchv1.Job {
	labels := map[string]string{
		"app": cr.Name,
	}
	var restartPolicy = corev1.RestartPolicyNever
	var Containers []corev1.Container

	Containers = append(Containers, corev1.Container{})

	return &batchv1.Job{
		ObjectMeta: metav1.ObjectMeta{
			Name:      cr.Name + "-job",
			Namespace: cr.Namespace,
		},
		Spec: batchv1.JobSpec{
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Name: cr.Name,
				},
				Spec: corev1.PodSpec{
					RestartPolicy: restartPolicy,
					Containers: Containers,
				},
			},
		},
	}
}
```

### Controller: Clone playbook source-code

Let's start adding features. First we need to clone the playbook source.
Modify our function `newJobForCR` and add:

```go
	initContainers := []corev1.Container{
		corev1.Container{
			Name:  "git-sync",
			Image: "k8s.gcr.io/git-sync:v3.1.6",
			VolumeMounts: []corev1.VolumeMount{
				corev1.VolumeMount{
					Name:      "project",
					MountPath: "/tmp/git",
				},
			},
			Env: []corev1.EnvVar{
				corev1.EnvVar{Name: "GIT_SYNC_REPO", Value: cr.Spec.Repo},
				corev1.EnvVar{Name: "GIT_SYNC_ROOT", Value: "/tmp/git"},
				corev1.EnvVar{Name: "GIT_SYNC_DEST", Value: "project"},
				corev1.EnvVar{Name: "GIT_SYNC_ONE_TIME", Value: "true"},
			},
		},
	}
```

Also, let's add a shared volume for containers:

```go
	Volumes := []corev1.Volume{
		{
			Name: "project",
			VolumeSource: corev1.VolumeSource{
				EmptyDir: &corev1.EmptyDirVolumeSource{},
			},
		},
	}
```

Finally, let's bind them both to the `PodSpec`:

```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go

 				Spec: corev1.PodSpec{
					RestartPolicy:  restartPolicy,
+					Volumes:        Volumes,
+					InitContainers: initContainers,
 					Containers:     Containers,
 				},
```

### Controller: Ansible Runner

Add `path` to imports:
```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go

 import (
        "context"
+       "path"
 )
```

```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go

-	Containers = append(Containers, corev1.Container{})
+	Containers = append(Containers, corev1.Container{
+		Name:       "ansible",
+		Image:      "ansible/ansible-runner:1.4.6",
+		WorkingDir: path.Join("/srv", "project"),
+		Args: []string{
+			"ansible-playbook",
+			"--inventory",
+			cr.Spec.Inventory,
+			"--limit",
+			cr.Spec.Limit,
+			cr.Spec.Playbook,
+		},
+		Env: []corev1.EnvVar{{Name: "ANSIBLE_FORCE_COLOR", Value: "true"}},
+		VolumeMounts: []corev1.VolumeMount{
+			corev1.VolumeMount{Name: "project", MountPath: "/srv"},
+		},
+	})
```

### Controller: Run Only Once

Add limits:

```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go

// newJobForCR returns a pod with the same name/namespace as the cr
func newJobForCR(cr *k8sv1alpha1.AnsibleJob) *batchv1.Job {
	var restartPolicy = corev1.RestartPolicyNever
	var Containers []corev1.Container
+	var backoffLimit int32 = 0
+	var completions int32 = 1
+	var parallelism int32 = 1
```

And bind to `JobSpec`:

```diff
+++ pkg/controller/ansiblejob/ansiblejob_controller.go

	return &batchv1.Job{
		ObjectMeta: metav1.ObjectMeta{
			Name:      cr.Name,
			Namespace: cr.Namespace,
		},
		Spec: batchv1.JobSpec{
+			BackoffLimit: &backoffLimit,
+			Completions:  &completions,
+			Parallelism:  &parallelism,
			Template: corev1.PodTemplateSpec{
```

## Run Operator's First Playbook

First, make sure you are _not_ running the operator-sdk server currently.
Ensure you have the CRD and CR deployed:

```bash
$ kubectl get crds
NAME                         CREATED AT
ansiblejobs.k8s.tikalk.com   2019-05-02T11:07:04Z

$ kubectl get ansiblejobs.k8s.tikalk.com
NAME                 AGE
playbook-debug-job   3h
```

Take notice to current pods & jobs:

```bash
$ kubectl get pod,job
No resources found.
```

:warning: If you have a pod here, delete it with: `kubectl delete pod --all`

Now, run the operator-sdk development local server:

```bash
$ export OPERATOR_NAME=ansible-operator
$ operator-sdk run local --watch-namespace=default

INFO[0000] Running the operator locally; watching namespace "default"
{ "msg": "Operator Version: 0.0.1" }
{ "msg": "Go Version: go1.14.4" }
{ "msg": "Go OS/Arch: darwin/amd64" }
{ "msg": "Version of operator-sdk: v0.18.2" }
```

You should see these logs:

```bash
Starting Controller
Starting workers
Reconciling AnsibleJob
Creating a new Pod
```

Check the status of Jobs and Pods:

```bash
$ kubectl get pod,job
NAME                               READY   STATUS      RESTARTS   AGE
pod/example-ansiblejob-job-rj86z   0/1     Completed   0          3m5s

NAME                               COMPLETIONS   DURATION   AGE
job.batch/example-ansiblejob-job   1/1           8s         3m5s
```

Finally, let's see the actual Ansible playbook logs:

```bash
$ kubectl logs job/example-ansiblejob-job

PLAY [all] *********************************************************************

TASK [Gathering Facts] *********************************************************
ok: [localhost]

TASK [Output group names] ******************************************************
ok: [localhost] => {
    "group_names": [
        "local"
    ]
}

TASK [Output host ansible facts] ***********************************************
ok: [localhost] => {
    "ansible_facts.env": {
        "ANSIBLE_FORCE_COLOR": "true",
        "HOME": "/root",
        "HOSTNAME": "playbook-debug-job-rj86z",
        "KUBERNETES_PORT": "tcp://10.96.0.1:443",
        "KUBERNETES_PORT_443_TCP": "tcp://10.96.0.1:443",
        "KUBERNETES_PORT_443_TCP_ADDR": "10.96.0.1",
        "KUBERNETES_PORT_443_TCP_PORT": "443",
        "KUBERNETES_PORT_443_TCP_PROTO": "tcp",
        "KUBERNETES_SERVICE_HOST": "10.96.0.1",
        "KUBERNETES_SERVICE_PORT": "443",
        "KUBERNETES_SERVICE_PORT_HTTPS": "443",
        "LANG": "en_US.UTF-8",
        "LANGUAGE": "en_US:en",
        "LC_ALL": "en_US.UTF-8",
        "PATH": "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
        "PWD": "/srv/rev-07d60cbba4826befeb89cd1f8ff376e9fb02a0af/playbooks",
        "RUNNER_BASE_COMMAND": "ansible-playbook",
        "SHLVL": "2",
        "_": "/usr/bin/python"
    }
}

PLAY RECAP *********************************************************************
localhost                  : ok=3    changed=0    unreachable=0    failed=0
```

Great success! :fireworks:

[Operator SDK]: https://sdk.operatorframework.io/docs/golang/installation/
[project layout]: https://sdk.operatorframework.io/docs/golang/references/project-layout/
