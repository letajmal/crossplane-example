# Crossplane Demonstrations

## Introduction: The Global Control Plane

Crossplane can be thought of as a modern replacement for Terraform, built from the ground up for the GitOps era. While traditional Infrastructure as Code (IaC) tools require discrete CI/CD pipeline runs to manage state, Crossplane makes Kubernetes the **global control plane** for all your infrastructure. 

Operating on a continuous reconciliation loop, it integrates natively with GitOps tools like ArgoCD out of the box. The core philosophy is simple: **If a system has an API, you can manage it from Kubernetes using Crossplane.**

---

This document demonstrates two key capabilities of Crossplane:
1. Bootstrapping a complete project environment using a single Composite Resource (XR).
2. Extending Crossplane by creating a custom provider for an internal API (IMS API).

---

## Demo 1: Bootstrapping a Project via a Composite Resource (XR)

In this demonstration, we showcase how Crossplane can abstract complex infrastructure provisioning into a simple, single configuration file called a Composite Resource (XR). 

We will define an XR named `IMSProject`. When a developer creates this resource, Crossplane will automatically provision the following:
- A Kubernetes cluster
- A GitOps repository in GitLab
- An ArgoCD Application and Project
- Necessary credentials to link the GitLab repository and the new Kubernetes cluster to ArgoCD.

### The Developer Experience: `IMSProject` Claim

To keep the demonstration concise, we are only showing the Composite Resource (the Claim) that a developer would submit. The underlying `CompositeResourceDefinition` (XRD) and `Composition` files define *how* these resources are wired together, but they are typically long and maintained by Platform Engineers.

Here is what the developer applies:

```yaml
apiVersion: ims.example.org/v1alpha1
kind: IMSProject
metadata:
  name: demo-microservice-project
  namespace: development
spec:
  parameters:
    # Cluster details
    clusterSize: medium
    region: us-east-1
    # Repository details
    repoName: demo-microservice
    visibility: private
    # ArgoCD details
    destinationNamespace: demo-microservice-ns
```

### How It Works

1. **Submission**: The developer submits the `IMSProject` manifest to the control plane.
2. **Reconciliation**: Crossplane reads the corresponding `Composition` for `IMSProject`.
3. **Provisioning**: The `Composition` instructs various Crossplane Providers (e.g., `provider-kubernetes`, `provider-gitlab`, `provider-helm`/ArgoCD providers) to:
   - Request a new cluster from the cloud provider.
   - Call the GitLab API to create `demo-microservice`.
   - Apply Kubernetes manifests to the cluster where ArgoCD is hosted to register the new cluster and repository.
   - Create ArgoCD `AppProject` and `Application` custom resources pointing to the newly created GitLab repo.
4. **Ready State**: Once all underlying resources are successfully created and healthy, the `IMSProject` status becomes `READY`.

### Building an Internal Developer Platform (IDP) with Backstage

We can put a frontend like **Backstage** in front of this Crossplane API to create a complete Internal Developer Platform (IDP).

1. **Software Templates**: In Backstage, we define a "Create New IMS Project" Software Template.
2. **Form Inputs**: The developer fills out a web form asking for the `repoName`, `clusterSize`, etc.
3. **Action Execution**: Backstage takes the form inputs, generates the `IMSProject` YAML manifest, and automatically pushes a commit to your infrastructure Git repository.
4. **GitOps Sync**: ArgoCD detects the new manifest in Git and syncs the `IMSProject` resource directly to the Crossplane control plane.
5. **Visibility**: Backstage can then use its Kubernetes plugin to track the status of the `IMSProject` resource, providing visual feedback to the developer on the provisioning progress without them ever needing to touch `kubectl`.

---

## Demo 2: Creating a Custom Provider for the IMS API

Crossplane providers are essentially Kubernetes operators that reconcile custom resources to external API states. In this demonstration, we explain how to build a custom provider for an internal service, the "IMS API," which has a CRUD `/users` endpoint.

### Overview of Custom Providers

To create a custom provider, you typically use the [Crossplane Provider Template](https://github.com/crossplane/provider-template). The process involves defining the Custom Resource Definition (CRD) representing your API object (e.g., a `User`) and writing Go code that implements the CRUD operations against your API.

### 1. Defining the Custom Resource

First, you define the Go types for the API resource in the provider codebase.

```go
// apis/users/v1alpha1/user_types.go

type UserParameters struct {
	// FirstName is the user's first name
	FirstName string `json:"firstName"`
	// LastName is the user's last name
	LastName string `json:"lastName"`
	// Role assigned in the IMS system
	Role string `json:"role"`
}

// User is the Schema for the IMS Users API
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
type User struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   UserSpec   `json:"spec"`
	Status UserStatus `json:"status,omitempty"`
}
```

### 2. Implementing the External Client

Next, you implement the `ExternalClient` interface, which tells Crossplane how to `Observe`, `Create`, `Update`, and `Delete` the resource in the external system (the IMS API).

```go
// internal/controller/user/user.go

type external struct {
	client *imsapi.Client
}

func (c *external) Observe(ctx context.Context, mg resource.Managed) (managed.ExternalObservation, error) {
	cr, ok := mg.(*v1alpha1.User)
	if !ok {
		return managed.ExternalObservation{}, errors.New("unexpected managed resource type")
	}

	// Call IMS API GET /users/{id}
	user, err := c.client.GetUser(ctx, cr.Status.AtProvider.ID)
	if err != nil {
		return managed.ExternalObservation{ResourceExists: false}, nil
	}

	// Compare external state with desired state to see if an update is needed
	isUpToDate := cr.Spec.ForProvider.FirstName == user.FirstName &&
                  cr.Spec.ForProvider.LastName == user.LastName

	return managed.ExternalObservation{
		ResourceExists:    true,
		ResourceUpToDate:  isUpToDate,
	}, nil
}

func (c *external) Create(ctx context.Context, mg resource.Managed) (managed.ExternalCreation, error) {
	cr, _ := mg.(*v1alpha1.User)
	
    // Call IMS API POST /users
	resp, err := c.client.CreateUser(ctx, imsapi.CreateUserRequest{
		FirstName: cr.Spec.ForProvider.FirstName,
		LastName:  cr.Spec.ForProvider.LastName,
        Role:      cr.Spec.ForProvider.Role,
	})
	
	if err != nil {
		return managed.ExternalCreation{}, err
	}

	// Save the external ID to the Status
	cr.Status.AtProvider.ID = resp.ID
	return managed.ExternalCreation{}, nil
}

func (c *external) Update(ctx context.Context, mg resource.Managed) (managed.ExternalUpdate, error) {
	cr, _ := mg.(*v1alpha1.User)
	
    // Call IMS API PUT /users/{id}
	err := c.client.UpdateUser(ctx, cr.Status.AtProvider.ID, imsapi.UpdateUserRequest{
        // ... mapped fields
	})
	
	return managed.ExternalUpdate{}, err
}

func (c *external) Delete(ctx context.Context, mg resource.Managed) error {
	cr, _ := mg.(*v1alpha1.User)
	
    // Call IMS API DELETE /users/{id}
	return c.client.DeleteUser(ctx, cr.Status.AtProvider.ID)
}
```

### 3. Usage

Once the provider is compiled, packaged, and installed in the Crossplane cluster, a user can manage IMS API users natively using Kubernetes manifests:

```yaml
apiVersion: users.ims.example.org/v1alpha1
kind: User
metadata:
  name: jdoe
spec:
  forProvider:
    firstName: John
    lastName: Doe
    role: Administrator
  providerConfigRef:
    name: ims-provider-config
```

When this manifest is applied, Crossplane uses the custom provider to issue the HTTP `POST` to the IMS API, track the user's state, and ensure it remains consistent with the declared configuration.
