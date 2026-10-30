To install cert-manager on a Minikube cluster for local development, you can choose between two standard methods: Helm (recommended for customization) or kubectl apply (quickest method).
Before you start, make sure your Minikube cluster is up and running (minikube start).
------------------------------
## Method 1: Install using Helm (Recommended)
Using Helm allows you to manage lifecycle updates easily and seamlessly enable custom settings.

## Step 1: Add the Jetstack Helm repository:
   
   ```bash
   helm repo add jetstack https://charts.jetstack.io
   helm repo update
   ```
## Step 2: Install cert-manager with Custom Resource Definitions (CRDs) enabled:
   ```bash
   helm install cert-manager jetstack/cert-manager \
     --namespace cert-manager \
     --create-namespace \
     --set crds.enabled=true
   ```
   
------------------------------
## Step 3: Verify the Installation
Check that all three cert-manager pods (webhook, cainjector, and the main controller) are up and running: [3] 
```
kubectl get pods --namespace cert-manager
```
Expected Output:
```
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-5c6866597-xlwsh               1/1     Running   0          30s
cert-manager-cainjector-745979967f-77g8t   1/1     Running   0          30s
cert-manager-webhook-79c55b6bf-46fpb       1/1     Running   0          30s
```
------------------------------
## Step 4: Configure for Local Development (Self-Signed Certificates)
For a local environment, you typically do not want to use Let's Encrypt (which requires a public domain and DNS challenges). Instead, you should create a Self-Signed ClusterIssuer to issue local TLS certificates.

   1. Create a file named issuer.yaml:
   ```yaml
   apiVersion: cert-manager.io/v1
   kind: ClusterIssuer
   metadata:
     name: selfsigned-cluster-issuer
   spec:
     selfSigned: {}
   ```
   2. Apply the Issuer:
   ```
   kubectl apply -f issuer.yaml
   ```
   
## Step 5: Secure an Ingress (Example)
To automatically provision a TLS certificate for your local applications, enable Minikube's built-in NGINX Ingress controller:
```
minikube addons enable ingress
```
Then, you can annotate your applications' standard Ingress resources to tell cert-manager to secure it: 
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    # This annotation triggers cert-manager to issue the cert
    cert-manager.io/cluster-issuer: "selfsigned-cluster-issuer" 
spec:
  tls:
  - hosts:
    - vaibhavlagad.local.com
    secretName: my-app-tls-secret # cert-manager will generate this secret
  rules:
  - host: vaibhavlagad.local.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 8080
```

------------------------------------------


To access your application at https://vaibhavlagad.local.com, you must first map that domain to your Minikube IP address in your local machine's hosts file. Because your browser doesn't automatically know where vaibhavlagad.local.com is, it needs a manual routing pointer.
Follow these steps to complete the link:
## 1. Get your Minikube IP Address
Run the following command in your terminal to find your cluster's local IP address:
```
minikube ip
```
(For this example, let's assume the output is 192.168.49.2)
## 2. Update your Local Hosts File
You need to add a routing mapping on your main computer (not inside Minikube).

* On Mac / Linux:
1. Open your terminal and run:
   ```
   sudo nano /etc/hosts
   ```
   2. Add the following line at the bottom of the file (replace 192.168.49.2 with your actual Minikube IP):
   ```
   192.168.49.2 vaibhavlagad.local.com
   ```
   3. Press Ctrl + O then Enter to save, and Ctrl + X to exit.
* On Windows (Administrator):
   1. Open Notepad as an Administrator.
   2. Open C:\Windows\System32\drivers\etc\hosts.
   3. Add the same line to the bottom: 192.168.49.2 vaibhavlagad.local.com and save.

## 3. Match the Host in your Ingress Manifest
Ensure the host rules inside your Kubernetes Ingress match your new domain name exactly:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    cert-manager.io/cluster-issuer: "selfsigned-cluster-issuer"
spec:
  tls:
  - hosts:
    - vaibhavlagad.local.com
    secretName: vaibhavlagad-tls-secret
  rules:
  - host: vaibhavlagad.local.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service  # Make sure this matches your actual service name
            port:
              number: 80         # Make sure this matches your service port
```
## ⚠️ A Note on Browser Warnings
Since you are using a Self-Signed issuer, your browser will display a "Your connection is not private" (NET::ERR_CERT_AUTHORITY_INVALID) warning when you open https://vaibhavlagad.local.com.

* This is completely normal for local development!
* You can click "Advanced" and then "Proceed to vaibhavlagad.local.com (unsafe)" to view your app.

To get rid of the browser warning, you need to transition from a Self-Signed issuer to a Local CA (Certificate Authority) issuer. By generating your own root certificate, adding it to your computer’s trusted store, and letting cert-manager sign your app certificates with it, your browser will show a green lock icon.

Follow these steps to set up a trusted Local CA:

## Step 1: Create a Certificate Authority (CA) Private Key and Certificate
Run the following commands in your terminal to generate a private key and a root certificate valid for 10 years:

# 1. Generate a secure RSA private key for your CA
```bash
openssl genrsa -out local-ca.key 2048
```
# 2. Create the Root CA certificate
```bash
openssl req -x509 -new -nodes -key local-ca.key -sha256 -days 3650 -out local-ca.crt \
  -subj "/CN=Local Dev Root CA/O=My Local Development"
  ```
Use double slash (//) in windows

## Step 2: Make your Computer Trust the CA
You must tell your local operating system to trust this new root certificate.

* Mac Users:
  1. Open Keychain Access.
  2. Drag and drop the local-ca.crt file into the System keychain.
  3. Double-click the imported certificate, expand Trust, and change "When using this certificate" to Always Trust.
* Windows Users:
  1. Press Win + R, type certlm.msc, and hit Enter.
  2. Right-click Trusted Root Certification Authorities > All Tasks > Import.
  3. Browse and select your local-ca.crt file to complete the wizard.
* Linux (Ubuntu/Debian) Users:
  ```bash
  sudo cp local-ca.crt /usr/local/share/ca-certificates/local-ca.crt
  sudo update-ca-certificates
  ```

## Step 3: Inject the CA into your Minikube Cluster
Cert-manager needs access to this key pair to issue domain certificates. Create a Kubernetes TLS secret inside the **cert-manager** namespace we created in previous stage containing your new CA files:
```bash
kubectl create secret tls ca-secret-local \
  --cert=local-ca.crt \
  --key=local-ca.key \
  --namespace=cert-manager
```
## Step 4: Create a CA ClusterIssuer

Now, create a new cluster issuer that references the secret you just uploaded.
Create a file named ca-cluster-issuer.yaml:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuermetadata:
  name: local-ca-cluster-issuer
spec:
  ca:
    secretName: ca-secret-local
```
Apply it to your cluster:
```
kubectl apply -f ca-cluster-issuer.yaml
```

## Step 5: Update your Ingress Annotation
Finally, update the annotation on your application's Ingress manifest to point to your new local-ca-cluster-issuer instead of the old self-signed one:
```yaml
metadata:
  name: my-app-ingress
  annotations:
    # Update this line to use the new Local CA Issuer
    cert-manager.io/cluster-issuer: "local-ca-cluster-issuer" 
```
Apply your updated Ingress manifest (kubectl apply -f <your-ingress-file>.yaml).

## Final Verification

   1. Restart your browser completely to clear out cached SSL state.
   2. Navigate to https://local.com.
   3. The warning should be gone, and you will see a secure connection lock.

-------