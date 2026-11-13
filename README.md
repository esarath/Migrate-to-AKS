# Migrate to AKS

Deployment assets for modernizing Spring PetClinic and running it on Azure
Kubernetes Service (AKS) with Azure Database for PostgreSQL Flexible Server.
The workflow uses Microsoft Entra ID authentication, AKS Workload Identity,
and Azure Service Connector; it does not put a database password in the
deployment.

## Prerequisites

- An Azure subscription with permission to create AKS, PostgreSQL Flexible
  Server, managed identity, and Azure Container Registry resources.
- Azure CLI, `kubectl`, Docker, Java 17 or 21, Maven 3.8+, Git, and Visual
  Studio Code with the Java and GitHub Copilot extensions.
- A modernized PetClinic application configured for PostgreSQL passwordless
  authentication and an Azure Container Registry image.

## Deploy

1. Follow the [Migrate to AKS workshop](https://azure-samples.github.io/aks-labs/docs/migration/migrate-to-aks/)
   to prepare the local PetClinic app, create Azure resources, configure the
   PostgreSQL Service Connector, modernize the app, and build/push the image.
   Start PetClinic locally using the workshop's setup instructions and confirm
   it is reachable at `http://localhost:8080` before beginning modernization.
2. From the modernized PetClinic project, run its tests, then build and push
   the container image:

   ```sh
   mvn clean test
   az acr login --name "$ACR_NAME"
   az acr build --registry "$ACR_NAME" --image petclinic:0.0.1 .
   ```

3. Before deploying, edit `k8s/petclinic.yaml`:
   - Replace `replace-with-service-connector-account` with the service account
     created by the PostgreSQL Service Connector (find it in `sc.log`).
   - Replace `replace-with-acr-login-server.azurecr.io` with your registry's
     login server.
   - Confirm the image tag and Service Connector secret name/keys match your
     environment.
4. Apply and monitor the deployment:

   ```sh
   kubectl apply -f k8s/petclinic.yaml
   kubectl get pods,services,deployments
   kubectl logs -l app=spring-petclinic
   ```

5. Visit the external address shown for the `petclinic` service, or use port
   forwarding while the load balancer is provisioning:

   ```sh
   kubectl port-forward service/petclinic 8080:80
   curl http://localhost:8080/actuator/health
   ```

Check the pod logs and inspect password-related environment variables. Only
the passwordless-authentication flag should be present; database credentials
should not be:

```sh
kubectl logs -l app=spring-petclinic
kubectl exec deploy/petclinic -- env | grep -i pass
```

The manifest enables Workload Identity on the pod, reads database connection
details and the managed identity client ID from the Service Connector secret,
and configures health checks and resource limits. It intentionally contains
placeholders rather than environment-specific Azure resource names.