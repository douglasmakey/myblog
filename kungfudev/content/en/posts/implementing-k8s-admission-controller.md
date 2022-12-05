---
title: "Implementing a simple K8s admission controller in Go"
date: 2021-03-01
tags: ["go", "k8s", "admissioncontroller"]
cover:
    image: "https://d33wubrfki0l68.cloudfront.net/af21ecd38ec67b3d81c1b762221b4ac777fcf02d/7c60e/images/blog/2019-03-21-a-guide-to-kubernetes-admission-controllers/admission-controller-phases.png"
---

## What is an admission controller?

> In a nutshell, Kubernetes admission controllers are plugins that govern and enforce how the cluster is used. They can be thought of as a gatekeeper that intercept (authenticated) API requests and may change the request object or deny the request altogether. The admission control process has two phases: the mutating phase is executed first, followed by the validating phase.

Kubernetes admission Controller Phases:

![](https://d33wubrfki0l68.cloudfront.net/af21ecd38ec67b3d81c1b762221b4ac777fcf02d/7c60e/images/blog/2019-03-21-a-guide-to-kubernetes-admission-controllers/admission-controller-phases.png)

An admission controller is a piece of software that intercepts requests to the Kubernetes API server before the persistence of the object (the k8s resource such as Pod, Deployment, Service, etc...) in the etcd database, but after the request is authenticated and authorized.

With the admission controller, we can validate or "mutate" the resource on the incoming requests. For example, imagine the following use cases:

* You want to apply a simple security validation that none of the containers in your cluster could use the `latest` tag.
* You may want to set some default values such as `annotations` or `labels` in every resource that you deploy.

There are two types of admission controllers in Kubernetes. They are validating admission controller and mutating admission controller. Mutating admission controllers are invoked first and can "modify" objects. After all object modifications are complete, and after the incoming object is validated by the API server, validating admission controllers are invoked. They can reject requests to enforce custom policies.

> Note: I put the words "modify" and "mutate" in quotes because we don't really modify the resource itself. We tell Kubernetes what it should modify to the object "the K8s resource" using JSON Patch format.

A mutating admission controller can act as a mutating or validating controller. It can perform both actions to the request simultaneously. However, remember that the mutating admission controllers are executed first. To ensure that you will validate an object's last state, you should use the validating admission controller.

## Register our admission controller webhooks

I created this repository that has all the code for this example and a simple boilerplate for an admission controller in Go [GitHub](https://github.com/douglasmakey/admissioncontroller).

A cluster on which this example can be tested must be running Kubernetes 1.9.0 or above. Also it should have the admissionregistration.k8s.io/v1beta1 API enabled. You can verify that using the following command:

```
kubectl api-versions
...
admissionregistration.k8s.io/v1beta1
...
```

You should check that `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` are activated in your cluster inspecting the `kube-apiserver`.

```text
--enable-admission-plugins=..,MutatingAdmissionWebhook,ValidatingAdmissionWebhook.."
```

To implement our admission controller, the Kubernetes API server needs to know when and where to send the incoming requests to our admission controller. We have to create a`ValidatingWebhookConfiguration` or `MutatingWebhookConfiguration` object in Kubernetes depends on what we want.

For example, the following configuration is to register a `ValidatingWebhookConfiguration` to apply some validations to creating a `Pod`.

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-validation
webhooks:
  - name: pod-validation.default.svc
    clientConfig:
      service:
        name: admission-server
        namespace: default
        path: "/validate/pods"
      caBundle: "${CA_BUNDLE}"
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
```

The main parts here are:

* `clientConfig`: The config for our admission controller server.
	* `service.name`: The name of the `service` for our admission controller server.
	* `service.namespace`: In what namespace our admission controller server is.
	* `service.path`: The path where our admission controller will receive this webhook request.
* `rules`: Contains the operations, resources, resources' api versions and groups that you want to intercept
	* `operations`: The operations you wish to intercept for these resources. For example, `CREATE`, `DELETE`, `UPDATED`
	* `resources`: The resources you want to intercept on this webhook. For example, `Pods`, `Deployment`, `Service`, etc.
	* `apiGroups` and `apiVersions` of these resources.

The Kubernetes API server makes an HTTPS POST request to the given service and URL path. Since a webhook must be served via HTTPS, we need proper certificates for the server. These certificates can be self-signed, but we need Kubernetes to instruct the respective CA certificate when talking to the webhook server. For that, you see the `caBundle` in the configuration.

In the GitHub repository, you will find the `demo/deploy.sh` script that will create a self-signed certificates and create the webhooks in `demo/webhooks.yaml` for this example.

To correctly manage your certificates in production, you could use something like a https://cert-manager.io.

## Create our admission controller server

> Note: All the code examples here have been simplified to make them easier to read. For the full implementation, please visit the [repository](https://github.com/douglasmakey/admissioncontroller).

Let’s start creating a simple HTTPS. It should have an endpoint for the path that we defined for our admission webhook.

```go
// http/server.go
func NewServer(port string) *http.Server {
	// Instances hooks
	podsValidation := pods.NewValidationHook()
	// Routers
	ah := newAdmissionHandler()
	mux := http.NewServeMux()
	mux.Handle("/validate/pods", ah.Serve(podsValidation)) // The path of the webhook for Pod validation.
	return &http.Server{
		Addr:    fmt.Sprintf(":%s", port),
		Handler: mux,
	}
}

// cmd/main.go
func main() {
	// flags
	// ...
	server := http.NewServer(port)
	if err := server.ListenAndServeTLS(tlscert, tlskey); err != nil {
		log.Errorf("Failed to listen and serve: %v", err)
	}
}
```

Then we have to create the `admissionHandler` to receive all the requests from our webhooks. These requests are coming with a JSON-encoded [AdmissionReview](https://github.com/kubernetes/api/blob/master/admission/v1beta1/types.go#L45) (with the Request field filled) in the request body. The response should be a JSON AdmissionReview with the Response field filled.

```go
// http/handlers.go

type admissionHandler struct {...}

// Serve returns a http.HandlerFunc for an admission webhook
func (h *admissionHandler) Serve(hook admissioncontroller.Hook) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		// HTTP validations
		// ...
		body, err := io.ReadAll(r.Body)
		if err != nil {...}

		var review admission.AdmissionReview
		if _, _, err := h.decoder.Decode(body, nil, &review); err != nil {...}

		result, err := hook.Execute(review.Request)
		if err != nil {...}

		admissionResponse := v1beta1.AdmissionReview{
			Response: &v1beta1.AdmissionResponse{
				UID:     review.Request.UID,
				Allowed: result.Allowed,
				Result:  &meta.Status{Message: result.Msg},
			},
		}
		//...
		res, err := json.Marshal(admissionResponse)
		if err != nil {...}

		w.WriteHeader(http.StatusOK)
		w.Write(res)
	}
}
```

As you noticed in the code above, our handler receives a `Hook` struct and invokes its `Execute` method to process the request. In the `Hook` struct, we can register an `AdmitFunc` for each operation allowed.

```go
// AdmitFunc defines how to process an admission request
type AdmitFunc func(request *admission.AdmissionRequest) (*Result, error)

// Hook represents the set of functions for each operation in an admission webhook.
type Hook struct {
	Create  AdmitFunc
	Delete  AdmitFunc
	Update  AdmitFunc
	Connect AdmitFunc
}

// Execute evaluates the request and try to execute the function for operation specified in the request.
func (h *Hook) Execute(r *admission.AdmissionRequest) (*Result, error) {
	switch r.Operation {
	case admission.Create:
		return wrapperExecution(h.Create, r)
	.....
	}
	return &Result{Msg: fmt.Sprintf("Invalid operation: %s", r.Operation)}, nil
}
```

Now, we are ready to just focus on our business logic. We need to write the code for the webhook that we registered.

We will implement the logic for our `ValidatingWebhook` that we registered for the `CREATE` operation in the `Pod` resource. We want to validate that none of the pod's containers could use the `latest` tag in its image.

```go
// pods/pods.go
// NewValidationHook creates a new instance of pods validation hook
func NewValidationHook() admissioncontroller.Hook {
	return admissioncontroller.Hook{
		Create: validateCreate(),
	}
}

// validateImages validates that none of the containers use the `latest` tag.
func validateImages() admissioncontroller.AdmitFunc {
	return func(r *v1beta1.AdmissionRequest) (*admissioncontroller.Result, error) {
		pod, err := parsePod(r.Object.Raw)
		if err != nil {
			return &admissioncontroller.Result{Msg: err.Error()}, nil
		}

		for _, c := range pod.Spec.Containers {
			if strings.HasSuffix(c.Image, ":latest") {
				return &admissioncontroller.Result{Msg: "You cannot use the tag 'latest' in a container."}, nil
			}
		}

		return &admissioncontroller.Result{Allowed: true}, nil
	}
}
```

Of course, this is a very basic example, and you could implement more complex validations depends on your use cases. For instance, on the repository, we have another interesting example. Using a `MutattingWebhook` and JSON Patch, we inject a container to our pod as a sidecar.

> Note: Istio uses a similar approach to inject its sidecar containers.

```go
func mutateCreate() admissioncontroller.AdmitFunc {
	return func(r *v1beta1.AdmissionRequest) (*admissioncontroller.Result, error) {
		var operations []admissioncontroller.PatchOperation
		pod, err := parsePod(r.Object.Raw)
		if err != nil {
			return &admissioncontroller.Result{Msg: err.Error()}, nil
		}

		// Very simple logic to inject a new "sidecar" container.
		if pod.Namespace == "special" {
			var containers []v1.Container
			containers = append(containers, pod.Spec.Containers...)
			sideC := v1.Container{
				Name:    "test-sidecar",
				Image:   "busybox:stable",
				Command: []string{"sh", "-c", "while true; do echo 'I am a container injected by mutating webhook'; sleep 2; done"},
			}
			containers = append(containers, sideC)
			operations = append(operations, admissioncontroller.ReplacePatchOperation("/spec/containers", containers))
		}

		return &admissioncontroller.Result{
			Allowed:  true,
			PatchOps: operations,
		}, nil
	}
}
```

## Deploy and test

Run `demo/deploy.sh` will create a self-signed CA, a certificate, and private for the server and the webhooks. It also will create the following resources:

* Secret TLS
* The Deployment of our admission server using the `demo/deployment.yaml`
* A Service for our admission server
* All the Admission webhooks

> Note: demo/deploy.sh is just for develop/test environment. It was not intended for production.

You can see all the created resources:

```shell
kubectl get svc
NAME               TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
admission-server   ClusterIP   10.43.120.27   <none>        443/TCP   1h

kubectl get deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
admission-server   1/1     0            1           1h

kubectl get secret
NAME                  TYPE                                  DATA   AGE
admission-tls         kubernetes.io/tls                     2      1h

kubectl get mutatingwebhookconfigurations
NAME           WEBHOOKS   AGE
pod-mutation   1          1h

kubectl get validatingwebhookconfigurations
NAME                    WEBHOOKS   AGE
deployment-validation   1          1h
pod-validation          1          1h
```

Now, we can test our webhook. If we try to create a `Pod` using the following manifest will fail.

```yaml
# demo/pods/01_fail_pod_creation_test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webserver
spec:
  containers:
    - name: webserver
      image: nginx:latest
      ports:
        - containerPort: 80
```

> Note: You can use the different manifests inside `demo/pods` and `demo/deployments` to test the validations and mutations.

```shell
 kubectl create -f pods/01_fail_pod_creation_test.yaml
Error from server: error when creating "pods/01_fail_pod_creation_test.yaml": admission webhook "pod-validation.default.svc" denied the request: You cannot use the tag 'latest' in a container.
```

## Conclusion

As you can see, the admission controller is a powerful feature that allows us to implement custom rules and default values to the k8s resources. It is one way you can extend the k8s behavior.

I hope you have enjoyed this article, and any feedback will be very well received.

If you are a Go developer or a Kubernetes ecosystem fan and want to work on many exciting stuff. In that case, we are hiring in my team at [Ubisoft](https://jobs.smartrecruiters.com/Ubisoft2/743999727156666-kubernetes-sre-specialist-ubisoft-kubernetes-service?trid=4556ca14-794c-4023-bd62-6813e288177e).

Many thanks.

### Some interesting links

* https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/#why-do-i-need-admission-controllers
* https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers
* https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/
* https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
* http://jsonpatch.com