# Gitlab Runner for K8s
This project covers how to deploy the custom GitLab Runner via Helm and how to configure your project's CI/CD pipeline to fetch secrets from CyberArk Conjur at runtime.

## 1. Standing Up the Runner Deployment
Deploy the GitLab Runner to your Kubernetes cluster using the provided Helm chart. Ensure you have your GitLab registration token and Conjur URL ready.

  1.  Navigate to the directory containing the gitlab-runner-conjur Helm chart.

  2.  Run the following command to deploy the runner, replacing the placeholder values with your actual configuration:

  ```bash
  helm install my-conjur-runner ./gitlab-runner-conjur \
  --namespace default \
  --set gitlab.url="https://gitlab.yourcompany.com/" \
  --set gitlab.registrationToken="YOUR_GITLAB_REGISTRATION_TOKEN" \
  --set conjur.applianceUrl="https://<subdomain>.secretsmgr.cyberark.cloud/api" \
  --set conjur.account="conjur"
  ```

Once deployed, the runner will automatically register itself with your GitLab instance and start listening for jobs tagged with conjur-demo.

## 2. Retrieving Job Secrets at Runtime

To use the runner in your project and fetch secrets dynamically during a pipeline run, configure your .gitlab-ci.yml file to use GitLab's native id_tokens alongside the CyberArk helper image.

Add the following job definition to your project's .gitlab-ci.yml:

```yaml
fetch-conjur-secrets:
  # Targets the Helm-deployed runner
  tags:
    - conjur-demo
  
  # Generates a JWT for Conjur authentication
  id_tokens:
    GITLAB_OIDC_TOKEN:
      aud: conjur

  # Uses the CyberArk helper image to process the JWT
  image: cyberark/authn-jwt-gitlab:alpine-1.0.0

  variables:
    CONJUR_AUTHN_JWT_TOKEN: $GITLAB_OIDC_TOKEN
    CONJUR_AUTHN_JWT_SERVICE_ID: "gitlab" 

  script:
    # Retrieve secrets by assigning the Conjur path to CONJUR_SECRET_ID and executing the helper binary
    - export DB_USERNAME=$(CONJUR_SECRET_ID="data/my-project/db-username" /authn-jwt-gitlab)
    - export DB_PASSWORD=$(CONJUR_SECRET_ID="data/my-project/db-password" /authn-jwt-gitlab)
    
    # The secrets are now available as local environment variables for your workflow
    - echo "Successfully retrieved secrets for $DB_USERNAME."
    - ./deploy.sh --user "$DB_USERNAME" --password "$DB_PASSWORD"
```