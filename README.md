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

Once running, the application processes will stay resident and running until the container is shutdown. If you want to enable periodic restarts of the application process, but keep the container running, you can set a restart interval.

```
oc set env dc/ctfd MOD_WSGI_RESTART_INTERVAL=60
```

The container in which CTFd runs has a memory limit of 256Mi. If you need to scale up the application by increasing the number of worker processes, check that there is still sufficient memory and adjust the memory limit for the ``ctfd`` container if necessary.

Application Security
--------------------

The CTFd application is used for tracking progress in contests where the aim of the contest can be to hack into special prepared applications exhibiting some security vulnerability. As participants are potentially going to be well versed in application security and hacking, it is  important also that CTFd be secure.

Since this repository only concerns itself with deployment of CTFd, it cannot vouch for how secure CTFd is, and whether someone could break into the application, access the database and make updates or steal answers to challenges. What can be done is ensure that if someone is able to break into the CTFd application, that they aren't then able to access other applications deployed for the competition, or access OpenShift or the underlying host itself.

A description of the problems that might arise when running a capture the flag event on Kubernetes can be found in:

* https://hackernoon.com/capturing-all-the-flags-in-bsidessf-ctf-by-pwning-our-infrastructure-3570b99b4dd0

Although OpenShift is Kubernetes, it adds additional security on top of Kubernetes which would have circumvented the issues described in that post. These include the default implementation of a secure role based access control (RBAC) configuration which prevents access to the Kubernetes REST API and internal image registry.

Although a default OpenShift would have helped in that situation, the deployments using the build scripts and templates provided here take additional steps to further try and avoid issues arising. These are:

* A separate service account is created for the CTFd application and the ``default`` service account is not used. The ``default`` service account doesn't have any special access under OpenShift, but by creating a separate service account for the deployment, also with no special access, it avoids problems caused by the ``default`` service account for a project accidentally being granted additional roles by a project admin.
* The service account information, including the access token is prohibited from being mounted into any application. This is done in the service account resource, but is also blocked in the pod template for the deployment.
* The application runs as a unique user ID assigned to the project it is deployed in. This comes from the default security policy of OpenShift. The application is not run as ``root``.
* The container image for CTFd on Docker Hub is not used as it is designed to run as ``root``. Instead, the container image is built inside of OpenShift using a non privileged Source-to-Image (S2I) builder, against a specific tagged version of the CTFd source code available on GitHub.
* A custom S2I build script is used to ensure that the application directory containing the application source code and installed Python packages are not writable to the running application. Normally a S2I builder would fix up permissions so that these directories would be writable, but the custom build script reverts the write access. This avoids the possibility of someone breaking into the application, rewriting source code, and triggering a restart of the web server processes to reload the modified Python source code.
* Once the CTFd application data has been loaded, a restart interval can be specified for application processes so they are periodically restarted. This could be used if concerned that someone might break into the CTFd application processes and modify in process Python objects. The restart would periodically reset the process and discard such changes making it more difficult to install an in memory modification.

For separate applications deployed as part of a challenge, the following is recommended.

* Deploy each application into a separate OpenShift project.
* Use a separate service account for each application, don't use the ``default`` service account.
* Prohibit mounting of the service account details into application containers.
* Store flag values as secrets and mount them into the application container, do not embed the flag value in the container image.
