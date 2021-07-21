# **Configure a Disconnected Registry and RHEL CoreOS Cache**

In this lab we will explore what has been a popular topic when it comes to OpenShift: the disconnected installation. A disconnected installation is one where the master and worker nodes do not have access to the internet. Thus the Red Hat Enterprise Linux CoreOS (RHCOS) images and the OpenShift pod images need to be hosted locally to support the given installation.  The AIO lab when deployed without deploying the OpenShift cluster does not have an option for automatically setting up a disconnected environment.  However this lab can provide the needed steps and insights into what that process looks like.

The current AIO lab already has some of the components needed for a disconnected installation so lets first explore those. The first component needed is the oc client. If we type `oc version` at the command line we can see what version we have. 

And then check the version:
~~~bash
[root@ocp4-bastion ~]# oc version
Client Version: 4.7.19
Server Version: 4.7.19
Kubernetes Version: v1.20.0+87cc9a4
~~~

The output above shows us we have the 4.5.12 client for oc. This will determine what version of the OpenShift cluster we will deploy. Lets go ahead and set the environment variable of VERSION to that version now:

~~~bash
[root@ocp4-bastion ~]# export VERSION=4.7.19
[root@ocp4-bastion ~]# echo $VERSION
4.7.19
~~~

Before we move further along lets make sure we have the correct version of the Openshift baremetal installer in our home directory.  To do that we will need to set the version variable and extract it from the remote registry:

~~~bash
[root@ocp4-bastion ~]# export VERSION=4.7.19
[root@ocp4-bastion ~]# oc adm release extract --registry-config /root/pull-secret.json --command=openshift-baremetal-install --to /root $VERSION
~~~

Now lets examine the version of the openshift-baremetal-install binary version. The **commit number** is important as that will be used to determine what version of the RHCOS image is pulled down later on in the lab.

~~~bash
[root@ocp4-bastion ~]# ~/openshift-baremetal-install version
/root/openshift-baremetal-install 4.7.19
built from commit 3948d1fe9f04057823b871133085d602e6575067
release image quay.io/openshift-release-dev/ocp-release@sha256:eafdac268e1f65053de423ba4a028e8de5133ab78e7954d76ed838bcf5f4f666
~~~

Now that we have examined the oc and openshift-baremetal-install binaries we are ready to build our private registry and httpd cache on the provisioning node. In a production environment this registry and httpd cache could be anywhere within the organization but for this lab we will keep it simple and place it on the provisioning node.

The first step is to validate that podman, httpd and httpd-tools are installed.  They should be but its always wise to confirm:

~~~bash
[root@ocp4-bastion ~]# rpm -qa podman httpd httpd-tools podman-docker
httpd-tools-2.4.37-39.module_el8.4.0+778+c970deab.x86_64
httpd-2.4.37-39.module_el8.4.0+778+c970deab.x86_64
podman-3.0.1-7.module_el8.4.0+830+8027e1c4.x86_64
podman-docker-3.0.1-7.module_el8.4.0+830+8027e1c4.noarch
~~~

If by chance those packages are not installed use the following to install them:

~~~bash
[root@ocp4-bastion ~]# sudo dnf -y install podman httpd httpd-tools podman-docker
Last metadata expiration check: 1 day, 0:42:29 ago on Wed 14 Jul 2021 06:25:33 PM UTC.
Package podman-3.0.1-7.module_el8.4.0+830+8027e1c4.x86_64 is already installed.
Package httpd-2.4.37-39.module_el8.4.0+778+c970deab.x86_64 is already installed.
Package httpd-tools-2.4.37-39.module_el8.4.0+778+c970deab.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
~~~

After installing the required packages we also need to open a few ports on the local firewall to enable remote access for the registry and image cache.  These services will run on ports 5000 and 80 respectively.  Lets ago ahead and make the adjustments:

~~~bash
[root@ocp4-bastion ~]# firewall-cmd --zone=public --permanent --add-port=5000/tcp
success
[root@ocp4-bastion ~]# firewall-cmd --zone=public --permanent --add-port=80/tcp
success
[root@ocp4-bastion ~]# firewall-cmd --reload
success
~~~

Now let's create the directories you'll need to run the registry. These directories will be mounted in the container runtime environment for the registry.

~~~bash
[root@ocp4-bastion ~]# mkdir -p /nfs/registry/{auth,certs,data}
[root@ocp4-bastion ~]# ls -l /nfs/registry/
total 0
drwxr-xr-x. 2 root root 6 Jul 15 19:11 auth
drwxr-xr-x. 2 root root 6 Jul 15 19:11 certs
drwxr-xr-x. 2 root root 6 Jul 15 19:11 data
~~~

We also need to create a self signed certificate for the registry:

~~~bash
[root@ocp4-bastion ~]# openssl req -newkey rsa:4096 -nodes -sha256 -keyout /nfs/registry/certs/domain.key -x509 -days 365 -out /nfs/registry/certs/domain.crt -subj "/C=US/ST=NorthCarolina/L=Raleigh/O=Red Hat/OU=Marketing/CN=ocp4-bastion.aio.example.com" -addext "subjectAltName = DNS:ocp4-bastion.aio.example.com" -addext "certificatePolicies = 1.2.3.4"
Generating a RSA private key
...............................................................................................++++
......................................................................++++
writing new private key to '/nfs/registry/certs/domain.key'
-----
~~~

Once the certificate has been created copy it into your home directory and also into the trust anchors on the provisioning node. We will also need to run the update-ca-trust command:

~~~bash
[root@ocp4-bastion ~]# cp /nfs/registry/certs/domain.crt ~/domain.crt
[root@ocp4-bastion ~]# cp /nfs/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
[root@ocp4-bastion ~]# update-ca-trust extract
~~~

Our registry will need a simple authentication mechanism so we will use `htpasswd`. Note that when you try to authenticate to your registry the password being passed has to be in [bcrypt](https://en.wikipedia.org/wiki/Bcrypt) format.

~~~bash
[root@ocp4-bastion ~]# htpasswd -bBc /nfs/registry/auth/htpasswd dummy dummy
Adding password for user dummy
~~~

Now that we have a directory structure, certificate, and a user configured for authentication we can go ahead and create the registry pod. The command below will pull down the pod and mount the appropriate directory mount points we created earlier.

However we will first need to login to [docker.io](https://www.docker.com/increase-rate-limits):

~~~bash
[root@ocp4-bastion ~]# docker login docker.io
Emulate Docker CLI using podman. Create /etc/containers/nodocker to quiet msg.
Username: schmaustech
Password: 
Login Succeeded!
~~~

Then we can create the container:

~~~bash
[root@ocp4-bastion ~]# podman create --name poc-registry --net host -p 5000:5000 \
     -v /nfs/registry/data:/var/lib/registry:z -v /nfs/registry/auth:/auth:z \
     -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry" \
     -e "REGISTRY_HTTP_SECRET=ALongRandomSecretForRegistry" \
     -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v /nfs/registry/certs:/certs:z \
     -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
     -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key docker.io/library/registry:2
Trying to pull docker.io/library/registry:2...
Getting image source signatures
Copying blob 5b94580856e6 done  
Copying blob 363ab70c2143 done  
Copying blob 12008541203a done  
Copying blob 6eda6749503f done  
Copying blob ddad3d7c1e96 done  
Copying config 1fd8e1b0bb done  
Writing manifest to image destination
Storing signatures
Port mappings have been discarded as one of the Host, Container, Pod, and None network modes are in use
5d0f0a92956a1b181b487e71bb05887bd7b9cdaf9de77dc4906553cb3b49ef3b
~~~

Once pod creation is complete we can start the pod:

~~~bash
[root@ocp4-bastion ~]# podman start poc-registry
poc-registry
~~~

Finally, verify the pod is up and running via the podman command:

~~~bash
[root@ocp4-bastion ~]# podman ps 
CONTAINER ID  IMAGE                         COMMAND               CREATED        STATUS             PORTS   NAMES
5d0f0a92956a  docker.io/library/registry:2  /etc/docker/regis...  4 minutes ago  Up 17 seconds ago          poc-registry
~~~

We can further validate the registry is functional by using a curl command and passing the user/password to the registry URL. Note here it's not necessary to use a bcrypt formatted password.

~~~bash
[root@ocp4-bastion ~]# curl -u dummy:dummy -k https://ocp4-bastion.aio.example.com:5000/v2/_catalog
{"repositories":[]}
~~~

Now that our registry pod is up and we have validated that it's working it's time to configure the httpd cache pod which stores our RHCOS images locally. The first step is to create some directory structures and add the appropriate permissions:

~~~bash
[root@ocp4-bastion ~]# export IRONIC_DATA_DIR=/nfs/ocp/ironic
[root@ocp4-bastion ~]# export IRONIC_IMAGES_DIR="${IRONIC_DATA_DIR}/html/images"
[root@ocp4-bastion ~]# export IRONIC_IMAGE=quay.io/metal3-io/ironic:master
[root@ocp4-bastion ~]# mkdir -p $IRONIC_IMAGES_DIR
[root@ocp4-bastion ~]# chown -R "${USER}:users" "$IRONIC_DATA_DIR"
[root@ocp4-bastion ~]# find $IRONIC_DATA_DIR -type d -print0 | xargs -0 chmod 755
[root@ocp4-bastion ~]# chmod -R +r $IRONIC_DATA_DIR
~~~

With the directory structures in place we can now create the caching pod. For this we use the ironic pod that already exists in quay.io.

~~~bash
[root@ocp4-bastion ~]# podman pod create -n ironic-pod
63fd4b75d85ea1bf78e1b3a4c2ab4a27290e8fcd66335c6ced7e000d9ba53d76
~~~

And now run the pod:

~~~bash
[root@ocp4-bastion ~]# podman run -d --net host --privileged --name httpd --pod ironic-pod \
     -v $IRONIC_DATA_DIR:/shared --entrypoint /bin/runhttpd ${IRONIC_IMAGE}
Trying to pull quay.io/metal3-io/ironic:master...
Getting image source signatures
Copying blob 7a0437f04f83 done  
Copying blob b9cff270b3ae done  
(...)
Copying blob 6acc707c4aa8 done  
Copying config 9e82c290c3 done  
Writing manifest to image destination
Storing signatures
5e5a18b8917dd3d638b225934d229609127c7df554fedbb582c135b481b769a0
~~~

Because we ran the **create** command and then a **run** command after it there is no need to actually use podman to start the httpd pod. We can see that it is running by looking at the running pods on the provisioning node:

~~~bash
[root@ocp4-bastion ~]# podman ps
CONTAINER ID  IMAGE                                         COMMAND               CREATED         STATUS             PORTS   NAMES
5d0f0a92956a  docker.io/library/registry:2                  /etc/docker/regis...  11 minutes ago  Up 6 minutes ago           poc-registry
97b804b05471  registry.access.redhat.com/ubi8/pause:latest                        2 minutes ago   Up 43 seconds ago          63fd4b75d85e-infra
5e5a18b8917d  quay.io/metal3-io/ironic:master                                     43 seconds ago  Up 42 seconds ago          httpd
~~~

As shown you should see **httpd** and **poc-registry** running.

Further we can test that our httpd cache is operational by using the `curl` command. If you get a 301 code that is normal since we have yet to actually place any images in the cache.

~~~bash
[root@ocp4-bastion ~]# curl http://ocp4-bastion.aio.example.com/images
<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://ocp4-bastion.aio.example.com/images/">here</a>.</p>
</body></html>
~~~

We are almost ready to do some downloading of images but we still have a few items to tend to. First we need to generate a bcrypt password from our username and password we set on the registry. We can do this by piping them into `base64`:

~~~bash
[root@ocp4-bastion ~]# echo -n 'dummy:dummy' | base64 -w0 && echo
ZHVtbXk6ZHVtbXk=
~~~

The bcrypt password needs to be embedded in a registry secret text file:

~~~bash
[root@ocp4-bastion ~]# cat <<EOF > ~/reg-secret.txt
"ocp4-bastion.aio.example.com:5000": {
    "email": "dummy@redhat.com",
    "auth": "$(echo -n 'dummy:dummy' | base64 -w0)"
}
EOF
~~~

When we first deployed AIO environment we inputted our pull-secret.  However now that we are doing a disconnected installation we need to update that pull-secret that was embedded in our install-config.yaml file.  While we certainly could manually make the change the following steps make the addition of the local registry secret easier to manage.

First we need to pull the current pull-secret out of the install-config.yaml and also remove it from the install-config.yaml:

~~~bash
[root@ocp4-bastion ~]# grep pullSecret ~/lab/install-config.yaml|awk -F'pullSecret: ' '{print $2}'| awk '{print substr($0, 2, length($0) - 2)}' > ~/pull-secret.json
[root@ocp4-bastion ~]# sed -i '/pullSecret:/d' ~/lab/install-config.yaml
~~~

Next we will need to merge the reg-secret.txt file we created with our existing pull-secret.json file we extracted from the install-config.yaml:

~~~bash
[root@ocp4-bastion ~]# cat ~/pull-secret.json |jq ".auths += {`cat ~/reg-secret.txt`}"|tr -d '[:space:]' > ~/merged-pull-secret.json
~~~

Now we should have a merged-pull-secret.json file that contains the original pull-secret along with the new registry secret.   Confirm now that the file looks correct.

If the file looks correct lets go ahead and add it back into our install-config.yaml file by appending it:

~~~bash
[root@ocp4-bastion ~]# awk '{print "pullSecret: '\''"$0}' ~/merged-pull-secret.json | awk '{print $0"'\''"}' >> ~/lab/install-config.yaml
~~~

Verify that the pull secret in `install-config.yaml` now includes the credentials for our local registry.

~~~bash
[root@ocp4-bastion ~]# grep pullSecret ~/lab/install-config.yaml | sed 's/^pullSecret: //' | tr -d \' | jq .
(...)
    },
    "ocp4-bastion.aio.example.com:5000": {
      "email": "dummy@redhat.com",
      "auth": "ZHVtbXk6ZHVtbXk="
    }
  }
}
~~~

Because our registry has a self signed certificate we will also need to add the certificate to our trust bundles in our install-config.yaml as well:

~~~bash
[root@ocp4-bastion ~]# sed -i -e 's/^/  /' ~/domain.crt
[root@ocp4-bastion ~]# echo "additionalTrustBundle: |" >> ~/lab/install-config.yaml
[root@ocp4-bastion ~]# cat ~/domain.crt >> ~/lab//install-config.yaml
~~~

And check that our certificate is included:

~~~bash
[root@ocp4-bastion ~]# cat ~/lab/install-config.yaml
apiVersion: v1
baseDomain: example.com
(...)
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIGKDCCBBCgAwIBAgIUYvrQxDIHbkaPoNk7YRyqhdVthk0wDQYJKoZIhvcNAQEL
  BQAwgYQxCzAJBgNVBAYTAlVTMRYwFAYDVQQIDA1Ob3J0aENhcm9saW5hMRAwDgYD
  VQQHDAdSYWxlaWdoMRAwDgYDVQQKDAdSZWQgSGF0MRIwEAYDVQQLDAlNYXJrZXRp
  bmcxJTAjBgNVBAMMHG9jcDQtYmFzdGlvbi5haW8uZXhhbXBsZS5jb20wHhcNMjEw
  NzE5MTg1OTUyWhcNMjIwNzE5MTg1OTUyWjCBhDELMAkGA1UEBhMCVVMxFjAUBgNV
  BAgMDU5vcnRoQ2Fyb2xpbmExEDAOBgNVBAcMB1JhbGVpZ2gxEDAOBgNVBAoMB1Jl
  ZCBIYXQxEjAQBgNVBAsMCU1hcmtldGluZzElMCMGA1UEAwwcb2NwNC1iYXN0aW9u
  LmFpby5leGFtcGxlLmNvbTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIB
  AMN4cy310E6ISXWsCWNW8nAmISRvVo/jVArbxYcPktshNml3KrawdX+WT5dFw1XB
  ECrWF6gUxqJ3gosymm3uepoLPqJViiG1R7OS+Dvi2h4xhCdVhms0wWohIDcmJZsG
  rYp6Y7UnLe5p12mrRpMpX/5Yf/oeEDTQieZAGDRkO3bT2YoVRffMXWB/jYQc15JC
  j+19koj10JljC7/cllbZFpq5kKjJaobofJ5W2imFyvHCjpL9oOD3BVIdhByHjDTh
  vk8C28DylwirlVXD8os1avukExeLtaA6eDx/b1kFvInQx0bWOHBLdzeZnSnV6FBy
  /QrtX+p5N28hKbXep+31QPUY+PdDvLB22KbkslpFib9oZ+g5YTmLgPBVUfUd5nBt
  WxjdBnVuPxexVRmevddfOKir+Wm3j6l+ImlnrZYhhRVByHUFktRDihZHMGYXqIp/
  o1YQZq1CgEk0LnE+IfS3ppM3K8ZAwq04BqpeSAwChRB1sHnRLRHoarbAtNVXRAbT
  wMRgmnhYrT4eKfyXD+3IgF0PWJHLCwj8C8O3z0KUiSr4DeSlAFYoG76++Z3lJ2Le
  ZIYAVrV0Z1dHzjpttwpMeePMHNneRFYnXTP51cmrjsztnf2ECTd3jY3AUs0eT/yx
  B1SIY+S09Hm7NmjZp3PeS5DGzkw1/cE7HBbSA3bg0XqTAgMBAAGjgY8wgYwwHQYD
  VR0OBBYEFEuKf/KUyt+V0yI6RIPGfqfizE6fMB8GA1UdIwQYMBaAFEuKf/KUyt+V
  0yI6RIPGfqfizE6fMA8GA1UdEwEB/wQFMAMBAf8wJwYDVR0RBCAwHoIcb2NwNC1i
  YXN0aW9uLmFpby5leGFtcGxlLmNvbTAQBgNVHSAECTAHMAUGAyoDBDANBgkqhkiG
  9w0BAQsFAAOCAgEADUmGD3Ep7u4YqHm/86siL39fkxMaK/FQRdRj300/+4BLpMrH
  seYtNBbo9Pi7BOHALlWJu9r4uTOlk2Qj2YZfBFRikONPRqeuDZclayEavP/Hu4Th
  skPKDTFzYIC9tVhs7/nJrmb1VYBDcLqkFdV4pzXPJgi2UJelqkAH21C7r0KhPsLh
  89JtHsoHob194/cYMwjcuJPSqim3c/vFVSbFKDdSak4rQLoEsynbKdzn4pd8ggNs
  mdMUmUdOuW+tLSAfb+T/2SKje71MsxbcEfB53+uAtVjcgX+qJsfC+yf8wJkhNdwz
  Uysk035m4QMxDmlMVYANMjBrbGcPfi7xs+/8d8u+DE2K3pZqDj8FlzM2yWQYW2JZ
  n1fDd2TdxJ8HOdel1P+e4Fo+iy7xVSDiO7nAjmDKFSc7g6tmhTCPTvlzBEICmiWj
  91Lf4mhd1g7uf7P9/jYOD9lwO65kS6EP/VT6GKD0TSu8wR1hASyW2e9AsiPCJ1pL
  FFiXHhY7zf/4ZT1GDOQFfwOXbLsy2xZ6y1wnEuYTlrdchJkqQgC1a178etUgpnV4
  6kAEJ44+15MeBNPMuRyKjeQPL7UELyFY02tEAoFRNovFZSKmrmUTALnzPCMlKSiu
  At2z3dZJhGLgieaS6e93iTQTsokH0p7GQv8c0h7GDXAK1a2ZgHOcoLnQF7U=
  -----END CERTIFICATE-----
~~~

Finally at this point we can sync down the pod images from quay.io to our local registry. To do this we need to perform a few steps below:

~~~bash
[root@ocp4-bastion ~]# export UPSTREAM_REPO="quay.io/openshift-release-dev/ocp-release:$VERSION-x86_64"
[root@ocp4-bastion ~]# export PULLSECRET=~/merged-pull-secret.json
[root@ocp4-bastion ~]# export LOCAL_REG="ocp4-bastion.aio.example.com:5000"
[root@ocp4-bastion ~]# export LOCAL_REPO='ocp4/openshift4'
~~~

So what did we do above?

We used the VERSION variable we set earlier to help us set the correct upstream registry and repository for the 4.7.19 release. Further we set variables for our pull secret, local registry, and local repository.

Now we can actually execute the mirroring:

~~~bash
[root@ocp4-bastion ~]# oc adm release mirror -a $PULLSECRET --from=$UPSTREAM_REPO \
    --to-release-image=$LOCAL_REG/$LOCAL_REPO:$VERSION --to=$LOCAL_REG/$LOCAL_REPO
info: Mirroring 130 images to ocp4-bastion.aio.example.com:5000/ocp4/openshift4 ...
ocp4-bastion.aio.example.com:5000/
  ocp4/openshift4
    blobs:
      quay.io/openshift-release-dev/ocp-release sha256:7df5b82029357ab3601bec214038b683e20603bbbd2e99e447aba5e50e56d607 1.701KiB
      quay.io/openshift-release-dev/ocp-release sha256:64607cc74f9cbe0e12f167547df0cf661de5a8b1fb4ebe930a43b9f621ca457f 1.748KiB
      quay.io/openshift-release-dev/ocp-release sha256:ef0beb9b789e9643af1ec4f5e726994f1cbf0bf4be22139f8d620efe7b9b5b61 483.9KiB
(...)
sha256:abd42e1de0b4febf4757d29921166ec777734db52f43ea59401b508d9a4c5962 ocp4-bastion.aio.example.com:5000/ocp4/openshift4:4.7.19-x86_64-openstack-cinder-csi-driver
sha256:d41a0ca485c9bcb4ddd180d4020ff271d37b5fca4177f1ef4d8262496000ab12 ocp4-bastion.aio.example.com:5000/ocp4/openshift4:4.7.19-x86_64-machine-api-operator
sha256:c016f3c5bb263f026492c1093517769beee4f9f80d8f2cdb32ef149a092fe356 ocp4-bastion.aio.example.com:5000/ocp4/openshift4:4.7.19-x86_64-jenkins
info: Mirroring completed in 4m43.42s (26.74MB/s)

Success
Update image:  ocp4-bastion.aio.example.com:5000/ocp4/openshift4:4.7.19
Mirror prefix: ocp4-bastion.aio.example.com:5000/ocp4/openshift4
Mirror prefix: ocp4-bastion.aio.example.com:5000/ocp4/openshift4:4.7.19

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - ocp4-bastion.aio.example.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - ocp4-bastion.aio.example.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - ocp4-bastion.aio.example.com:5000/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - ocp4-bastion.aio.example.com:5000/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
~~~

The above command takes a few minutes but once complete should have mirrored all the required images from the remote registry to our local registry. One key piece of output above is **imageContentSources**. That section is needed for the install-config.yaml file that is used for our OpenShift deployment.  Lets go ahead and add that information to the install-config.yaml:

~~~bash
[root@ocp4-bastion ~]# cat <<EOF >> ~/lab/install-config.yaml
imageContentSources:
- mirrors:
  - ocp4-bastion.aio.example.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - ocp4-bastion.aio.example.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
EOF
~~~

Now that we have the images synced down we can move on to syncing the RHCOS images needed. There are two required: an RHCOS qemu image and RHCOS openstack image. The RHCOS qemu image is the image used for the bootstrap virtual machines that is created on the provisioning host during the initial phases of the deployment process. The openstack image is used to image the master and worker nodes during the deployment process.

First, extract the commit ID from the installer output.

~~~bash
[root@ocp4-bastion ~]# INSTALL_COMMIT=$( ~/openshift-baremetal-install version | grep commit | cut -d' ' -f4)
~~~

Save the machine image information (JSON) associated with this commit into an environment variable.

~~~bash
[root@ocp4-bastion ~]# IMAGE_JSON=$(curl -s \
 https://raw.githubusercontent.com/openshift/installer/${INSTALL_COMMIT}/data/data/rhcos.json)
~~~

We're interested in several items in this JSON:

- `baseURI`: The location from which we will download images
- `images.qemu`: Information about the image used for the bootstrap VM
- `images.openstack`: Information about the image used (by Metal³ and OpenStack Ironic) on the master and worker nodes

Examine this information.

~~~bash
[root@ocp4-bastion ~]# echo $IMAGE_JSON | jq .baseURI
"https://releases-art-rhcos.svc.ci.openshift.org/art/storage/releases/rhcos-4.7/47.83.202105220305-0/x86_64

[root@ocp4-bastion ~]# echo $IMAGE_JSON | jq .images.qemu
{
  "path": "rhcos-47.83.202105220305-0-qemu.x86_64.qcow2.gz",
  "sha256": "4be67c5c1a3ac7ca67154244325f2bbf4d2ca872716e650c79e52eff4c512140",
  "size": 941849543,
  "uncompressed-sha256": "d3e6f4e1182789480dcb81fc8cdf37416ec9afa34b4be1056426b21b62272248",
  "uncompressed-size": 2399993856
}
[root@ocp4-bastion ~]# echo $IMAGE_JSON | jq .images.openstack
{
  "path": "rhcos-47.83.202105220305-0-openstack.x86_64.qcow2.gz",
  "sha256": "156e695b0a834cc9efe1ef95609cec078c65f8030be8c68b8fe946f82a65dd3a",
  "size": 940770428,
  "uncompressed-sha256": "94058cc4cff50e63ebeba8e044215c1591d0a4daea2ffdb778836d013290868e",
  "uncompressed-size": 2366046208
}
~~~

Store the useful bits in variables.

~~~bash
[root@ocp4-bastion ~]# URL_BASE=$(echo $IMAGE_JSON | jq -r .baseURI)
[root@ocp4-bastion ~]# QEMU_IMAGE_NAME=$(echo $IMAGE_JSON | jq -r .images.qemu.path)
[root@ocp4-bastion ~]# QEMU_IMAGE_SHA256=$(echo $IMAGE_JSON | jq -r .images.qemu.sha256)
[root@ocp4-bastion ~]# QEMU_IMAGE_UNCOMPRESSED_SHA256=$(echo $IMAGE_JSON | jq -r '.images.qemu."uncompressed-sha256"')
[root@ocp4-bastion ~]# OPENSTACK_IMAGE_NAME=$(echo $IMAGE_JSON | jq -r .images.openstack.path)
[root@ocp4-bastion ~]# OPENSTACK_IMAGE_SHA256=$(echo $IMAGE_JSON | jq -r .images.openstack.sha256)
~~~

Download the images into the local cache.

~~~bash
[root@ocp4-bastion ~]# curl -L -o ${IRONIC_DATA_DIR}/html/images/${QEMU_IMAGE_NAME} \
 ${URL_BASE}/${QEMU_IMAGE_NAME}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   145  100   145    0     0    464      0 --:--:-- --:--:-- --:--:--   463
100  898M  100  898M    0     0  37.6M      0  0:00:23  0:00:23 --:--:-- 45.6M

[root@ocp4-bastion ~]# curl -L -o ${IRONIC_DATA_DIR}/html/images/${OPENSTACK_IMAGE_NAME} \
 ${URL_BASE}/${OPENSTACK_IMAGE_NAME}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   145  100   145    0     0   2589      0 --:--:-- --:--:-- --:--:--  2589
100  897M  100  897M    0     0  45.9M      0  0:00:19  0:00:19 --:--:-- 53.4M
~~~

Check the images against the checksums from the JSON.

~~~bash
[root@ocp4-bastion ~]# echo "$QEMU_IMAGE_SHA256 ${IRONIC_DATA_DIR}/html/images/${QEMU_IMAGE_NAME}" \
 | sha256sum -c
/nfs/ocp/ironic/html/images/rhcos-47.83.202105220305-0-qemu.x86_64.qcow2.gz: OK

[root@ocp4-bastion ~]# echo "$OPENSTACK_IMAGE_SHA256 ${IRONIC_DATA_DIR}/html/images/${OPENSTACK_IMAGE_NAME}" \
 | sha256sum -c
/nfs/ocp/ironic/html/images/rhcos-47.83.202105220305-0-openstack.x86_64.qcow2.gz: OK
~~~

`install-config.yaml` has already been partly customized to refer to the local RHCOS image cache.

~~~bash
[root@ocp4-bastion ~]# grep Image: ~/lab/install-config.yaml
    bootstrapOSImage: RHCOS_QEMU_IMAGE
    clusterOSImage: RHCOS_OPENSTACK_IMAGE
~~~

We need to replace `RHCOS_QEMU_IMAGE` and `RHCOS_OPENSTACK_IMAGE` with the actual file names **and** checksums.  In the case of the QEMU image used by the bootstrap VM, the checksum must be that of the uncompressed image.

~~~bash
[root@ocp4-bastion ~]# RHCOS_QEMU_IMAGE=http://ocp4-bastion.aio.example.com/images/${QEMU_IMAGE_NAME}?sha256=${QEMU_IMAGE_UNCOMPRESSED_SHA256}

[root@ocp4-bastion ~]# RHCOS_OPENSTACK_IMAGE=http://ocp4-bastion.aio.example.com/images/${OPENSTACK_IMAGE_NAME}?sha256=${OPENSTACK_IMAGE_SHA256}

[root@ocp4-bastion ~]# sed -i "s*RHCOS_QEMU_IMAGE*$RHCOS_QEMU_IMAGE*g" ~/lab/install-config.yaml

[root@ocp4-bastion ~]# sed -i "s*RHCOS_OPENSTACK_IMAGE*$RHCOS_OPENSTACK_IMAGE*g" ~/lab/install-config.yaml
~~~

Check the results:

~~~bash
[root@ocp4-bastion ~]# grep Image: ~/lab/install-config.yaml
    bootstrapOSImage: http://ocp4-bastion.aio.example.com/images/rhcos-47.83.202105220305-0-qemu.x86_64.qcow2.gz?sha256=d3e6f4e1182789480dcb81fc8cdf37416ec9afa34b4be1056426b21b62272248
    clusterOSImage: http://ocp4-bastion.aio.example.com/images/rhcos-47.83.202105220305-0-openstack.x86_64.qcow2.gz?sha256=156e695b0a834cc9efe1ef95609cec078c65f8030be8c68b8fe946f82a65dd3a
~~~

Finally, ensure that we are able to retrieve the images from those URLs.

~~~bash
[root@ocp4-bastion ~]# curl -o /dev/null http://ocp4-bastion.aio.example.com/images/${RHCOS_QEMU_IMAGE}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   196  100   196    0     0  39200      0 --:--:-- --:--:-- --:--:-- 39200
[root@ocp4-bastion ~]# curl -o /dev/null http://ocp4-bastion.aio.example.com/images/${RHCOS_OPENSTACK_IMAGE}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   196  100   196    0     0  49000      0 --:--:-- --:--:-- --:--:-- 65333
~~~

As you can see it is rather easy to build a local registry and httpd cache for the pod images and RHCOS images. In the next lab we will leverage this content with a deployment of OpenShift!

[Move on to Creating an OpenShift Cluster](https://github.com/RHFieldProductManagement/baremetal-ipi-lab/blob/master/02-deploy-cluster.md)!
