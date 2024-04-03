# build-docker-image-action

Composite Actions | Build Docker image

## Summary

This Composite Action does the following:

1. Builds a Docker image with a custom command (including all the target tags)
2. (Optional) Tests the image with [container-structure-test](https://github.com/GoogleContainerTools/container-structure-test)
3. (Optional) Pushes the image and all its tag to an ECR registry

## Requirements

- For tests: a valid [container-structure-test](https://github.com/GoogleContainerTools/container-structure-test) config file (binary installed in the action)
- For ECR push: AWS credentials with permission to push to the ECR registry

## Examples of use

### Build & test image only

```yaml
jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Gutenberg-Technology/build-docker-image-action@v1.0.0
        with:
          build-cmd: "docker build -t my-image:latest -t my-image:${{ github.sha }} ."
          test-cmd: "container-structure-test test --image my-image --config test-config.yaml"
```

### Build, test & push image

Basic example: build, test and push the `gt/abn/keycloak/latest` tag to an ECR registry.

```yaml
jobs:
  build-test-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: Gutenberg-Technology/build-docker-image-action@v1.0.0
        with:
          build-cmd: "docker build --no-cache -t gt/abn/keycloak:latest ."
          test-cmd: "container-structure-test test --image gt/abn/keycloak:latest --config container_structure_test.yaml"
          push-to-ecr: true
          ecr-repo: "gt/abn/keycloak"
          ecr-tags-to-push: "latest"
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: "us-east-1"
```

Advanced usage: Set `env` vars, use several tags, including the commit SHA, and push it to the ECR registry.

```yaml
jobs:
  build-test-push:
      name: Build, test & push image
      runs-on: ubuntu-latest
      env:
        KEYCLOAK_REGISTRY: "gt/abn/keycloak-mef"

      steps:
        - name: Checkout code
          uses: actions/checkout@v4
          with:
            ref: ${{ inputs.git-ref }}

        - name: Set short git commit SHA
          id: vars
          run: |
            shortSha=$(git rev-parse --short ${{ github.sha }})
            echo "COMMIT_SHORT_SHA=$shortSha" >> $GITHUB_ENV
          
        - name: Confirm git commit SHA output
          run: echo ${{ env.COMMIT_SHORT_SHA }}

        - uses: Gutenberg-Technology/build-docker-image-action@v1.0.0
            build-cmd: "docker build --no-cache -t ${{ env.KEYCLOAK_REGISTRY }}:latest -t ${{ env.KEYCLOAK_REGISTRY }}:${{ env.COMMIT_SHORT_SHA }} ."
            test-cmd: "container-structure-test test --image ${{ env.KEYCLOAK_REGISTRY }}:${{ env.COMMIT_SHORT_SHA }} --config container_structure_test.yaml"
            push-to-ecr: true
            ecr-repo: "${{ env.KEYCLOAK_REGISTRY }}"
            ecr-tags-to-push: "latest,${{ env.COMMIT_SHORT_SHA }}"
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: "us-east-1"
```