CTFd quickstart for OpenShift
=============================

This repository contains build/run scripts and templates for deploying the CTFd (Capture the Flag) software on OpenShift.

The original repository for CTFd can be found at:

* https://github.com/CTFd/CTFd

This repository is not a fork, nor does it make changes to the original source code of the CTFd application. The building of CTFd for OpenShift works by injecting into the Source-to-Image (S2I) build process, custom ``assemble`` and ``run`` scripts from this repository.

The container images for CTFd available on Docker Hub are not used as they require the image be run as ``root``.

To make the process of building and deployment of CTFd easy, OpenShift templates are provided. The templates currently use version 1.2.0 of CTFd.

Two separate templates are provided.

* templates/all-in-one-deployment.json - This deploys CTFd, Redis and MySQL altogether in the one pod sharing a single persistent volume. This template can be used in OpenShift Online Starter, where no other applications have already been deployed.
* templates/distinct-deployments.json - This deploys CTFd, Redis and MySQL using separate deployments, with each in their own pods. Because separate deployments are used, a separate persistent volume is required for each.

With both templates, the CTFd pod should not be scaled to multiple replicas. If you need to increase capacity, the application should be scaled up by overriding the number of worker processes used by ``mod_wsgi-express``.

Creating the deployment
-----------------------

To deploy CTFd, use ``oc new-app``, passing it the URL to the template.

```
oc new-app https://raw.githubusercontent.com/openshift-evangelists/ctfd-quickstart/master/templates/all-in-one-deployment.json
```

To determine the public URL for the application run ``oc get routes``.

The default name for the deployed application is ``ctfd``. If you want to override the name of the deployed application, pass the ``APPLICATION_NAME`` parameter for the template.

```
oc new-app https://raw.githubusercontent.com/openshift-evangelists/ctfd-quickstart/master/templates/all-in-one-deployment.json --param APPLICATION_NAME=ctfd
```

The Git repository URL and Git reference can be overridden using the ``GIT_REPOSITORY_URL`` and ``GIT_REFERENCE`` template parameters.

Scaling up the application
--------------------------

The default configuration for ``mod_wsgi-express``, used to host CTFd, should be sufficient for most scenarios. If you want to adjust the capacity of the web server, you can set environment variables to control the number of worker processes and threads used by ``mod_wsgi-express``.

```
oc set env dc/ctfd MOD_WSGI_PROCESSES=5 MOD_WSGI_THREADS=3
```

Because CTFd is implemented in Python, and due to the concurrency issues of the GIL, increasing the number of threads to increase capacity is not recommended, use additional processes instead. A maximum of 5 threads per process is recommended. The default configuration is a single worker process with 5 threads.

The container in which CTFd runs has a memory limit of 256Mi. If you need to scale up the application by increasing the number of worker processes, check that there is still sufficient memory and adjust the memory limit for the ``ctfd`` container if necessary.
