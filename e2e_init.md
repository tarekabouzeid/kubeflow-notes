# Step 1 
Get admin credentials from the system-level secret (kubeflow namespace) (FYI: This is a workaround until fixing seaweedfs IAM )
These are the SeaweedFS admin keys that have full bucket-level access
```bash
kubectl get secret mlpipeline-minio-artifact -n kubeflow \
  -o jsonpath='{.data.accesskey}' | base64 -d && echo   # → minio
kubectl get secret mlpipeline-minio-artifact -n kubeflow \
  -o jsonpath='{.data.secretkey}' | base64 -d && echo   # → minio123
 ```
# Step 2
Create the KServe S3 secret with admin credentials
```bash
kubectl create secret generic kserve-s3-secret \
  --from-literal=AWS_ACCESS_KEY_ID=minio \
  --from-literal=AWS_SECRET_ACCESS_KEY=minio123 \
  -n kubeflow-user-example-com
 ```

# Step 3
Annotate the secret with S3 endpoint config
s3-usevirtualbucket="0" is required — SeaweedFS only supports path-style
Without it, the storage-initializer uses virtual-hosted style which fails
```bash
kubectl annotate secret kserve-s3-secret \
  -n kubeflow-user-example-com \
  serving.kserve.io/s3-endpoint="seaweedfs.kubeflow.svc.cluster.local:9000" \
  serving.kserve.io/s3-usehttps="0" \
  serving.kserve.io/s3-region="minio" \
  serving.kserve.io/s3-usevirtualbucket="0"
 ```
# Step 4 
Annotate the service account so the pod-mutator webhook
knows which secret to inject into the storage-initializer
```bash
kubectl annotate serviceaccount default-editor \
  -n kubeflow-user-example-com \
  serving.kserve.io/s3-secret-name=kserve-s3-secret \
  --overwrite
 ```
# Step 5
Add the secret to the SA's secrets[] list
Required in Kubernetes 1.24+ (no auto-population of secrets[])
sso the KServe webhook actually mounts the secret into the pod
```bash
kubectl patch serviceaccount default-editor \
  -n kubeflow-user-example-com \
  --type=json \
  -p='[{"op":"add","path":"/secrets/-","value":{"name":"kserve-s3-secret"}}]'
 ```
