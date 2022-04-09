# OpenShift and Amazon Elastic Container Registry Integration

## Background

*** **NOTE TO READER: THIS IS NOT A RED HAT OR AMAZON SUPPORTED SOLUTION** ***

From the [Amazon docs], an authentication token is used to access any Amazon ECR registry and is valid for 12 hours.  At the time of this writing, if the token is configured as a secret in OpenShift, there is no native way to refresh the secret.  In this integration, we will create a cronjob that periodically pulls the latest token used by ECR and refreshes the secret within a given OpenShift project.

## Infrastructure

* Tested with Red Hat OpenShift on AWS (ROSA), but solution applies generically to OpenShift 4
* Tested with OpenShift 4.9 and 4.10
* Amazon Elastic Container Registry (ECR)

## Initial Setup

1. Create an AWS user

In Amazon ECR, create an AWS user for API access to ECR, and assign it the `AmazonEC2ContainerRegistryPowerUser` policy.  Save the user's credentials for use at a later time, in step 3.

2. Build an image with oc and aws cli

We will build an image that has both the oc and aws command line interface (cli).  We will start by creating a project for this one time build and import an image which contains the oc cli.  We'll also set this project to be visible for other projects to pull this image.  In my example, I'll use the `aws-oc-build` project, but you can use anything you'd like.

```
$ oc new-project aws-oc-build
$ oc import-image ose-cli-artifacts-alt-rhel8:latest --from=registry.redhat.io/openshift4/ose-cli-artifacts-alt-rhel8:latest --confirm
$ oc adm policy add-role-to-group view system:authenticated
$ oc adm policy add-role-to-group system:image-puller system:authenticated
```

We will install the aws cli on this image with these build artifacts: [aws-oc-cli.artifacts.yaml]
```
oc apply -f aws-oc-cli.artifacts.yaml
```

3. Apply the secret rotation artifacts

In a new namespace from which you'd like to build and push application images to ECR, download the following ECR artifacts: [ecr.artifacts.yaml]

Update the `AWS_ACCOUNT`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY` values in the ecr-creds-rotator-secret Secret, and update the `AWS_REGION` value in the ecr-creds-rotator-cm ConfigMap.

Then create the artifacts:
```
oc apply -f ecr.artifacts.yaml
```

4. Configure a project template

As an alternative to step 3, you can configure a [project template] to automatically create these artifacts for new projects.  Here is a sample template: [ecr.template.yaml]
```
$ oc create -f ecr.template.yaml -n openshift-config
$ oc edit project.config.openshift.io/cluster

apiVersion: config.openshift.io/v1
kind: Project
metadata:
  ...
spec:
  projectRequestTemplate:
    name: project-request
```

5. Now let's test a build

In ECR, create a new repository named `<AWS_ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com/sample-ecr:latest`
In OpenShift, create a new namespace and create this sample BuildConfig: [sample.buildconfig.yaml].  Replace the values of `<AWS_ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com/sample-ecr:latest` with your AWS account and region.

```
oc create -f sample.buildconfig.yaml
```

6. Optional cleanup for testing purposes

If you'd like to delete the artifacts created in step 3 or 4 in a new project, you can run the following:
```
oc delete secret ecr-creds-rotator-secret
oc delete cm ecr-creds-rotator-cm
oc delete cronjob ecr-creds-rotator
oc delete role role-update-secrets
oc delete rolebinding ecr-creds-rotator-rolebinding
oc delete sa ecr-creds-rotator-sa
oc delete secret ecr-secret
```

## License
GPLv3

## Author
Kevin Chung

[Amazon docs]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html
[aws-oc-cli.artifacts.yaml]: aws-oc-cli.artifacts.yaml
[ecr.artifacts.yaml]: ecr.artifacts.yaml
[project template]: https://docs.openshift.com/container-platform/latest/applications/projects/configuring-project-creation.html
[ecr.template.yaml]: ecr.template.yaml
[sample.buildconfig.yaml]: sample.buildconfig.yaml

