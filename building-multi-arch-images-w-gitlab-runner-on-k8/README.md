1. Add containerized_buildah profile to apparmor
   copy containerized_buildah file to /etc/apparmor.d/containerized_buildah
2. Restart apparmour
   `$ service apparmour restart`
3. Install fuse device plugin
   `$ kubectl apply -f fuse-device-plugin.yaml`
4. Get deployment name from gitlab runner helm chart (Cmd must be run in same folder as kustomize.yaml)
   `$ kubectl kustomize . --enable-helm | less`
5. Place deployment name from step 4 into Line 22 in kustomization.yaml
6. Preview and create runner registration token secret
   ```
   # Preview
   $ kubectl -n <-insert-namespace-here-> create secret generic <-insert-secret-name-here-> \
    --from-literal=runner-registration-token=<-insert-registration-token-here-> \
    --from-literal=runner-token= --dry-run=client -o yaml

   # Create
   $  kubectl -n <-insert-namespace-here-> create secret generic <-insert-secret-name-here-> \
    --from-literal=runner-registration-token=<-insert-registration-token-here-> \
    --from-literal=runner-token=
    ```
7. Edit values.yaml
   - Line 16: add secret name from step 6
   - Line 18: add any relivant tags 
   - Line 19: name your runner
8. Preview and deploy runner
   ```
   # Preview
   kubectl kustomize . --enable-helm | less
   # Deploy
   $ kubectl kustomize . --enable-helm | kubectl apply -f - 
   ```