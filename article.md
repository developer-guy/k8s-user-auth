# Vanilla Kubernetes User authentication and autorization in depth
## Introduction
When you're managing vanilla K8s clusters you need to solve some problems like Ingress Balancing, Distributed storage and User Management, in this article we will focus on the later.

User [authentication &  authorization](https://auth0.com/docs/authorization/authentication-and-authorization), in simple terms, consists on verifying who a user is and verifying what they have access to, we will be using [RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) for authorization in our clusters.

Kubernetes can use client-certificates, bearer tokens, a proxy or HTTP basic auth, all of these methods will be validated by the API server, the attributes the API server uses for authentication are the following:
- Username: This is a string that identifies the user, should be unique among all of them, (e.g. admin@example.com, my-user-id, x24diausu=,  etc.)
- UID: Serves the same purpose as the Username but this attempts to be more consistent than the later, if possible you should be putting the same value on both of them
- Groups: A set of strings, this indicates a membership in logical manner to a collection of users in K8s, common usage of this is to prefix the groups, for example `system:masters` or `oidc:devs`
- Extra fiels: A map of strings that contains extra information that could be used by plugins, connectors, etc.

The kubernetes docs recommends using at least two methods:
- Service account tokens for service accounts attached to pods
- At least another method for user authentication, we will cover 3 methods.

Enough blahbery, let's cut to the chase.

## Certificate Signing Request
This method allows a client to ask for and X.509 certificate to be issued by the CA and delivered to him
### Manual process
For a test environment spin up minikube instance with `minikube start`, this was tested with `minikube version: v1.13.0`
1. Create your private key
`openssl genrsa -out myUsername.key 2048`
2. Create the CSR file
`openssl req -new -key myUsername.key -out myUsername.csr -subj "O=admin/CN=myUsername"`
3. Create a certificate request using kubectl
```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  # This has to match the id that you will use
  name: myUsername
spec:
  groups:
  # This means we want to add this csr to all of the authenticated users
  - system:authenticated
  request: $(cat myUsername.csr | base64 | tr -d "\n")
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF
```
3. As an admin, approve the request with `kubectl certificate approve MyUsername`.
4. Get the certificate with `kubectl get csr/MyUsername -o jsonpath="{.status.certificate}" | base64 -d > myUsername.crt`.
5. Create a clusterrole binding for the admin group.
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: admin-binding
subjects:
- kind: Group
  # This value is the one that k8s uses to define group membership
  # Must be the same in the openssl subject
  name: admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```
6. Add the new credentials to the kubeconfig with `kubectl config set-credentials myUsername --client-key=myUsername.key --client-certificate=myUsername.crt --embed-certs=true`.
7. Add the context with `kubectl config set-context myContext--cluster=minikube --user=myUsername`.
8. Use the context with `kubectl config use-context myContext`.  
You should have admin access to your cluster.
### Using go k8s client
1. Create the rsa private key
```
func getPrivateKeyPEMBytes(key *rsa.PrivateKey) *[]byte {
	var privateKey = &pem.Block{
		Type:  "PRIVATE KEY",
		Bytes: x509.MarshalPKCS1PrivateKey(key),
	}

	var b bytes.Buffer
	err := pem.Encode(&b, privateKey)
	if err != nil {
		fmt.Println("Fatal error ", err.Error())
		panic(err)
	}
	var rsaBytes = b.Bytes()
	return &rsaBytes
}
```
2. Create the CSR
```
type RSAData struct {
	Csr *[]byte
	Key *[]byte
}

func getCsrBytes(commonName string, organization string) *RSAData {
	reader := rand.Reader
	bitSize := 2048
	key, err := rsa.GenerateKey(reader, bitSize)
	keyBytes := getPrivateKeyPEMBytes(key)
	if err != nil {
		fmt.Println("Fatal error ", err.Error())
		panic(err)
	}

	subj := pkix.Name{
		CommonName:   commonName,
		Organization: []string{organization},
	}

	templateCsr := x509.CertificateRequest{
		Subject:            subj,
		SignatureAlgorithm: x509.SHA256WithRSA,
	}

	var b bytes.Buffer
	csrBytes, _ := x509.CreateCertificateRequest(rand.Reader, &templateCsr, key)
	err = pem.Encode(&b, &pem.Block{Type: "CERTIFICATE REQUEST", Bytes: csrBytes})
	sliceAddr := b.Bytes()
	var result = RSAData{
		Csr: &sliceAddr,
		Key: keyBytes,
	}
	return &result
}
```
3. Create the CSR in the cluster
```
var csr = k8sCertsV1.CertificateSigningRequest{}
csr.Name = user
csr.Spec.Groups = []string{"system:authenticated"}
csr.Spec.Usages = []k8sCertsV1.KeyUsage{"client auth"}
csr.Spec.Request = *rsaData.Csr
csr.Spec.SignerName = "kubernetes.io/kube-apiserver-client"
_, err := k8sClient.CertificatesV1().CertificateSigningRequests().Create(ctx, &csr, k8sMetaV1.CreateOptions{})
check(err)
```
4. Approve the CSR
```
func approveCsr(ctx context.Context, k8sClient *kubernetes.Clientset, csr k8sCertsV1.CertificateSigningRequest) {
	csr.Status.Conditions = append(csr.Status.Conditions, k8sCertsV1.CertificateSigningRequestCondition{
		Type:           k8sCertsV1.CertificateApproved,
		Reason:         "Approved by CICD",
		Message:        "This CSR was approved by CICD",
		Status:         "True",
		LastUpdateTime: k8sMetaV1.Now(),
	})
	_, err := k8sClient.CertificatesV1().CertificateSigningRequests().UpdateApproval(ctx, csr.Name, &csr, k8sMetaV1.UpdateOptions{})
	check(err)
}
```
5. Create the kubeconfig
```
func createKubeConfig(data KubeConfigData) {
	t, err := template.New("kubeconfig").Parse(kubeConfigTemplate)
	check(err)
	resultFile := fmt.Sprintf("/tmp/%v.kubeconfig", data.UserEmail)
	f, err := os.Create(resultFile)
	check(err)
	w := bufio.NewWriter(f)
	err = t.Execute(w, data)
	check(err)
	err = w.Flush()
	check(err)
}
```
This is the template:
```
apiVersion: v1
kind: Config
clusters:
- cluster:
    {{ .CAKey }}: {{ .ClusterCA }}
    server: {{ .ClusterEndpoint }}
  name: {{ .ClusterName }}
users:
- name: {{ .UserEmail }}
  user:
    client-certificate-data: {{ .ClientCertificateData }}
    client-key-data: {{ .ClientKeyData }}
contexts:
- context:
    cluster: {{ .ClusterName }}
    user: {{ .UserEmail }}
  name: {{ .User }}-{{ .ClusterName }}
current-context: {{ .User }}-{{ .ClusterName }}
```
## Webhook token
For demo purposes we use `k3d versionk3d version v3.0.1` and `k3s version v1.18.6-k3s1 (default)`  in this example.
This method allows authentication by verifying bearer tokens.  For this you need a service that handles a token that is provided by kubernetes once a user sends a request to the API server. We will specify the process bellow:

1. Create a file with the following contents and save it as `webhook-config.yaml`
```
apiVersion: v1
kind: Config
clusters:
  # The name of the service
  - name: myServiceName
    cluster:
      server: http://localhost:3000/authenticate
users:
  # The api configuration for the webhook
  - name: apiUsername
    user:
      token: secret
contexts:
  - name: webhook
    context:
      cluster: myServiceName
      user: apiUsername
current-context: webhook
```
2. For this part we need an aplication that handles the bearer token in some way and tells the api server that the user is authenticated. The api server expects that your application has an endpoint at `/authenticate` with the POST method, following we have an example for this in GO
```

type AuthResponseStatus struct {
	Authenticated bool                    `json:"authenticated"`
	User          *AuthResponseStatusUser `json:"user,omitempty"`
}

type AuthResponseStatusUser struct {
	Username string   `json:"username"`
	Uid      string   `json:"uid"`
	Groups   []string `json:"groups"`
}

type AuthResponse struct {
	ApiVersion string             `json:"apiVersion"`
	Kind       string             `json:"kind"`
	Status     AuthResponseStatus `json:"status"`
}

func authenticate(w http.ResponseWriter, r *http.Request) {
  # Accepts json
	w.Header().Set("Content-Type", "application/json")

  # Read the body as text
	reqBody, _ := ioutil.ReadAll(r.Body)
	var authRequest AuthRequest

  # Unmarshal the text into json
	err := json.Unmarshal(reqBody, &authRequest)

	if err != nil {
		json.NewEncoder(w).Encode(unauthorizedRespose)
		log.Printf("User : %v Cause: %v, ", reqBody, err)
		return
	}
	// Query github data username and groups of an org
	ctx := context.Background()
	ts := oauth2.StaticTokenSource(
		&oauth2.Token{AccessToken: authRequest.Spec.Token},
	)

  // Create and oauth2 client to connect to github
	tc := oauth2.NewClient(ctx, ts)
	client := github.NewClient(tc)
	req, _, err := client.Users.Get(context.Background(), "")

	if err != nil {
		json.NewEncoder(w).Encode(unauthorizedRespose)
		log.Printf("Cause: %v, ", err)
		return
	}

	user := *req.Login
  
  // Query the membership of the user to an specified organization
	membership, _, err := client.Organizations.GetOrgMembership(context.Background(), "", ",MY_GITHUB_ORGANIZATION>")

	if err != nil {
		json.NewEncoder(w).Encode(unauthorizedRespose)
		log.Printf("User : %v Cause: %v, ", user, err)
		return
	}
  
  // This is what kubernetes expects. See https://kubernetes.io/docs/reference/access-authn-authz/authentication/#webhook-token-authentication
	authRespose := AuthResponse{
		ApiVersion: authRequest.ApiVersion,
		Kind:       authRequest.Kind,
		Status: AuthResponseStatus{
			Authenticated: true,
			User: &AuthResponseStatusUser{
				Username: user,
				Uid:      user,
				Groups:   []string{*membership.Role},
			},
		},
	}
	json.NewEncoder(w).Encode(authRespose)
	log.Printf("User %v authenticated sucessfully", user)
}
```
3. Build the image with docker (e.g. `docker build -t webhook-app:v1 -f app/Dockerfile ./app`)
4. Create a k3d cluster:
```
# Notice that authentication-token-webhook-config-file flag points to the file create previously
k3d cluster create webhook \
-v $PWD/config:/etc/webhook \
--k3s-server-arg "--kube-apiserver-arg=authentication-token-webhook-config-file=/etc/webhook/webhook-config.yaml"
```
5. Import the webhook service image into k3d: `k3d image import webhook-app:v1 -c webhook`
6. Create a daemonset with this app
```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    k8s-app: webhook-app
  name: webhook-app
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: webhook-app
  template:
    metadata:
      labels:
        k8s-app: webhook-app
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
    spec:
      tolerations:
      # Allow the pods to be runned in master nodes (when the api server lives)
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - image: webhook-app:v1
        name: webhook-app
        ports:
        - containerPort: 3000
          hostPort: 3000
          protocol: TCP
      # This is for accessing it as localhost
      hostNetwork: true
      restartPolicy: Always
EOF
```
7. Create a github token with profile access(This is and example to showcase the felixibility, you can implement another auth method, the important thing it's that you return the json to the API server specifying that the user is authenticated or not)
8. Add the new credentials to the kubeconfig with `kubectl config set-credentials webhook --token=YOUR_GITHUB_TOKEN`.
7. Add the context with `kubectl config set-context myWHContext --cluster=webhook --user=webhook`.
9. Use the context with `kubectl config use-context myWHContext`.  
So now kubernetes uses your github token to verify that you belong to an organization.

## OIDC (OpenId Connect)
For demo purposes we use `k3d versionk3d version v3.0.1` and `k3s version v1.18.6-k3s1 (default)`  in this example.

K8s allows an OIDC provider as an Identity provider this is an excellent sequence diagram from the official docs.

![OIDC Sequence Diagram](https://d33wubrfki0l68.cloudfront.net/d65bee40cabcf886c89d1015334555540d38f12e/c6a46/images/docs/admin/k8s_oidc_login.svg)

As we can see, the magic happens when we, as an user, login to the IDP to get and `id token` and then the token is used as a bearer token with the kubectl commands.

In this example we will spinning up our own [dex](https://github.com/dexidp/dex) instance that has access to Gitlab as an upstream provider.

