# PodSet Operator

An example Kubernetes/Openshift Operator. This is an updated version of the operator example from: https://learn.openshift.com/operatorframework/go-operator-podset/, which is a few versions out of date. 

## Usage

First, install the `operator-sdk` binary using one of the methods described [here](https://github.com/operator-framework/operator-sdk/blob/master/doc/user/install-operator-sdk.md).

Then, log into a kubernetes cluster. Your operator will use your local `~/.kube/config` file to determine which cluster to monitor. 

Now clone this repo. If the `PodSet` Custom Resource Definition (CRD) has not yet been created in your Openshift cluster, run the following command:

```bash
$ oc create -f deploy/crds/app.example.com_appservices_crd.yaml
```

Now run the operator in dev mode:

```bash
$ operator-sdk run --local --namespace podset-operator
```

This will start the operator (the controller code, specifically) on your local machine, linked to the kubernetes cluster you logged into. Keep this terminal open to watch operator logs. 

Open another terminal. Create an instance of a `PodSet`:

```bash
$ oc create -f deploy/crds/app.example.com_v1alpha1_podset_cr.yaml
```

Check out your new `PodSet`:

```bash
$ oc get podsets
NAME                     AGE
another-example-podset   2d21h
example-podset           2d22h
```

## Steps to Create The PodSet Operator From Scratch

This repo already has a fully-generated PodSet operator with working code. If you want to create the example on your own from scratch, here are some instructions.

Start with an empty directory. Create a namespace-scoped operator:

```bash
$ operator-sdk new podset-operator --type=go --repo=podset-operator
```

Enter the newly created directory:

```bash
$ cd podset-operator
```

Use the SDK binary to generate scaffolding for the PodSet API definition (version `v1alpha1`):

```bash
$ operator-sdk add api --api-version=app.example.com/v1alpha1 --kind=PodSet
```

Edit the `PodSetSpec` and `PodSetStatus` sections of the file `pkg/apis/app/v1alpha1/podset_types.go` to look like:

```golang
package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// PodSetSpec defines the desired state of PodSet
type PodSetSpec struct {
	Replicas int32 `json:"replicas"`
}

// PodSetStatus defines the observed state of PodSet
type PodSetStatus struct {
	Replicas int32    `json:"replicas"`
	PodNames []string `json:"podNames"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// PodSet is the Schema for the podsets API
// +kubebuilder:subresource:status
// +kubebuilder:resource:path=podsets,scope=Namespaced
type PodSet struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   PodSetSpec   `json:"spec,omitempty"`
	Status PodSetStatus `json:"status,omitempty"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// PodSetList contains a list of PodSet
type PodSetList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []PodSet `json:"items"`
}

func init() {
	SchemeBuilder.Register(&PodSet{}, &PodSetList{})
}
```

Now generate scaffolding for the controller:

```
operator-sdk add controller --api-version=app.example.com/v1alpha1 --kind=PodSet
```

and alter the file `pkg/controller/podset/podset_controller.go` to match the following:

```golang
package podset

import (
	"context"
	"reflect"

	appv1alpha1 "podset-operator/pkg/apis/app/v1alpha1"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/manager"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
	"sigs.k8s.io/controller-runtime/pkg/source"
)

var log = logf.Log.WithName("controller_podset")

/**
* USER ACTION REQUIRED: This is a scaffold file intended for the user to modify with their own Controller
* business logic.  Delete these comments after modifying this file.*
 */

// Add creates a new PodSet Controller and adds it to the Manager. The Manager will set fields on the Controller
// and Start it when the Manager is Started.
func Add(mgr manager.Manager) error {
	return add(mgr, newReconciler(mgr))
}

// newReconciler returns a new reconcile.Reconciler
func newReconciler(mgr manager.Manager) reconcile.Reconciler {
	return &ReconcilePodSet{client: mgr.GetClient(), scheme: mgr.GetScheme()}
}

// add adds a new Controller to mgr with r as the reconcile.Reconciler
func add(mgr manager.Manager, r reconcile.Reconciler) error {
	// Create a new controller
	c, err := controller.New("podset-controller", mgr, controller.Options{Reconciler: r})
	if err != nil {
		return err
	}

	// Watch for changes to primary resource PodSet
	err = c.Watch(&source.Kind{Type: &appv1alpha1.PodSet{}}, &handler.EnqueueRequestForObject{})
	if err != nil {
		return err
	}

	// TODO(user): Modify this to be the types you create that are owned by the primary resource
	// Watch for changes to secondary resource Pods and requeue the owner PodSet
	err = c.Watch(&source.Kind{Type: &corev1.Pod{}}, &handler.EnqueueRequestForOwner{
		IsController: true,
		OwnerType:    &appv1alpha1.PodSet{},
	})
	if err != nil {
		return err
	}

	return nil
}

// blank assignment to verify that ReconcilePodSet implements reconcile.Reconciler
var _ reconcile.Reconciler = &ReconcilePodSet{}

// ReconcilePodSet reconciles a PodSet object
type ReconcilePodSet struct {
	// This client, initialized using mgr.Client() above, is a split client
	// that reads objects from the cache and writes to the apiserver
	client client.Client
	scheme *runtime.Scheme
}

// Reconcile reads that state of the cluster for a PodSet object and makes changes based on the state read
// and what is in the PodSet.Spec
// TODO(user): Modify this Reconcile function to implement your Controller logic.  This example creates
// a Pod as an example
// Note:
// The Controller will requeue the Request to be processed again if the returned error is non-nil or
// Result.Requeue is true, otherwise upon completion it will remove the work from the queue.
func (r *ReconcilePodSet) Reconcile(request reconcile.Request) (reconcile.Result, error) {
	reqLogger := log.WithValues("Request.Namespace", request.Namespace, "Request.Name", request.Name)
	reqLogger.Info("Reconciling PodSet")

	// Fetch the PodSet instance
	podSet := &appv1alpha1.PodSet{}
	err := r.client.Get(context.TODO(), request.NamespacedName, podSet)
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			return reconcile.Result{}, nil
		}
		// Error reading the object - requeue the request.
		return reconcile.Result{}, err
	}

	/* List all pods owned by this PodSet instance */

	// initialize empty pod list
	existingPods := &corev1.PodList{}

	// create a selector
	lbls := map[string]string{
		"app":     podSet.Name,
		"version": "v0.1",
	}
	labelSelector := labels.SelectorFromSet(lbls)

	// use the client to list all existing pods, put data in existingPods
	err = r.client.List(context.TODO(), existingPods,
		&client.ListOptions{
			Namespace:     request.Namespace,
			LabelSelector: labelSelector,
		},
	)
	if err != nil {
		reqLogger.Error(err, "failed to list existing pods in the podSet")
		return reconcile.Result{}, err
	}

	// count the pods that are running or pending. get their names so we
	// can populate the 'podNames' status object
	existingPodNames := []string{}
	for _, pod := range existingPods.Items {
		if pod.GetObjectMeta().GetDeletionTimestamp() != nil {
			continue
		}
		if pod.Status.Phase == corev1.PodPending || pod.Status.Phase == corev1.PodRunning {
			existingPodNames = append(existingPodNames, pod.GetObjectMeta().GetName())
		}
	}

	reqLogger.Info("Checking podset", "expected replicas", podSet.Spec.Replicas, "Found Pod.Names", existingPodNames)
	// Create a new status object based on observed status
	status := appv1alpha1.PodSetStatus{
		Replicas: int32(len(existingPodNames)),
		PodNames: existingPodNames,
	}

	// compare observed status to current recorded podset status. If
	// different, update recorded status to observed status
	if !reflect.DeepEqual(podSet.Status, status) {
		podSet.Status = status
		err := r.client.Status().Update(context.TODO(), podSet)
		if err != nil {
			reqLogger.Error(err, "failed to update the podSet")
			return reconcile.Result{}, err
		}
	}

	// If there are more pods than desired, scale down pods
	if int32(len(existingPodNames)) > podSet.Spec.Replicas {
		// delete a pod. Just one at a time (this reconciler will be called again afterwards)
		reqLogger.Info("Deleting a pod in the podset", "expected replicas", podSet.Spec.Replicas, "found replicas", len(existingPodNames))
		// pick first pod to delete
		pod := existingPods.Items[0]
		err = r.client.Delete(context.TODO(), &pod)
		if err != nil {
			reqLogger.Error(err, "failed to delete a pod")
			return reconcile.Result{}, err
		}
	}

	// if there are less pods than desired, scale up pods
	if int32(len(existingPodNames)) < podSet.Spec.Replicas {
		reqLogger.Info("Adding a pod in the podset", "expected replicas", podSet.Spec.Replicas, "found replicas", len(existingPodNames))
		// create a new pod. Just one at a time (this reconciler will be called again afterwards)
		pod := newPodForCR(podSet)
		if err := controllerutil.SetControllerReference(podSet, pod, r.scheme); err != nil {
			reqLogger.Error(err, "unable to set owner reference on new pod")
			return reconcile.Result{}, err
		}
		err = r.client.Create(context.TODO(), pod)
		if err != nil {
			reqLogger.Error(err, "failed to create a pod")
			return reconcile.Result{}, err
		}
	}
	return reconcile.Result{Requeue: true}, nil
}

// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *appv1alpha1.PodSet) *corev1.Pod {
	labels := map[string]string{
		"app":     cr.Name,
		"version": "v0.1",
	}
	return &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			GenerateName: cr.Name + "-pod",
			Namespace:    cr.Namespace,
			Labels:       labels,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "busybox",
					Image:   "busybox",
					Command: []string{"sleep", "3600"},
				},
			},
		},
	}
}
```

Generate the operator deployment manifests:

```bash
$ operator-sdk generate k8s
$ operator-sdk generate openapi
```

You now have definitions for a CRD and an operator! Look in the `Usage` section above to test it.
