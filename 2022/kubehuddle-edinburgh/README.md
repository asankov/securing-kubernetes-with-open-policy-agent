# KubeHuddle, Edinburgh 2022

These are the [slides](./presentation.pdf) and [manifests](./k8s/) from presenting this presentation at [KubeHuddle Edinburgh 2022](https://kubehuddle.com/2022/).

## Demo

To reproduce the demo:

1. [Install Gatekeeper](https://asankov.dev/blog/2022/04/21/securing-kubernetes-with-open-policy-agent/#installing-gatekeeper)
2. Apply the [`ConstraintTemplate`](k8s/constraint-template.yaml) resource

   ```shell
   kubectl apply -f k8s/constraint-template.yaml
   ```

3. Apply the [`Constraint`](k8s/constraint.yaml) resource

   ```shell
   kubectl apply -f k8s/constraint.yaml
   ```

4. Try to create the [deployments that violate the Policy](k8s/non-compliant-deployments.yaml) (Should fail)

   ```shell
   kubectl apply -f k8s/non-compliant-deployments.yaml
   ```

5. Try to create the deployment that does not violate the Policy (Should pass)

   ```shell
   kubectl apply -f k8s/compliant-deployment.yaml
   ```

6. Bonus: Apply the modified [`ConstraintTemplate`](k8s/constraint-template-2.yaml) and repeat the demo to make sure it still works

   ```shell
   kubectl apply -f k8s/constraint-template-2.yaml
   ```
