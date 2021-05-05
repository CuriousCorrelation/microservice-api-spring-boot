# microservice-api-spring-boot
[![Build Status](https://github.com/jonashackt/microservice-api-spring-boot/workflows/build-publish-deploy/badge.svg)](https://github.com/jonashackt/microservice-api-spring-boot/actions)

Example project showing how to interact with a Nuxt.js / Vue.js based frontend building a Spring Boot microservice 

The source is simply copied from my project https://github.com/jonashackt/spring-boot-vuejs/tree/master/backend

## GitHub Actions: Build, create Container Image with Paketo.io & Publish to GitHub Container Registry

Our [build-publish-deploy.yml](.github/workflows/build-publish-deploy.yml) already builds our Java/Spring Boot app using JDK 16. It also uses Paketo.io / Cloud Native Buildpacks to create a Container image (incl. a Paketo cache image to speed up builds, see https://stackoverflow.com/a/66598693/4964553) and publish it to GitHub Container Registry:

```yaml
name: build-publish-deploy

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - name: Set up JDK 16
        uses: actions/setup-java@v1
        with:
          java-version: 16

      - name: Build with Maven
        run: mvn package --batch-mode --no-transfer-progress

  publish-to-gh-container-registry:
    needs: build
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v2

      - name: Login to GitHub Container Registry
        uses: docker/login-action@v1
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Install pack CLI via the official buildpack Action https://github.com/buildpacks/github-actions#setup-pack-cli-action
        uses: buildpacks/github-actions/setup-pack@v4.1.0

      # Caching Paketo Build see https://stackoverflow.com/a/66598693/4964553
      # BP_OCI_SOURCE as --env creates the GitHub Container Registry <-> Repository link (https://paketo.io/docs/buildpacks/configuration/#applying-custom-labels)
      # BP_JVM_VERSION 16, because we use Java 16 inside our Maven build but Paketo defaults to 11
      - name: Build app with pack CLI & publish to bc Container Registry
        run: |
          pack build ghcr.io/jonashackt/microservice-api-spring-boot:latest \
              --builder paketobuildpacks/builder:base \
              --path . \
              --env "BP_OCI_SOURCE=https://github.com/jonashackt/microservice-api-spring-boot" \
              --env "BP_JVM_VERSION=16" \
              --cache-image ghcr.io/jonashackt/microservice-api-spring-boot-paketo-cache-image:latest \
              --publish
```

## GitHub Actions: Deploy to AWS Fargate using Pulumi

As already described here: https://github.com/jonashackt/azure-training-pulumi#pulumi-with-github-actions there are some steps to take in order to use Pulumi with GitHub Actions.

https://www.pulumi.com/docs/guides/continuous-delivery/github-actions/

It's really cool to see that there's a Pulumi GitHub action project https://github.com/pulumi/actions already ready for us.


#### Create needed GitHub Repository Secrets

First we need to create 5 new GitHub Repository Secrets (encrypted variables) in your repo under `Settings/Secrets`.

We should start to create a new Pulumi Access Token `PULUMI_ACCESS_TOKEN` at https://app.pulumi.com/jonashackt/settings/tokens

Now we need to create the AWS specific variables: `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` (they need to be exactly named like this, see https://www.pulumi.com/docs/intro/cloud-providers/aws/setup/#environment-variables). Create them all as GitHub Repository Secrets.


#### Create GitHub Actions workflow

Let's add a job `deploy-to-aws-fargate` to our GitHub Actions workflow [build-publish-deploy.yml](.github/workflows/build-publish-deploy.yml):

```yaml
  deploy-to-aws-fargate:
    needs: publish-to-gh-container-registry
    runs-on: ubuntu-latest
    env:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}
    steps:
      - name: Checkout
        uses: actions/checkout@master

      - name: Setup node env
        uses: actions/setup-node@v2.1.2
        with:
          node-version: '14'

      - name: Cache node_modules
        uses: actions/cache@v2
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
          restore-keys: |
            ${{ runner.os }}-node-

      - name: Install Pulumi dependencies before npm run generate to prevent it from breaking the build
        run: npm install
        working-directory: ./deployment

      - name: Install Pulumi CLI
        uses: pulumi/action-install-pulumi-cli@v1.0.2

      - name: Run pulumi preview & pulumi up
        run: |
          pulumi stack select dev
          pulumi preview
          pulumi up -y
        working-directory: ./deployment

      - name: Configure AWS credentials for GitHub pre-installed AWS CLI
        uses: aws-actions/configure-aws-credentials@v1
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-central-1

      - name: Deploy Spring Boot app to AWS Fargate
        run: |
          echo "deploy our app here"
          echo "Access the Nuxt.js app at the following URL:"
          pulumi stack output bucketUrl
        working-directory: ./deployment
```

We use the possibility [to define the environment variables on the workflow's top level](https://docs.github.com/en/actions/reference/environment-variables) to reduce the 3 definition to one. 
