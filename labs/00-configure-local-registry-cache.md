# **Configure a Disconnected Registry and Red Hat Enterprise Linux CoreOS Cache**

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
[root@ocp4-bastion ~]# rpm -qa podman httpd httpd-tools
httpd-tools-2.4.37-39.module_el8.4.0+778+c970deab.x86_64
httpd-2.4.37-39.module_el8.4.0+778+c970deab.x86_64
podman-3.0.1-7.module_el8.4.0+830+8027e1c4.x86_64
~~~

If by chance those packages are not installed use the following to install them:

~~~bash
[root@ocp4-bastion ~]# sudo dnf -y install podman httpd httpd-tools
Last metadata expiration check: 1 day, 0:42:29 ago on Wed 14 Jul 2021 06:25:33 PM UTC.
Package podman-3.0.1-7.module_el8.4.0+830+8027e1c4.x86_64 is already installed.
Package httpd-2.4.37-39.module_el8.4.0+778+c970deab.x86_64 is already installed.
Package httpd-tools-2.4.37-39.module_el8.4.0+778+c970deab.x86_64 is already installed.
Dependencies resolved.
Nothing to do.
Complete!
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

STOP HERE - NEED to fix everything below

```bash
[lab-user@provision scripts]$ sudo openssl req -newkey rsa:4096 -nodes -sha256 \
    -keyout /nfs/registry/certs/domain.key -x509 -days 365 -out /nfs/registry/certs/domain.crt \
    -subj "/C=US/ST=NorthCarolina/L=Raleigh/O=Red Hat/OU=Marketing/CN=provision.$GUID.dynamic.opentlc.com"
Generating a RSA private key
..................................................................................................
..................................................................................................
..................................................................++++
..................................................................................................
..................................................................................................
.........................................................++++
writing new private key to '/nfs/registry/certs/domain.key'
-----
```

Once the certificate has been created copy it into your home directory and also into the trust anchors on the provisioning node. We will also need to run the update-ca-trust command:

```bash
[lab-user@provision scripts]$ sudo cp /nfs/registry/certs/domain.crt $HOME/scripts/domain.crt
[lab-user@provision scripts]$ sudo chown lab-user $HOME/scripts/domain.crt
[lab-user@provision scripts]$ sudo cp /nfs/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
[lab-user@provision scripts]$ sudo update-ca-trust extract
```

Our registry will need a simple authentication mechanism so we will use `htpasswd`. Note that when you try to authenticate to your registry the password being passed has to be in [bcrypt](https://en.wikipedia.org/wiki/Bcrypt) format.

```bash
[lab-user@provision scripts]$ sudo htpasswd -bBc /nfs/registry/auth/htpasswd dummy dummy
Adding password for user dummy
```

Now that we have a directory structure, certificate, and a user configured for authentication we can go ahead and create the registry pod. The command below will pull down the pod and mount the appropriate directory mount points we created earlier.

```bash
[lab-user@provision scripts]$ sudo podman create --name poc-registry --net host -p 5000:5000 \
    -v /nfs/registry/data:/var/lib/registry:z -v /nfs/registry/auth:/auth:z \
    -e "REGISTRY_AUTH=htpasswd" -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry" \
    -e "REGISTRY_HTTP_SECRET=ALongRandomSecretForRegistry" \
    -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -v /nfs/registry/certs:/certs:z \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key docker.io/library/registry:2

Trying to pull docker.io/library/registry:2...
Getting image source signatures
Copying blob cbdbe7a5bc2a done
Copying blob c1cc712bcecd done
Copying blob 47112e65547d done
Copying blob 3db6272dcbfa done
Copying blob 46bcb632e506 done
Copying config 2d4f4b5309 done
Writing manifest to image destination
Storing signatures
be06131e5dc4b98a1f55fdefc6afa6989cfbc8d878b6d65cf40426e96e2bede1
```

Once pod creation is complete we can start the pod:

```bash
[lab-user@provision scripts]$ sudo podman start poc-registry
poc-registry
```

Finally, verify the pod is up and running via the podman command:

```bash
[lab-user@provision scripts]$ sudo podman ps
CONTAINER ID  IMAGE                         COMMAND               CREATED        STATUS             PORTS  NAMES
be06131e5dc4  docker.io/library/registry:2  /etc/docker/regis...  2 minutes ago  Up 39 seconds ago         poc-registry
```

We can further validate the registry is functional by using a curl command and passing the user/password to the registry URL. Note here it's not necessary to use a bcrypt formatted password.

```bash
[lab-user@provision scripts]$ curl -u dummy:dummy -k \
    https://provision.$GUID.dynamic.opentlc.com:5000/v2/_catalog
{"repositories":[]}
```

Now that our registry pod is up and we have validated that it's working it's time to configure the httpd cache pod which stores our RHCOS images locally. The first step is to create some directory structures and add the appropriate permissions:

```bash
[lab-user@provision scripts]$ export IRONIC_DATA_DIR=/nfs/ocp/ironic
[lab-user@provision scripts]$ export IRONIC_IMAGES_DIR="${IRONIC_DATA_DIR}/html/images"
[lab-user@provision scripts]$ export IRONIC_IMAGE=quay.io/metal3-io/ironic:master
[lab-user@provision scripts]$ sudo mkdir -p $IRONIC_IMAGES_DIR
[lab-user@provision scripts]$ sudo chown -R "${USER}:users" "$IRONIC_DATA_DIR"
[lab-user@provision scripts]$ sudo find $IRONIC_DATA_DIR -type d -print0 | xargs -0 chmod 755
[lab-user@provision scripts]$ sudo chmod -R +r $IRONIC_DATA_DIR
```

With the directory structures in place we can now create the caching pod. For this we use the ironic pod that already exists in quay.io.

```bash
[lab-user@provision scripts]$ sudo podman pod create -n ironic-pod
12385a4f6f8cb912e7733b725c2b488de4e21aef049552efd21afc28dd647014
```

And now run the pod:

```bash
[lab-user@provision scripts]$ sudo podman run -d --net host --privileged --name httpd --pod ironic-pod \
    -v $IRONIC_DATA_DIR:/shared --entrypoint /bin/runhttpd ${IRONIC_IMAGE}

Trying to pull quay.io/metal3-io/ironic:master...
Getting image source signatures
Copying blob 3c72a8ed6814 done
Copying blob dedbfd2c2275 done
(...)
Copying blob db435f5910cb done
Copying config 3733498f02 done
Writing manifest to image destination
Storing signatures
f069949f68fa147206d154417a22c20c49983f0c5b79e9c06d56750e9d3f470d
```

Because we ran the **create** command and then a **run** command after it there is no need to actually use podman to start the httpd pod. We can see that it is running by looking at the running pods on the provisioning node:

```bash
[lab-user@provision scripts]$ sudo podman ps
CONTAINER ID  IMAGE                            COMMAND               CREATED         STATUS             PORTS  NAMES
f069949f68fa  quay.io/metal3-io/ironic:master                        8 seconds ago   Up 7 seconds ago          httpd
be06131e5dc4  docker.io/library/registry:2     /etc/docker/regis...  22 minutes ago  Up 20 minutes ago         poc-registry
```

As shown you should see **httpd** and **poc-registry** running.

Further we can test that our httpd cache is operational by using the `curl` command. If you get a 301 code that is normal since we have yet to actually place any images in the cache.

```bash
[lab-user@provision scripts]$ curl http://provision.$GUID.dynamic.opentlc.com/images

<!DOCTYPE HTML PUBLIC "-//IETF//DTD HTML 2.0//EN">
<html><head>
<title>301 Moved Permanently</title>
</head><body>
<h1>Moved Permanently</h1>
<p>The document has moved <a href="http://provision.schmaustech.dynamic.opentlc.com/images/">here</a>.</p>
</body></html>
```

We are almost ready to do some downloading of images but we still have a few items to tend to. First we need to generate a bcrypt password from our username and password we set on the registry. We can do this by piping them into `base64`:

```bash
[lab-user@provision scripts]$ echo -n 'dummy:dummy' | base64 -w0 && echo
ZHVtbXk6ZHVtbXk=
```

The bcrypt password needs to be embedded in a registry secret text file:

```bash
[lab-user@provision scripts]$ cat <<EOF > ~/reg-secret.txt
"provision.$GUID.dynamic.opentlc.com:5000": {
    "email": "dummy@redhat.com",
    "auth": "$(echo -n 'dummy:dummy' | base64 -w0)"
}
EOF
```

Now we can take our existing lab pull secret and our registry pull secret and merge them. Below we will backup the existing pull secret and then add our registry pull secret to the file. Further we will add our pull secret and registry cert, as a trust bundle, to the existing install-config.yaml.

```bash
[lab-user@provision scripts]$ export PULLSECRET=$HOME/pull-secret.json
[lab-user@provision scripts]$ cp $PULLSECRET $PULLSECRET.orig
[lab-user@provision scripts]$ cat $PULLSECRET | jq ".auths += {`cat ~/reg-secret.txt`}" > $PULLSECRET
[lab-user@provision scripts]$ cat $PULLSECRET | tr -d '[:space:]' > tmp-secret
[lab-user@provision scripts]$ mv -f tmp-secret $PULLSECRET
[lab-user@provision scripts]$ rm -f ~/reg-secret.txt
[lab-user@provision scripts]$ sed -i -e 's/^/  /' $(pwd)/domain.crt
[lab-user@provision scripts]$ echo "additionalTrustBundle: |" >> $HOME/scripts/install-config.yaml
[lab-user@provision scripts]$ cat $HOME/scripts/domain.crt >> $HOME/scripts/install-config.yaml
[lab-user@provision scripts]$ sed -i "s/pullSecret:.*/pullSecret: \'$(cat $PULLSECRET)\'/g" \
    $HOME/scripts/install-config.yaml
```

Verify that the pull secret in `install-config.yaml` now includes the credentials for our local registry.

```bash
[lab-user@provision scripts]$ grep pullSecret install-config.yaml | sed 's/^pullSecret: //' | tr -d \' | jq .
(...)
    "provision.9mj2p.dynamic.opentlc.com:5000": {
      "email": "dummy@redhat.com",
      "auth": "ZHVtbXk6ZHVtbXk="
    }
  }
}
```

And check that our certificate is included.

```bash
[lab-user@provision scripts]$ cat install-config.yaml
apiVersion: v1
baseDomain: dynamic.opentlc.com
(...)
additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  MIIGDzCCA/egAwIBAgIUc3tgxZl2g92XdCUX15hWMAIGi10wDQYJKoZIhvcNAQEL
  BQAwgZYxCzAJBgNVBAYTAlVTMRYwFAYDVQQIDA1Ob3J0aENhcm9saW5hMRAwDgYD
  VQQHDAdSYWxlaWdoMRAwDgYDVQQKDAdSZWQgSGF0MRIwEAYDVQQLDAlNYXJrZXRp
  bmcxNzA1BgNVBAMMLnByb3Zpc2lvbi5zY2htYXVzdGVjaC5zdHVkZW50cy5vc3Au
  b3BlbnRsYy5jb20wHhcNMjAxMDA1MTM0OTAzWhcNMjExMDA1MTM0OTAzWjCBljEL
  MAkGA1UEBhMCVVMxFjAUBgNVBAgMDU5vcnRoQ2Fyb2xpbmExEDAOBgNVBAcMB1Jh
  bGVpZ2gxEDAOBgNVBAoMB1JlZCBIYXQxEjAQBgNVBAsMCU1hcmtldGluZzE3MDUG
  A1UEAwwucHJvdmlzaW9uLnNjaG1hdXN0ZWNoLnN0dWRlbnRzLm9zcC5vcGVudGxj
  LmNvbTCCAiIwDQYJKoZIhvcNAQEBBQADggIPADCCAgoCggIBAMvffu+qXR0r3Yxg
  Z1tUKYejJTmEXf7e4JDlKWyijeu8buDJD0T544gBtWDbEwpho7lsnRgC7w5Peasc
  DpOQAqI980vQp8tAnS9ncJVroUAtNtf3WLLVpoEPbNTyRdZ2clEh17KcnJQ4Hsjd
  mMiRNMLzmjBocAXeA2mGkjm2ZN/+fkaC2Zk1DtcPPuF7+apNRk9dizqYawupwgrF
  zSjFitvf1IC49NtO5b01VWW3056HX+bx8KkGAAMNvqaRlz703HWEeplfsEkyVvTL
  SOF2BJIbS1HxYZ92qnwIVjzgdx8eZPV954pDvQovEXJExShn9mDEZWuQDcwnwdyU
  o+zgvzp1dFm9y6iC1u+8eG5wnmoJRkFyxkE3Uoysj2yMcSNGUK8z9O2rBulA8rPC
  IO4oaizL102wUHj6+ESvbYm5Gjzj/trKuhEtCXYmtyndHe1PsKRmUEq8dZAJBrXY
  axasroyODSIN6g6wSNSyS490wfu4QZnuEb1X9qXNsvNOgGRwrCEodyAiwCMvVNMw
  eDA2XAukNktOUVmzrQiupn37lGVhpl47ssmPW5EKWI9SNehz09x16ZVlGRs2ojQU
  XbutiswxseFm8Qn9teBKLqR2HuOAZb5xS9EDesocwoGRnenmqP+jYt8ifq4ajOEV
  nJz8oGpIt9gLWaay0fnIzfG08KXHAgMBAAGjUzBRMB0GA1UdDgQWBBTJDjq5gFPw
  5oqaDPOci3iikL61GTAfBgNVHSMEGDAWgBTJDjq5gFPw5oqaDPOci3iikL61GTAP
  BgNVHRMBAf8EBTADAQH/MA0GCSqGSIb3DQEBCwUAA4ICAQDBHCZeAKgkcGAGZOfS
  F3ohdYj50MeN/lVtESbvUlirMGEs2f932YkY1oF8ulFy2n3EZftTTpUo2/tDKik7
  3rZs/cCD8KrPnHAdSJGny7ud27w85DM+dFTwxuIjdHAXdMhoOKvV+lSkziW9Ltmg
  p7MbOei2nqpxTpX42DfqqC2ZRZ1KyyQ8EClqTlYh3iozwyp1VwpHBnQkFnZLfLjc
  cHcbayrEgxN7TxUJYqHUP90A7guHA1OfWSSduNN1b8aFACegOtb9MFTRjbIrbNw4
  R1t5D5TsMc8RfIETHE+9xb1HLdojnXQA8Hwp3myVL7PNr6tKu01hKvbhhhykACFz
  KtVFLCdv3EF/dZJHahQSJksThFY6Jaeyj7rE6OJ1lJbB/RMGdV/3l7kyDbs7A/mf
  XKt6I4WoymEDcC7dlcif4sQHMWCKwHMtT8pen04T48CGb/5GLGBVwkx6qEynOE3S
  KukJj2o1QZJOSi5KdSfGeILAUHW1eOOWank9l1SIS5OIaNBGkSmem6J9heGy6ulv
  5IU9ahv5IMoJS8wJgkgTMc5B2B/Mbv2dL+kthbemyyPCdN62QtlvhkLCiOU4niI3
  JdXFoPLSZ5nEb+Y/XB+WVaaz+j8CtGlDcwbGr4BFlakHluVxfK2sDN5n0NOLcQlw
  Z2sCrU3XLJDME6tPetuPiBX/tA==
  -----END CERTIFICATE-----
```

Finally at this point we can sync down the pod images from quay.io to our local registry. To do this we need to perform a few steps below:

```bash
[lab-user@provision scripts]$ export UPSTREAM_REPO="quay.io/openshift-release-dev/ocp-release:$VERSION-x86_64"
[lab-user@provision scripts]$ export PULLSECRET=$HOME/pull-secret.json
[lab-user@provision scripts]$ export LOCAL_REG="provision.$GUID.dynamic.opentlc.com:5000"
[lab-user@provision scripts]$ export LOCAL_REPO='ocp4/openshift4'
```

So what did we do above?

We used the VERSION variable we set earlier to help us set the correct upstream registry and repository for the 4.5.12 release. Further we set variables for our pull secret, local registry, and local repository.

Now we can actually execute the mirroring:

```bash
[lab-user@provision scripts]$ oc adm release mirror -a $PULLSECRET --from=$UPSTREAM_REPO \
    --to-release-image=$LOCAL_REG/$LOCAL_REPO:$VERSION --to=$LOCAL_REG/$LOCAL_REPO

info: Mirroring 110 images to provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4 ...
provision.schmaustech.dynamic.opentlc.com:5000/
  ocp4/openshift4
    manifests:
      sha256:00edb6c1dae03e1870e1819b4a8d29b655fb6fc40a396a0db2d7c8a20bd8ab8d -> 4.5.12-local-storage-static-provisioner
      sha256:0259aa5845ce43114c63d59cedeb71c9aa5781c0a6154fe5af8e3cb7bfcfa304 -> 4.5.12-machine-api-operator
      sha256:07f11763953a2293bac5d662b6bd49c883111ba324599c6b6b28e9f9f74112be -> 4.5.12-cluster-kube-storage-version-migrator-operator
(...)
sha256:15be0e6de6e0d7bec726611f1dcecd162325ee57b993e0d886e70c25a1faacc3 provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.12-openshift-controller-manager
sha256:bc6c8fd4358d3a46f8df4d81cd424e8778b344c368e6855ed45492815c581438 provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.12-hyperkube
sha256:bcd6cd1559b62e4a8031cf0e1676e25585845022d240ac3d927ea47a93469597 provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.12-machine-config-operator
sha256:b05f9e685b3f20f96fa952c7c31b2bfcf96643e141ae961ed355684d2d209310 provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.12-baremetal-installer
info: Mirroring completed in 1.48s (0B/s)

Success
Update image:  provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4:4.5.12
Mirror prefix: provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
- mirrors:
  - provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4
  source: registry.svc.ci.openshift.org/ocp/release


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
  - mirrors:
    - provision.schmaustech.dynamic.opentlc.com:5000/ocp4/openshift4
    source: registry.svc.ci.openshift.org/ocp/release
```

The above command takes a few minutes but once complete should have mirrored all the required images from the remote registry to our local registry. One key piece of output above is **imageContentSources**. That section is needed for the install-config.yaml file that is used for our OpenShift deployment and if you look at **~/scripts/install-config.yaml** you will notice those lines have already been added to the config for you.

Now that we have the images synced down we can move on to syncing the RHCOS images needed. There are two required: an RHCOS qemu image and RHCOS openstack image. The RHCOS qemu image is the image used for the bootstrap virtual machines that is created on the provisioning host during the initial phases of the deployment process. The openstack image is used to image the master and worker nodes during the deployment process.

First, extract the commit ID from the installer output.

```bash
[lab-user@provision scripts]$ INSTALL_COMMIT=$(./openshift-baremetal-install version | grep commit | cut -d' ' -f4)
```

Save the machine image information (JSON) associated with this commit into an environment variable.

```bash
[lab-user@provision scripts]$ IMAGE_JSON=$(curl -s \
	https://raw.githubusercontent.com/openshift/installer/${INSTALL_COMMIT}/data/data/rhcos.json)
```

We're interested in several items in this JSON:

- `baseURI`: The location from which we will download images
- `images.qemu`: Information about the image used for the bootstrap VM
- `images.openstack`: Information about the image used (by Metal³ and OpenStack Ironic) on the master and worker nodes

Examine this information.

```bash
[lab-user@provision scripts]$ echo $IMAGE_JSON | jq .baseURI
"https://releases-art-rhcos.svc.ci.openshift.org/art/storage/releases/rhcos-4.5/45.82.202008010929-0/x86_64/"

[lab-user@provision scripts]$ echo $IMAGE_JSON | jq .images.qemu
{
  "path": "rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz",
  "sha256": "80ab9b70566c50a7e0b5e62626e5ba391a5f87ac23ea17e5d7376dcc1e2d39ce",
  "size": 898670890,
  "uncompressed-sha256": "c9e2698d0f3bcc48b7c66d7db901266abf27ebd7474b6719992de2d8db96995a",
  "uncompressed-size": 2449014784
}

[lab-user@provision scripts]$ echo $IMAGE_JSON | jq .images.openstack
{
  "path": "rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz",
  "sha256": "359e7c3560fdd91e64cd0d8df6a172722b10e777aef38673af6246f14838ab1a",
  "size": 896764070,
  "uncompressed-sha256": "036a497599863d9470d2ca558cca3c4685dac06243709afde40ad008dce5a8ac",
  "uncompressed-size": 2400518144
}
```

Store the useful bits in variables.

```bash
[lab-user@provision scripts]$ URL_BASE=$(echo $IMAGE_JSON | jq -r .baseURI)
[lab-user@provision scripts]$ QEMU_IMAGE_NAME=$(echo $IMAGE_JSON | jq -r .images.qemu.path)
[lab-user@provision scripts]$ QEMU_IMAGE_SHA256=$(echo $IMAGE_JSON | jq -r .images.qemu.sha256)
[lab-user@provision scripts]$ QEMU_IMAGE_UNCOMPRESSED_SHA256=$(echo $IMAGE_JSON | jq -r '.images.qemu."uncompressed-sha256"')
[lab-user@provision scripts]$ OPENSTACK_IMAGE_NAME=$(echo $IMAGE_JSON | jq -r .images.openstack.path)
[lab-user@provision scripts]$ OPENSTACK_IMAGE_SHA256=$(echo $IMAGE_JSON | jq -r .images.openstack.sha256)
```

Download the images into the local cache.

```bash
[lab-user@provision scripts]$ curl -L -o ${IRONIC_DATA_DIR}/html/images/${QEMU_IMAGE_NAME} \
	${URL_BASE}/${QEMU_IMAGE_NAME}
 % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   161  100   161    0     0    463      0 --:--:-- --:--:-- --:--:--   463
100  857M  100  857M    0     0  28.2M      0  0:00:30  0:00:30 --:--:-- 48.7M

[lab-user@provision scripts]$ curl -L -o ${IRONIC_DATA_DIR}/html/images/${OPENSTACK_IMAGE_NAME} \
	${URL_BASE}/${OPENSTACK_IMAGE_NAME}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
100   161  100   161    0     0    958      0 --:--:-- --:--:-- --:--:--   958
100  855M  100  855M    0     0  47.0M      0  0:00:18  0:00:18 --:--:-- 49.3M
```

Check the images against the checksums from the JSON.

```bash
[lab-user@provision scripts]$ echo "$QEMU_IMAGE_SHA256 ${IRONIC_DATA_DIR}/html/images/${QEMU_IMAGE_NAME}" \
	| sha256sum -c
/nfs/ocp/ironic/html/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz: OK

[lab-user@provision scripts]$ echo "$OPENSTACK_IMAGE_SHA256 ${IRONIC_DATA_DIR}/html/images/${OPENSTACK_IMAGE_NAME}" \
	| sha256sum -c
/nfs/ocp/ironic/html/images/rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz: OK
```

`install-config.yaml` has already been partly customized to refer to the local RHCOS image cache.

```bash
[lab-user@provision scripts]$ grep http://10.20.0.2 install-config.yaml
    bootstrapOSImage: http://10.20.0.2/images/RHCOS_QEMU_IMAGE
    clusterOSImage: http://10.20.0.2/images/RHCOS_OPENSTACK_IMAGE
```

We need to replace `RHCOS_QEMU_IMAGE` and `RHCOS_OPENSTACK_IMAGE` with the actual file names **and** checksums.  In the case of the QEMU image used by the bootstrap VM, the checksum must be that of the uncompressed image.

```bash
[lab-user@provision scripts]$ RHCOS_QEMU_IMAGE=${QEMU_IMAGE_NAME}?sha256=${QEMU_IMAGE_UNCOMPRESSED_SHA256}

[lab-user@provision scripts]$ RHCOS_OPENSTACK_IMAGE=${OPENSTACK_IMAGE_NAME}?sha256=${OPENSTACK_IMAGE_SHA256}

[lab-user@provision scripts]$ sed -i "s/RHCOS_QEMU_IMAGE/$RHCOS_QEMU_IMAGE/g" \
	$HOME/scripts/install-config.yaml

[lab-user@provision scripts]$ sed -i "s/RHCOS_OPENSTACK_IMAGE/$RHCOS_OPENSTACK_IMAGE/g" \
	$HOME/scripts/install-config.yaml
```

Check the results.

```bash
[lab-user@provision scripts]$ grep http://10.20.0.2 install-config.yaml
    bootstrapOSImage: http://10.20.0.2/images/rhcos-45.82.202008010929-0-qemu.x86_64.qcow2.gz?sha256=c9e2698d0f3bcc48b7c66d7db901266abf27ebd7474b6719992de2d8db96995a
    clusterOSImage: http://10.20.0.2/images/rhcos-45.82.202008010929-0-openstack.x86_64.qcow2.gz?sha256=359e7c3560fdd91e64cd0d8df6a172722b10e777aef38673af6246f14838ab1a
```

Finally, ensure that we are able to retrieve the images from those URLs.

```bash
[lab-user@provision scripts]$ curl -o /dev/null http://10.20.0.2/images/${RHCOS_QEMU_IMAGE}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  857M  100  857M    0     0  2279M      0 --:--:-- --:--:-- --:--:-- 2279M
[lab-user@provision scripts]$ curl -o /dev/null http://10.20.0.2/images/${RHCOS_OPENSTACK_IMAGE}
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  855M  100  855M    0     0  2221M      0 --:--:-- --:--:-- --:--:-- 2221M
```

As you can see it is rather easy to build a local registry and httpd cache for the pod images and RHCOS images. In the next lab we will leverage this content with a deployment of OpenShift!

[Move on to Creating an OpenShift Cluster](https://github.com/RHFieldProductManagement/baremetal-ipi-lab/blob/master/04-deploying-cluster.md)!
