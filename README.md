# OpenShift and Amazon Elastic Container Registry Integration

From the [Amazon docs], an authentication token is used to access any Amazon ECR registry and is valid for 12 hours.  This integration refreshes secrets within OpenShift based on this token rotation.

## Initial Setup

1. We will build an image that has both the `oc` and `aws` cli.  We will start by creating a project for this one time build and import an image which contains the `oc` cli.  We'll also set this project to be visible to pull images from other projects.  In my example, I'll use the `aws-oc-build` project, but you can use anything you'd like.
```
$ oc new-project aws-oc-build
$ oc import-image ose-cli-artifacts-alt-rhel8:latest --from=registry.redhat.io/openshift4/ose-cli-artifacts-alt-rhel8:latest --confirm
$ oc adm policy add-role-to-group view system:authenticated
$ oc adm policy add-role-to-group system:image-puller system:authenticated
```

2. We will install the `aws` cli on this image with these [build artifacts]:
```
oc apply -f aws-oc-cli.artifacts.yaml
```

3. In a new namespace from which you'd like to build and push application images to ECR, download the following [ECR artifacts].

Update the `AWS_ACCOUNT`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY` values in the ecr-creds-rotator-secret Secret, and update the `AWS_REGION` value in the ecr-creds-rotator-cm ConfigMap.

Then create the artifacts:
```
oc apply -f ecr.artifacts.yaml
```

4. Alternatively, you can configure a [project template] to automatically create these artifacts for new projects.  Here is a [sample template].
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

5. Now let's test a build.

In ECR, create a new repository named `<AWS_ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com/sample-ecr:latest`
In OpenShift, create a new namespace and create this [sample BuildConfig], replacing the values of `<AWS_ACCOUNT>.dkr.ecr.<AWS_REGION>.amazonaws.com/sample-ecr:latest` with your AWS account and region.

```
oc create -f sample.buildconfig.yaml
```

6. Optional cleanup for testing purposes.  If you'd like to delete the artifacts created in step 3 or 4 in a new project, you can run the following:
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

[Amazon docs]: https://docs.aws.amazon.com/AmazonECR/latest/userguide/registry_auth.html
[build artifacts]: aws-oc-cli.artifacts.yaml
[ECR artifacts]: ecr.artifacts.yaml
[project template]: https://docs.openshift.com/container-platform/latest/applications/projects/configuring-project-creation.html
[sample template]: ecr.template.yaml
[sample BuildConfig]: sample.buildconfig.yaml

