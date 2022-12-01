---
layout: post
title:  "Cloud-based Container Building"
date:   2022-11-30 20:18:00 +0000
categories: metricul
---
Many are looking into accelerating the development loops to ship features faster,
and there are a number of components that can be updated along the development to build
pipeline.

One such area I've taken an interest in is cloud-based container building.

There are some very interesting projects out there where the lines between local and remote
are being blurred, making use of higher performance infrastructure, sharing resources, and
having more up-to-date data than a local dev DB as well as supporting microservices. (See
[Telepresence](https://www.telepresence.io/) in the [CNCF](https://www.cncf.io/) as one such
example)

![image](/assets/images/cloud_container_building.jpg)

So currently I have been investigating [Kaniko](https://github.com/GoogleContainerTools/kaniko),
a tool to build container images from a Dockerfile, inside a container or Kubernetes cluster, and
I have to say it looks awesome!

It seamlessly combines building a container and pushing to a registry with a simple k8s pod
declaration (there are other ways to run kaniko as well).

Create the AWS credentials secret on your k8s cluster (if k8s is running on AWS, you may want to
use instance roles instead):
```bash
kubectl create secret generic aws-secret --from-file=~/.aws/credentials
```

Running the pod definition:
```bash
kubectl apply -f kaniko.yml
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kaniko-{{ uuid }}
spec:
  containers:
    - name: kaniko-{{ uuid }}
      image: gcr.io/kaniko-project/executor:latest
      imagePullPolicy: IfNotPresent
      args:
        - "--dockerfile={{ container_file }}"
        - "--context={{ context }}"
        - "--destination={{ destination }}"
      env:
        - name: AWS_REGION
          value: eu-west-2  # required or it will break
      volumeMounts:
        - name: aws-secret
          mountPath: /root/.aws/
  restartPolicy: Never
  volumes:
    # when not using instance role
    - name: aws-secret
      secret:
        secretName: aws-secret

```

What I've referenced above is the [`handlebars`](https://handlebarsjs.com/) template I've used to
get this to work (the reason being `handlebars` is a supported templating engine in the
[rocket](https://rocket.rs/), the backend API I'm currently working on as well).

With the AWS creds in place on a k8s cluster, this pod can take a tarball on S3 with code and a
Dockerfile, build the container, and push to AWS EKS; all in one command.

The next trick will be getting logs back easily to end users running these builds...
