# Non-root deployment

This example shows how to use Open Policy Agent and Gatekeeper to enforce that all Deployments in your cluster will run as non-root containers.

1. [Install Gatekeeper](https://asankov.dev/blog/2022/04/21/securing-kubernetes-with-open-policy-agent/#installing-gatekeeper)
2. Apply the [`ConstraintTemplate`](k8s/constraint-template.yaml) resource

   ```shell
   kubectl apply -f k8s/constraint-template.yaml
   ```

3. Apply the [`Constraint`](k8s/constraint.yaml) resource

   ```shell
   kubectl apply -f k8s/constraint.yaml
   ```

4. Try to create the [deployments that violate the Policy](k8s/non-compliant-deployment.yaml) (Should fail)

   ```shell
   kubectl apply -f k8s/non-compliant-deployment.yaml
   ```

5. Try to create the deployment that does not violate the Policy (Should pass)

   ```shell
   kubectl apply -f k8s/compliant-deployment.yaml
   ```
