# How to deploy AMQ7 on OpenShift using S2I

Configuration of the AMQ7 image can also be modified using the S2I (Source-to-image) feature.

Custom AMQ broker configuration can be specified by creating an broker.xml file inside the git directory of your application’s Git project root. On each commit, the file will be copied to the conf directory in the AMQ root and its contents used to configure the broker.

Create first a new project (namespace) called Broker:
```
oc new-project broker
```
Now let's s2i our conf to the jboss-amq image

```
$ oc new-build registry.redhat.io/amq7/amq-broker:7.6~https://github.com/avi5kdonrh/amq7-s2i.git
--> Found Docker image f554b48 (5 months old) from registry.redhat.io for "registry.redhat.io/amq7/amq-broker:7.6"

    Red Hat AMQ Broker 7.6.0 
    ------------------------ 
    A reliable messaging platform that supports standard messaging paradigms for a real-time enterprise.

    Tags: messaging, amq, java, jboss, xpaas

    * An image stream tag will be created as "amq-broker:7.6" that will track the source image
    * A source build using source code from https://github.com/avi5kdonrh/amq7-s2i.git will be created
      * The resulting image will be pushed to image stream tag "amq7-s2i:latest"
      * Every time "amq-broker:7.6" changes a new build will be triggered

--> Creating resources with label build=amq7-s2i ...
    imagestream.image.openshift.io "amq-broker" created
    imagestream.image.openshift.io "amq7-s2i" created
    buildconfig.build.openshift.io "amq7-s2i" created
--> Success
```

To stream the build progress, run 'oc logs -f bc/amq7-s2i', You can see here that our conf broker.xml has been copied to the image stream:

```
$ oc logs -f bc/amq7-s2i
Cloning "https://github.com/avi5kdonrh/amq7-s2i.git" ...
	Commit:	e800e4392d4d224fbe8a1e6c67dbae5a417e901d (new changes)
	Author:	avi5kdon <avi5kdon@gmail.com>
	Date:	Fri Sep 18 20:32:00 2020 +0530
Caching blobs under "/var/cache/blobs".
Getting image source signatures
Copying blob sha256:455ea8ab06218495bbbcb14b750a0d644897b24f8c5dcf9e8698e27882583412
..........
Getting image source signatures
Copying blob sha256:f985fa3adb1cc0d6179ecbaa77f28edc528319932bddf0da495ec66849650b56
............
Copying config sha256:1d031df63b7abe51b001fa7816d261f238a0a818ecb0bac34769a002611caa25
Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/amq7/amq7-s2i@sha256:c1af498169a2e76f0f72deb17b3041387f090d51e6beaf2445e5dceef09350fd
Push successful

```

Create the service account "amq-service-account"
```
echo '{"kind": "ServiceAccount", "apiVersion": "v1", "metadata": {"name": "amq-service-account"}}' | oc create -f -
serviceaccount "amq-service-account" created
```

Ensure the service account is added to the namespace for view permissions... (for pod scaling)
```
oc policy add-role-to-user view system:serviceaccount:broker:amq-service-account
```

Install templates in the namespace broker:
```
for template in amq-broker-76-basic.yaml \
amq-broker-76-ssl.yaml \
amq-broker-76-custom.yaml \
amq-broker-76-persistence.yaml \
amq-broker-76-persistence-ssl.yaml \
amq-broker-76-persistence-clustered.yaml \
amq-broker-76-persistence-clustered-ssl.yaml;
 do
 oc replace --force -f \
https://raw.githubusercontent.com/jboss-container-images/jboss-amq-7-broker-openshift-image/amq-broker-76-dev/templates/${template}
 done
```

Let's get now, our image stream URL:

```
$ oc get is
NAME         IMAGE REPOSITORY                                                   TAGS      UPDATED
amq-broker   image-registry.openshift-image-registry.svc:5000/amq7/amq-broker   7.6       12 minutes ago
amq7-s2i     image-registry.openshift-image-registry.svc:5000/amq7/amq7-s2i     latest    9 minutes ago

```

Specify the image (image-registry.openshift-image-registry.svc:5000/amq7/amq7-s2i) in one of the templates installed above in the parameter 'IMAGE':
For instance:

```
oc process amq-broker-76-basic -p APPLICATION_NAME=broker -p AMQ_NAME=broker -p AMQ_USER=user -p AMQ_PASSWORD=password -p AMQ_PROTOCOL=openwire,amqp,stomp,mqtt,hornetq -p IMAGE_STREAM_NAMESPACE=broker -p IMAGE=image-registry.openshift-image-registry.svc:5000/amq7/amq7-s2i -n broker | oc create -f -
```

Update of broker.xml

You should setup the GitHub webhook URL in order to trigger a new build after each update.

You can also do it manually, like following:
```
$ git commit -am "changing my broker conf"

$ git push -u origin master

$ oc start-build amq7-s2i -n broker
build "amq7-s2i-2" started

$ oc deploy broker-amq --latest -n broker
Started deployment #2
Use 'oc logs -f dc/broker-amq' to track its progress.
```
