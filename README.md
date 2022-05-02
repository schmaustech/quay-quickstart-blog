# **Quintessential Red Hat Quay Quickstart**

Red Hat Quay is a secure, private container registry that builds, analyzes and distributes container images. It provides a high level of automation and customization. Red Hat Quay is available via a deployed Operator in OpenShift and as a standalone component on a server.  In this blog we will focus on the deployment method using the operator on OpenShift and do all of the installation via the command line.



To begin the deployment process of Quay we first need to create a subscription yaml file.  

~~~bash
$ cat << EOF > ~/quay-operator-subscription.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: quay-operator
  namespace: openshift-operators
spec:
  channel: stable-3.6
  installPlanApproval: Automatic
  name: quay-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  startingCSV: quay-operator.v3.6.4
EOF
~~~

Once we have the subcription yaml created lets go ahead and create the subscription on cluster0:

~~~bash
$ oc create -f ~/quay-operator-subscription.yaml 
subscription.operators.coreos.com/quay-operator created
~~~

With the subscription created we can validate that the operator is running by looking at both the running pod and subscription:

~~~bash
$ oc get pods -A|grep quay
openshift-operators                  quay-operator.v3.6.4-64d4cc85d8-kgd45                             1/1     Running     0               30s

$ oc get sub -n openshift-operators quay-operator
NAME            PACKAGE         SOURCE             CHANNEL
quay-operator   quay-operator   redhat-operators   stable-3.6
~~~

With the Quay operator running we next need to create a namespace where the Quay registry will run.

~~~bash
$ oc create namespace quay-poc
namespace/quay-poc created
~~~

With the namespace created we can move onto generating an initial config.yaml for Quay.   In this yaml we are going to specify a quayadmin super and enable the ability to use the API of Quay:

~~~bash
$ cat << EOF > ~/config.yaml 
FEATURE_USER_INITIALIZE: true
BROWSER_API_CALLS_XHR_ONLY: false
SUPER_USERS:
- quayadmin
FEATURE_USER_CREATION: false
EOF
~~~

We can now take the config.yaml we created and generate a secret config bundle which we will store in the quay-poc namespace we created:

~~~bash
$ oc create secret generic --from-file config.yaml=config.yaml init-config-bundle-secret -n quay-poc 
secret/init-config-bundle-secret created
~~~

With the init-config-bundle-secret created we will now reference it in our quay-registry custom resource file.  For the POC we are keeping the custom resource file simple and allowing the operator to control every aspect of the Quay deployment.  However if one chose to maybe external S3 storage or an external postgres DB this would be the file where those modifications could be made.  Since we have ODF installed on cluster0 the Quay operator will leverage both block and S3 storage from ODF to meet the needs of Quay and no additional configuration is necessary.  

~~~bash
$ cat << EOF > ~/quay-registry.yaml
apiVersion: quay.redhat.com/v1
kind: QuayRegistry
metadata:
  name: poc-registry
  namespace: quay-poc
spec:
  configBundleSecret: init-config-bundle-secret
EOF
~~~

Take the custom resource file we created and create it on cluster0

~~~bash
$ oc create -f ~/quay-registry.yaml
quayregistry.quay.redhat.com/poc-registry created
~~~

After about 3-5 minutes we can validate that Quay is running by running the following command:

~~~bash
$ oc get pods -n quay-poc
NAME                                               READY   STATUS      RESTARTS       AGE
poc-registry-clair-app-6bdf6d4dbb-296ps            1/1     Running     0              59s
poc-registry-clair-app-6bdf6d4dbb-2d9wc            1/1     Running     0              69s
poc-registry-clair-postgres-7d9c8bdc9b-pb5bv       1/1     Running     1 (112s ago)   2m6s
poc-registry-quay-app-785b4588d6-gmf94             1/1     Running     1 (62s ago)    69s
poc-registry-quay-app-785b4588d6-kw5b5             1/1     Running     0              69s
poc-registry-quay-app-upgrade-5lwck                0/1     Completed   0              76s
poc-registry-quay-config-editor-6449998bb8-2p4r7   1/1     Running     0              69s
poc-registry-quay-database-68764f57d5-9rhxn        1/1     Running     0              2m6s
poc-registry-quay-mirror-6f6bfbf54c-9tw84          1/1     Running     0              59s
poc-registry-quay-mirror-6f6bfbf54c-vhss8          1/1     Running     0              59s
poc-registry-quay-postgres-init-mcl56              0/1     Completed   0              78s
poc-registry-quay-redis-796f46d85f-79527           1/1     Running     0              2m6s
~~~

At this point we have a working Quay deployment but we need to do a few configuration steps in order to establish a registry we can sync the OpenShift images to.   Before we can go through those steps however we need to ensure we have a few tools we will leverage: jq, podman and oc.

First lets install jq using the brew command:

~~~bash

~~~

Next lets install podman via brew.  Unlike jq, podman has a lot more dependencies to install it!

~~~bash

~~~

And finally the oc command for Mac can be downloaded here: https://console.redhat.com/openshift/downloads   I just decided to leave it in my download directory and place the path in my PATH variable:

~~~bash
$ which oc
/Users/bschmaus/Downloads/oc
~~~

With our tools in place we can now begin the configuration which we will do all via the Quay API.

Create initial quayadmin
~~~bash
$ curl -X POST -k  https://poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/api/v1/user/initialize --header 'Content-Type: application/json' --data '{ "username": "quayadmin", "password":"password", "email": "quayadmin@example.com", "access_token": true}'

Output:
{"access_token":"82HLQR6XH0YPGZVDYOZPVTB1ZAF6J0DBEAQICWSV","email":"quayadmin@example.com","encrypted_password":"2Fio6NQe8EcgpwuAR/lh8LXPIMOVszLljmfVKJMim9DCEUO/zT2iUBjcjO4Rf/yB","username":"quayadmin"}
~~~

Lets take the access token and set it to a variable so we can run the further commands using the token:

~~~bash
TOKEN=82HLQR6XH0YPGZVDYOZPVTB1ZAF6J0DBEAQICWSV
~~~

Now that we have set the access token lets move onto creating a few more things for our registry configuration.   The first thing we need is a Quay organization. Organizations provide a way of sharing repositories under a common namespace that does not belong to a single user, but rather to many users in a shared setting (such as a company).  Again we will create this via the curl to access the API and in this example call the organization openshift4:

~~~bash
$  curl -X POST -k --header 'Content-Type: application/json' -H "Authorization: Bearer $TOKEN" https://poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/api/v1/organization/ --data '{"name": "openshift4", "email": "openshift4@example.com"}'

Output:
"Created"
~~~

Inside the organization we just created we can now create a repository.  The repository is created under the organization to store the actual images for the various containers needed.  In this example we will also call the repository openshift4 and use curl to access the API:

~~~bash
$  curl -X POST -k --header 'Content-Type: application/json' -H "Authorization: Bearer $TOKEN" https://poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/api/v1/repository --data '{"namespace":"openshift4","repository":"openshift4","visibility":"public","description":"","repo_kind":"image"}'

Output:
{"namespace": "openshift4", "name": "openshift4", "kind": "image"}
~~~

In order to get images into the repository we just created we need to have a user and while we have the quayadmin user already created we really should create another user that will be the user to pull/push images into our openshift4 repository.  In the example below we will create a user called openshift via curl and the API:

~~~bash
$  curl -X POST -k  https://poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/api/v1/superuser/users/ -H "Authorization: Bearer $TOKEN" --header 'Content-Type: application/json' --data '{ "username": "openshift", "email": "openshift@example.com", "access_token": true}'

Output:
{"username": "openshift", "email": "openshift@example.com", "password": "G17KT667IJ82206DF7BIE2C12RQRIYSF", "encrypted_password": "jqDT/R10PUVsxMR6sjoXCCQL1WLmE2sBp6yZloj+oRTLT0IslK/31WgJ/aXiUH/gfAMwo6ZgTO8vp4Cc6E7hZhrpyEIa2JtLv0A+UM7DZmI="}
~~~

Lets take the above users encrypted password and set it along with the username in a few variables we will use it to create a pull-secret later on for this user that will be required in order to push/pull images into the openshift repository:

~~~bash
QUAYUSERNAME=openshift
QUAYPASSWORD=qDFQgW6q6lT+suKZ3HL1XMrQJG9rG376yhGGIP4Iuf6QUMsg8XzNl/jj8guEAbSbZvavAmiOAvPZeOemOHdDvk0ZeAhEbjOQbIHo0jXqvCk=
~~~

After creating our user and setting our variables we now need to associate the to the organization we created via curl and the API:

~~~bash
$  curl -X PUT -k  https://poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/api/v1/organization/openshift4/team/owners/members/openshift -H "Authorization: Bearer $TOKEN" --header 'Content-Type: application/json' --data '{}'

Output:
{"name": "openshift", "kind": "user", "is_robot": false, "avatar": {"name": "openshift", "hash": "b1321f59a2be698dd8abc79e4524c6b59a0b40715cfa3db60e291fbab2bd2098", "color": "#1f77b4", "kind": "user"}, "invited": false}
~~~

We will also need to give this user a role on the repository so they have the ability to push/pull images after they authenticate:

~~~bash
$  curl -X PUT -k  https://poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/api/v1/repository/openshift4/openshift4/permissions/user/openshift -H "Authorization: Bearer $TOKEN" --header 'Content-Type: application/json' --data '{ "role": "admin"}'

Output:
{"role": "admin", "name": "openshift", "is_robot": false, "avatar": {"name": "openshift", "hash": "b1321f59a2be698dd8abc79e4524c6b59a0b40715cfa3db60e291fbab2bd2098", "color": "#1f77b4", "kind": "user"}, "is_org_member": true}
~~~

This completes all the Quay steps needed for ensuring we have a repository to mirror the OpenShift images for a disconnected cluster.  Now we need to prepare for the actual mirroring of those images.  

First we need to extract the ca.crt from the new Quay installation.  I should note though this is only required when using self-signed certs like we are using in this example.  If the cluster is deployed with certs from a real certificate of authority then this would not be required.  To extract the CA for the self siged cert create the temporary directory of quay-ca and then using the openssl command extract the CA from the site and save as ca.crt:

~~~bash
$ mkdir ~/quay-ca
$ openssl s_client -connect poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com:443 -showcerts </dev/null 2>/dev/null|openssl x509 -outform PEM > ~/quay-ca/ca.crt
~~~

Next lets take the username and encrpyted password variables we set earlier and base64 encoded them into another variable:

~~~bash
$ BASE64AUTH=`echo $QUAYUSERNAME:$QUAYPASSWORD | base64`
~~~

With the BASE64AUTH variable set lets go ahead and create a snippit of quay-secret.json:

~~~bash
$ cat << EOF > ~/quay-secret.json 
    "poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com": {
      "auth": "$BASE64AUTH",
      "email": "openshift@example.com"
    }
EOF
~~~

Now take the quay-secret.json and merge it with an existing pull-secret.json that was retrieved from https://console.redhat.com:

~~~bash
$ cat ~/pull-secret.json | jq ".auths += {`cat ~/quay-secret.json`}" > ~/quay-merged-pull-secret.json
~~~

Now we have a quay-merged-pull-secret.json file which contains the credentials to pull images from Quay.io for OpenShift and also have credentials to push those images into our local Quay openshift4 repository.  Before we can actually execute the mirroring though we need to set a few more variables to tell the mirror command what release we are mirroring, the local registry and repository name and where to retrieve those images from along with our quay-merged-pull-secret.json:

~~~bash
$ OCP_RELEASE=4.10.3
$ LOCAL_REGISTRY='poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com'
$ LOCAL_REPOSITORY='openshift4/openshift4'
$ PRODUCT_REPO='openshift-release-dev'
$ LOCAL_SECRET_JSON='quay-merged-pull-secret.json' 
$ RELEASE_NAME="ocp-release"
$ ARCHITECTURE='x86_64'
~~~

At this point we are ready to mirror the OpenShift images to our local Quay registry.  We can do this with the oc adm release mirror command and by adding the --certificate-authority flag on the end because we are using a self signed cert.  However due to an active [BZ:2033212 ](https://bugzilla.redhat.com/show_bug.cgi?id=2033212) this method will not work yet.

~~~bash
oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --certificate-authority=./quay-ca/ca.crt
~~~

The alternate and again this is only being used because of the self signed cert involved here we can replace the --certificate-authorirty with --insecure=true to enable mirroring:

~~~bash
oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --insecure=true
info: Mirroring 163 images to poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4 ...
poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/
  openshift4/openshift4
    blobs:
      quay.io/openshift-release-dev/ocp-release sha256:d49bf3b88af465a4ae451c11b6d9a0f34d2d9bb17e86da4309971cda1b9b17a2 1.729KiB
      quay.io/openshift-release-dev/ocp-release sha256:47aa3ed2034c4f27622b989b26c06087de17067268a19a1b3642a7e2686cd1a3 1.747KiB
      quay.io/openshift-release-dev/ocp-release sha256:4dec0bb676ac392a9bff40159de60ffb613014ec9489b9cbcf3e757b334d8c63 602.2KiB
      quay.io/openshift-release-dev/ocp-release sha256:ed7bb2714da62ad5492dbd944cef2a3f92df8120ef2c877c03f1975ba01b2382 11.07MiB
      quay.io/openshift-release-dev/ocp-release sha256:01318f016ef2c64d07aebccbb8b60eabb3b68b5905b093f5ac6fce0747d94415 16.03MiB
      quay.io/openshift-release-dev/ocp-release sha256:78f06927a61bc9c7dbdc065cf44d60dcf838d8c4d0291dd59543cc0f6d1f6a30 22.72MiB
      quay.io/openshift-release-dev/ocp-release sha256:eac1b95df832dc9f172fd1f07e7cb50c1929b118a4249ddd02c6318a677b506a 79.44MiB
    blobs:
      quay.io/openshift-release-dev/ocp-v4.0-art-dev sha256:39382676eb30fabb7a0616b064e142f6ef58d45216a9124e9358d14b12dedd65 1.428KiB
      quay.io/openshift-release-dev/ocp-v4.0-art-dev sha256:47aa3ed2034c4f27622b989b26c06087de17067268a19a1b3642a7e2686cd1a3 1.747KiB
 (...)
      quay.io/openshift-release-dev/ocp-v4.0-art-dev sha256:dbd3f6e8a3d34b4b23e03f55ca8203e4de9c091736817fd8f7991d4aa1573cc5 480.7MiB
      quay.io/openshift-release-dev/ocp-v4.0-art-dev sha256:899ee432ff449360f38d26a7c364c840b3edf1acb3e3506bf82b7e9d67a47795 992.8MiB
      quay.io/openshift-release-dev/ocp-v4.0-art-dev sha256:e6bad2226fe207864af563c848bb14f191acb0b8c8fd5a6eaefacaf88f2e2f8b 1.038GiB
    manifests:
      sha256:0131ce0799fc0398044a8fcf51e618360b19c5d414614f2bbd3f93a7aa0868a9 -> 4.10.3-x86_64-vsphere-csi-driver-operator
      sha256:01a313ac0ddec713c37fbb240c57ad8378583df891b9f721a636596cefe70dc4 -> 4.10.3-x86_64-ironic-machine-os-downloader
      sha256:02c63a4a91d5c609b38d283761b60966609990853845e37b848f3cc1fd5de134 -> 4.10.3-x86_64-operator-lifecycle-manager
      sha256:088861d10073be4ff3debc2b88a18f349016f92f2a751b2238141e0e0c758617 -> 4.10.3-x86_64-cloud-network-config-controller
      sha256:0a9a31018ca57706e36fea0a672c5bf57348987d854298e0a40dbe2740448e58 -> 4.10.3-x86_64-ironic-hardware-inventory-recorder
      sha256:0d2383c4994a75d2deae0eb6173d92f1617c98365610ab718cb28c09107e8cde -> 4.10.3-x86_64-ovirt-machine-controllers
      sha256:170003dfa95ff4ea808b3a4de3d5778c7a0b3c683c6ac606571195222c03043b -> 4.10.3-x86_64-cli-artifacts
      sha256:1706eded63d2bc1ffe35674b8416885a00d7ad3a41f3e0584ddcc303842d9bc0 -> 4.10.3-x86_64-docker-registry
      sha256:178accd1823f44d52dd03c08e8be0a445a47d5e3c0fb58484f38fda24c88361a -> 4.10.3-x86_64-cluster-dns-operator
      sha256:17a0307955f8740801280aeaf155637432b0742c5d873b2ee6072594a2a650dd -> 4.10.3-x86_64-oc-mirror
      sha256:1920f7e4ae3542af71612e53cbb7d1e38fb1c0656dddc82df918650640435486 -> 4.10.3-x86_64-prom-label-proxy
      sha256:1a407e98752815ec8bdf027a2fc656f724fa81ed8a0f14dab924b7081acef053 -> 4.10.3-x86_64-openshift-state-metrics
      sha256:1ffc63dd7ad9da285edcd5e2a0e735496ec8ec03cd37f42774edca4003d6dfb1 -> 4.10.3-x86_64-multus-cni
      sha256:2399aa3c16a27632e5e3ea07c6509f4ed80b41b35cf6b05b6ede5cd0172c1058 -> 4.10.3-x86_64-openstack-machine-controllers
      sha256:2428bccc0fbcf9fa023900898c368b4d4fd8b5e50af455b75556721bb0615a12 -> 4.10.3-x86_64-installer-artifacts
      sha256:24d00f66dae4953e7e50a53f2658221f0cb21cd90d7c8055290b58e167abc257 -> 4.10.3-x86_64-oauth-apiserver
      sha256:25e47c484bc5e43197bb6739480817d5e656196958ba75897503a05363710ccb -> 4.10.3-x86_64-cli
      sha256:2629ff9990094c2f8b12076cdec54082bc548ced34e36e24b3718f3df463fe3e -> 4.10.3-x86_64-openstack-cinder-csi-driver
      sha256:277033eb80e3f34b3892913a4d3b19b6c04707c6613086472ef8940df487d003 -> 4.10.3-x86_64-gcp-machine-controllers
      sha256:28de1aee9f74e3cc9ac666b8761f0d9a0c5f625f1703fee17cf617c0a5cc30d3 -> 4.10.3-x86_64-cluster-authentication-operator
      sha256:2900da592ffebdc5bed489ada3584f37a25225f9532cc38e8304e9bf27ae6860 -> 4.10.3-x86_64-baremetal-installer
      sha256:2b23cb1054470d5035115ebf48923c713bc2b186cb4162c5d87d4d0b99b83c9e -> 4.10.3-x86_64-csi-driver-manila-operator
      sha256:2b72364802973bb7f6a39e1694b9fc13e84e13a317863d07f9204c7762e9b29f -> 4.10.3-x86_64-hyperkube
      sha256:2c8fb42f77f8ce6b18b42091f40772994ca93f9730eeedbae20889afede1581f -> 4.10.3-x86_64-jenkins-agent-nodejs
      sha256:2d7dba2a448826648008c88e197f996ad22613fcb23ad24ca910fea57b7a1663 -> 4.10.3-x86_64-azure-disk-csi-driver
      sha256:2d80cc5beffc10607a53eb6c013e87d03e267cdf0b8fc678b40acf5432957714 -> 4.10.3-x86_64-multus-networkpolicy
      sha256:2e23593c2d7cff4086d9ae5edf7429eeb98eb7797538ff2ca2a1d227ee7aedca -> 4.10.3-x86_64-csi-snapshot-validation-webhook
      sha256:2efc1ee4885e441bdc034e986b5f5527395fb7445e0dfb059ebb4f4511b6e1c8 -> 4.10.3-x86_64-cluster-monitoring-operator
      sha256:340dc20ff6a51de93eb313c1fc3f35e32fb8979cd67844c216979d458e28f691 -> 4.10.3-x86_64-ovirt-csi-driver-operator
      sha256:34c5a65a07eaa06bcd2c6c16eb408f227656ecdfb1ebb334b188534f34bfa677 -> 4.10.3-x86_64-tools
      sha256:365b62fa36f52bcaa7482ad98b583b339faf731496a95c778ba7c694b7c0fe3b -> 4.10.3-x86_64-csi-driver-shared-resource-operator
      sha256:366c5973b4a36a7d5219346fd08355dcebd7b846127c0fe8102f4d18a8964f74 -> 4.10.3-x86_64-csi-external-provisioner
      sha256:372d2a367da920e45b756cbff9e9820124f66bb546afa7d085489cd161eff02d -> 4.10.3-x86_64-cluster-network-operator
      sha256:39b70eac529577e4fc515bd0d3686e6eb019c3f5b135e44d76025f4ac96b6dc3 -> 4.10.3-x86_64-kuryr-controller
      sha256:39db32b4d72796e80ed69757f9dbb3928fd422542f23597548053075b69dabdb -> 4.10.3-x86_64-cluster-autoscaler-operator
      sha256:3a461818ec358a21e8a9b6edbd6e106efede3b0e5f6d842b1e1cb1e9e92c50d5 -> 4.10.3-x86_64-telemeter
      sha256:3f6ada6d16f25688e288d8160db5e6d70fb690279bff09f25ca867ee4047374a -> 4.10.3-x86_64-aws-pod-identity-webhook
      sha256:40c04ea43a0d1ed131cc406cbeb9b5192cf1b897129446f573bc1fc9e59eed86 -> 4.10.3-x86_64-cloud-credential-operator
      sha256:423a86af03fc657327f0395abd3d238febfa230082ac84883a3064f683743574 -> 4.10.3-x86_64-ibm-cloud-controller-manager
      sha256:465d123dafe0ba9c6d6e3787047962bc5158067bd9c77cf281f8ed46540a5573 -> 4.10.3-x86_64-installer
      sha256:479990cde02408983f5255ab11209007b1fbe265037f3cd94a8625bfdc191d4b -> 4.10.3-x86_64-cluster-bootstrap
      sha256:4b6376a0a30fd543e43ed46f53722300e58271fe9d615329334c703cd46dbf94 -> 4.10.3-x86_64-pod
      sha256:4d2d5e596a1776ffe93e9b55133a0aecaabef7cfeb1bd3078a7feea931af52bd -> 4.10.3-x86_64-kuryr-cni
      sha256:4f14d32cb853aed1f1c52e4e0e3d94f3d78efdbe9c55061182e332bf7769ae94 -> 4.10.3-x86_64-vsphere-cloud-controller-manager
      sha256:504022f2e4997d205459e4878a551df63878e47408f298d1abd14c1bd97c01e6 -> 4.10.3-x86_64-cluster-node-tuning-operator
      sha256:512b298050fc69e990a4653a6146c399ac02625686976e02e53d7fd2248e4964 -> 4.10.3-x86_64-openstack-cloud-controller-manager
      sha256:52899b1d96fe529814c25850e79c1606eb8433325a26c89dfee45f846fc24a44 -> 4.10.3-x86_64-machine-config-operator
      sha256:538db2ed20579dd71d9cd1af02b47e6dad3f833b8258a95b32b86d2515ef1399 -> 4.10.3-x86_64-operator-registry
      sha256:5458c92486e7446da7fbb71ef6c661ccd76fb85b2de00bea5f4a606ce4a80b5f -> 4.10.3-x86_64-aws-machine-controllers
      sha256:547a3ca1926feefb64a451e359e821a92757ddec0cebcc5e4a1669d96cfda98c -> 4.10.3-x86_64-jenkins
      sha256:55e8aa4d81098ee1174434bbdbfaf884295899c5e4593366a34d6c9144a1ce60 -> 4.10.3-x86_64-cluster-ingress-operator
      sha256:56cfbde30bd41b5868213c1b72653df32222233cde91927ed4f208392c217e9d -> 4.10.3-x86_64-gcp-cloud-controller-manager
      sha256:5778d32b4969d06e5bd50f4ab5494ee910bac0d2dae820bdfaec76d8a992242d -> 4.10.3-x86_64-ibm-vpc-node-label-updater
      sha256:5abd4776c235a7b436171959b49fb3eb3cdacd4c8b48b922dd8f8fdf7f1322c8 -> 4.10.3-x86_64-libvirt-machine-controllers
      sha256:5b1e3eab4f1c3f07d5226dcc1f366fcb56485c35066d13d0e4166affa5f5e02b -> 4.10.3-x86_64-jenkins-agent-base
      sha256:5b8f5bed0af950aad1e67b8cf741c65b423d5d1dbd15465dffd43b3dd92c44e2 -> 4.10.3-x86_64-ibm-vpc-block-csi-driver-operator
      sha256:5bb712d35ad20eb482fb0e589d6cac77a2678cdb38dd654155e6ac6dad45988e -> 4.10.3-x86_64-azure-file-csi-driver-operator
      sha256:5ce730df6a81a605635e48cbe2369f6f35cd8559db78b870bab2b8935885de89 -> 4.10.3-x86_64-azure-cloud-node-manager
      sha256:5d7c177423eac9a7f678583add82f11eb9dd776f763315e52387fd9e57419b0a -> 4.10.3-x86_64-network-metrics-daemon
      sha256:60619bd4e26671d18acc06b8469fc65cbb23a23b7130a5b88a9d73a8c3adec0a -> 4.10.3-x86_64-cluster-config-operator
      sha256:6105b9d0b950a0c8284f6e866133b3592add721fdc059fa5af5f96da8beff585 -> 4.10.3-x86_64-docker-builder
      sha256:64d573d13a4e16c91e93f283d773d730cb25e522dec53f25dd359ebfc56c43e3 -> 4.10.3-x86_64-multus-route-override-cni
      sha256:675694ddf83d03d7a58c27fa14d6abd3798612a22d95a387671c038c5bdc9857 -> 4.10.3-x86_64-csi-external-snapshotter
      sha256:67cbf9d98160ea9a1ad823bdb1c616fda12e6e11a58fc49c79238a5a88c6cda1 -> 4.10.3-x86_64-prometheus-operator
      sha256:69776e1c5ffa73e67a403e4958ea1cfc829ea3b19c405a2bf6090b611b066ca4 -> 4.10.3-x86_64-cluster-samples-operator
      sha256:6a1c576990cf58a1569be59202cb0023180b12b7001105648c6a446eda4b1746 -> 4.10.3-x86_64-network-interface-bond-cni
      sha256:6a21237e2070cb4e297bf80411729aaa3fd6afe88b5662f9af1bcaf61b26528f -> 4.10.3-x86_64-operator-marketplace
      sha256:6a61353191015cb155bee7dffbfc1f40c705cbcdefa0f34d789e95039616947a -> 4.10.3-x86_64-oauth-proxy
      sha256:6b2caece0410e35ed49491b63fc800a541ee5e69385b2e012fc6d9603efd47c2 -> 4.10.3-x86_64-kube-storage-version-migrator
      sha256:6dccd26949a4523840e4610a5b28938a0f6094d5ac57617de734be40133a46fd -> 4.10.3-x86_64-coredns
      sha256:6e08ccb940b37ced85c92179e913a5878a27ff69b77234db24722a1474243767 -> 4.10.3-x86_64-baremetal-operator
      sha256:70baef8e0f932a66ca16738e0ddba5148645bbd9947b40ec4cec63630dd33c39 -> 4.10.3-x86_64-alibaba-cloud-controller-manager
      sha256:73a00e95d08cce0cf5e34602d160f4de72da69ffdd05a347fa705ee6e13cae7c -> 4.10.3-x86_64-cluster-kube-apiserver-operator
      sha256:76c63c436053bcdbe4759096923e0af6e8f63256f6fb9aa99a0fe8488a4048c1 -> 4.10.3-x86_64-csi-driver-nfs
      sha256:7bf3966bb6a2b310342425f6a6c59c105f3d128ec572c3fd043b6a09f66f1f48 -> 4.10.3-x86_64-configmap-reloader
      sha256:7ce1b6c6a6c7f940132dcd516a5856175fa164626a037bf71c26b0561935c27b -> 4.10.3-x86_64-cluster-policy-controller
      sha256:7ffe4cd612be27e355a640e5eec5cd8f923c1400d969fd590f806cffdaabcc56 -> 4.10.3-x86_64
      sha256:83786a893d20d11556b1be5fb036556b2897e5d1f928f6c6dde57a2af0636cdc -> 4.10.3-x86_64-cluster-csi-snapshot-controller-operator
      sha256:83b057ea7dfc855e4a93dc92f052ab2d942ba0744bc91f92d78e753109ecf13d -> 4.10.3-x86_64-multus-admission-controller
      sha256:84e2c8281030b1b41eac8cb37988aa4025890d6e0defc20599d7e7bc907f97a3 -> 4.10.3-x86_64-azure-cloud-controller-manager
      sha256:867ef8c9bf8946c0bc0f01138fceec52d41409be2d035a8584b7646f2eb3b3c7 -> 4.10.3-x86_64-openshift-apiserver
      sha256:8705564ee52c5b655529860687f73db78daef4eaa54f5fdd3160cfbfc4aac438 -> 4.10.3-x86_64-driver-toolkit
      sha256:87cfe1562cc78f7a1100df7f860e8e51fe01cc9e7d753e8881caccb5b16abc4e -> 4.10.3-x86_64-cluster-machine-approver
      sha256:88852bf51fe05b30a197a0688b73abf38a85d5f938211462b566fff7ff88be6f -> 4.10.3-x86_64-powervs-machine-controllers
      sha256:88eb01dc7d3aa4c8bb4f15b13ce802a057782fa20252885db005b0b20ed87542 -> 4.10.3-x86_64-ibm-vpc-block-csi-driver
      sha256:88ffa8c48906949177beef1d6c7527331e759eb6016b40360486df0607bc7c6b -> 4.10.3-x86_64-aws-cloud-controller-manager
      sha256:8aa5e5c8750b41887c0366b5b5e285d927a80050b43f92717dddf9c7cf2ba1a3 -> 4.10.3-x86_64-ironic
      sha256:8b4e85188cbc22b62e1091bd068622663656986048e62a0eac913110028e8a41 -> 4.10.3-x86_64-csi-node-driver-registrar
      sha256:8c3df2e8e5911ce2409c00a15376fb69d657c6626a8aa172ffdeb7ebf7fedfea -> 4.10.3-x86_64-csi-snapshot-controller
      sha256:8ec836136d39baa7807780e6e443cbcc189107d530ad27988df50f8f6d10b479 -> 4.10.3-x86_64-haproxy-router
      sha256:904f557b93f24296e38980df04633a29fb5156051c5c13850c15648433c53fcf -> 4.10.3-x86_64-tests
      sha256:91d672a9423a60e7eaed7355923e9a8852ae83a5036aa4420b23cc9241957806 -> 4.10.3-x86_64-csi-livenessprobe
      sha256:942a2875546f667b9fb91e540fe39bc17747a9dee04ba5eb9f462b5ada88399b -> 4.10.3-x86_64-prometheus-alertmanager
      sha256:9439e3e873a7b9005d08f0afc33b13252c4fe79630496d11cd659ca7327d96f6 -> 4.10.3-x86_64-deployer
      sha256:9539951c1c9bde09b4cdc8cf126215218ea57a9851a4452d074c9f341c3cb350 -> 4.10.3-x86_64-openstack-machine-api-provider
      sha256:98ad96d0f1dcc35047698878860c69da7f9d3c0797da60fceaa6a0fa0e311ce6 -> 4.10.3-x86_64-openshift-controller-manager
      sha256:9a2bb19b5aac5d32c11d4b649311d05eccc501b5e2ff447522e98e4d3f50a4bc -> 4.10.3-x86_64-cluster-kube-storage-version-migrator-operator
      sha256:9c24cf4b07aef7acc00bf99ad1ba98a97998a90cb355e85639d46b90b32ee44d -> 4.10.3-x86_64-egress-router-cni
      sha256:9d979a80795052277a036425c933a556c1eea56861edb6a5de2c3134e5965304 -> 4.10.3-x86_64-vsphere-csi-driver
      sha256:9f9bda4a6fa6e591285fb252b4f984ab6e92b372295bacf7c02f231dd8adaea8 -> 4.10.3-x86_64-csi-driver-manila
      sha256:a11a9e3d8de3384a2b169a63aa7b84578284c8458b474471ad4a82de8b7d1b8c -> 4.10.3-x86_64-cluster-autoscaler
      sha256:a340ea3d86560de8cf9ee2c1afe42ad24e5eefc9903a0073abe9dc54815bf710 -> 4.10.3-x86_64-console-operator
      sha256:a3fc7a9880e5fe4dd7ad6a8f7f18eb72eaaad87b360166811c8081f97f2895cb -> 4.10.3-x86_64-cluster-baremetal-operator
      sha256:a52941a2bddc9abf32d9a3a4763252cc40c1a98a319a2c91da8c8cb74e23f153 -> 4.10.3-x86_64-multus-whereabouts-ipam-cni
      sha256:a52bf52659062209293a78923d5f3255505469a4ff2f3e2632c846a64592a980 -> 4.10.3-x86_64-cluster-kube-controller-manager-operator
      sha256:a68198dbebb310565dd0a70475135b2618779606bb5a12e488bb64708eabec9c -> 4.10.3-x86_64-cluster-capi-operator
      sha256:a9b9ab81b60f0f0d86e0730a63ef126400ab6236383f9d4cfd432d77154299ed -> 4.10.3-x86_64-k8s-prometheus-adapter
      sha256:abf9059fc4d4e1b710c5471f3832e4b0df8dcd51109763a0f37c427beb49ffb8 -> 4.10.3-x86_64-csi-external-resizer
      sha256:ad307371d2b92ef76828c8e73ffd739af4dd07cede8a3cde13c72ee4644c4769 -> 4.10.3-x86_64-cluster-openshift-apiserver-operator
      sha256:ad3a9cdaa03026106d2f944e94436f9c28ab90b4720ff403afff54f676ddbb75 -> 4.10.3-x86_64-sdn
      sha256:ad7b660f9ff09c227dacc4a242008ce7628cf3357b2d72f3f9d44a06eea2933c -> 4.10.3-x86_64-keepalived-ipfailover
      sha256:affe6ea2043e6b584c0129a5691b365cc6455728863ab9f63de613aa7c66f6ea -> 4.10.3-x86_64-openstack-cinder-csi-driver-operator
      sha256:b0c5145966edc43f3f2d77f5b790e16ad3ec035f3d7170126125a4b796eab218 -> 4.10.3-x86_64-ovirt-csi-driver
      sha256:b0fb5eee9c9619d0fc66b67e0cfe61f697e531a22af5c5639e6b65de6542fe86 -> 4.10.3-x86_64-ironic-static-ip-manager
      sha256:b3471af90ab21d77351703e88b9973bd455cdb6435935c9cf9f2696b27cb7d21 -> 4.10.3-x86_64-azure-disk-csi-driver-operator
      sha256:bca7251b67e28bef37f543568b906bf27f72a01d65f5335d768ea3fa810d0a3c -> 4.10.3-x86_64-grafana
      sha256:be5ad7c623910ef4fb75078e51b46083d6a2aa90af6486ebe228cae3feb9344d -> 4.10.3-x86_64-cluster-cloud-controller-manager-operator
      sha256:beb9dcff1b9bdb6ad7ee43beb70c01e320ac492f5f1a96d06244a166d4698b4f -> 4.10.3-x86_64-image-customization-controller
      sha256:bfd50abec62b4b89cc929ff0a15005b039d29655fa107cbd628d96a206c91b9c -> 4.10.3-x86_64-console
      sha256:bffba87818dc2d4360ff0649accadcd0bcde418a1c7acbba9476afdd5d2ee9a5 -> 4.10.3-x86_64-cluster-version-operator
      sha256:c1ddff974157e74cf75f887ac16503ce38e16728e6960ae7ae7e80f2ce4bcf1f -> 4.10.3-x86_64-machine-api-operator
      sha256:c394713d976b1fca8c8530bae9dab9b8e82602ea7d9d397bf6d455b8b5a88ccc -> 4.10.3-x86_64-csi-driver-shared-resource
      sha256:c41a5a65c5a0e1d7626d4d1e1976a5ff73d374ca947b482f8cdd46288a381d95 -> 4.10.3-x86_64-prometheus
      sha256:c53385a901aee40c78a23cd7b7193afe46152eb9945bda56579fb83319ce589f -> 4.10.3-x86_64-oauth-server
      sha256:c74555756486796c95fb4015876ef1527d31aa26331055d7950aa9fd82f63d70 -> 4.10.3-x86_64-thanos
      sha256:c983273151cbb08875640ef4b319807f424e03f9f3f48a70c0ee066b57efd81d -> 4.10.3-x86_64-cluster-capi-controllers
      sha256:ca7f898bc0724f8583007081f7d8e66aeec328763798ebb2bb5475c4c48ede7d -> 4.10.3-x86_64-aws-ebs-csi-driver-operator
      sha256:cbc888fdb8f2c639e7a62927bc75a9132d5c5a965fa471b67b77e27b03139093 -> 4.10.3-x86_64-etcd
      sha256:cc7e1f905a28c0a0ff779ff5355958a4619317619b175f003c0331ef7e991b3b -> 4.10.3-x86_64-baremetal-machine-controllers
      sha256:cc91a5999ed5d933ac183e1f326ba66aacb262beb40a48eb079ba74cdf345a6a -> 4.10.3-x86_64-machine-os-images
      sha256:ccebf57d0beaee32beb8aeec3608bcbdab2ae661b3052ce8f1df8552289dd9bd -> 4.10.3-x86_64-aws-ebs-csi-driver
      sha256:cf42488f36ac11a2c14d91bf7a25252f18f811da9c914efc57dfb60128c28b3c -> 4.10.3-x86_64-azure-file-csi-driver
      sha256:d213d1b43a0b45c3b9383e903c5a0bac1556977dc49077603622460fae326df7 -> 4.10.3-x86_64-jenkins-agent-maven
      sha256:d22995a5fbec291a332fb5b391202b3c82c95e6026168c29f9488ff7995d5c3f -> 4.10.3-x86_64-cluster-kube-scheduler-operator
      sha256:d846a05f9ced30f3534e1152802c51c0ccdf80da1d323f9b9759c2e06b3e03b4 -> 4.10.3-x86_64-vsphere-problem-detector
      sha256:d972b7a0349b2d2cb42df09d1e854ab0e0dc6919404d37a309a4dd94fee41619 -> 4.10.3-x86_64-csi-external-attacher
      sha256:db66145d4d9fe8b408211918508bc2db90dc2b2f34ea64cbe8a1c289df5d34a0 -> 4.10.3-x86_64-vsphere-csi-driver-syncer
      sha256:dd285a402d7f0aaf7978164d6900bc1154e2f3633f5008dfc6105a4e689bacfd -> 4.10.3-x86_64-gcp-pd-csi-driver-operator
      sha256:ddfec107df7b9406de1c1660ff83506f0dc7c190c1b0855685af64b3ab38ec39 -> 4.10.3-x86_64-network-tools
      sha256:df3b0ec40395ea460fbf2728ca7adff79dbaebddcffce003e88b3fb9cb2c9759 -> 4.10.3-x86_64-must-gather
      sha256:e03515e236623aeafd4493bca0fcc8d85775f483fdeb9417dfccc6ae7b6d6523 -> 4.10.3-x86_64-kube-proxy
      sha256:e0505809ab1ee5471518264733a16b9d3f256ccc239f6ecb5d1a0acbe01827db -> 4.10.3-x86_64-cluster-openshift-controller-manager-operator
      sha256:e052ee37b444aac4602a89f583cca0ed2ba976ba8d3d2e48882263a2e4c8a5a0 -> 4.10.3-x86_64-baremetal-runtimecfg
      sha256:e1a2e68265667bf616b9bc68ec255758dc60d85dae77a54391a820755a256f55 -> 4.10.3-x86_64-ironic-agent
      sha256:e5a9f405dfe938b8b5ffa331f3feb3ca472860cb960aee9244cd58dec7510b20 -> 4.10.3-x86_64-azure-machine-controllers
      sha256:eba010c5a6130d31c2fe5287436894955cc086ff2be34859967880adca234e86 -> 4.10.3-x86_64-cluster-update-keys
      sha256:ec276e99bc4f4f057567fb0b613f2e0fbcf18a8488ba04ff4c78986f4ff94bd5 -> 4.10.3-x86_64-ovn-kubernetes
      sha256:ec806f2f9be1fdbd85bd0d096118e69b1e709b13cc8660e3c04d5a4678ce463c -> 4.10.3-x86_64-alibaba-machine-controllers
      sha256:ecb05331404a88c40518d223a8dfd8cbd3b147e4f68b23836e360ad65dab9e85 -> 4.10.3-x86_64-machine-os-content
      sha256:f12c7f1901daef65b43c83aa2b57a826cc79213ce9b111c289bbd6881ebdb9e7 -> 4.10.3-x86_64-prometheus-config-reloader
      sha256:f1472992e770cc0194da355264653e3a7c82448cb79f4fc0f051a3277291b0a9 -> 4.10.3-x86_64-kube-rbac-proxy
      sha256:f334fb8c2faa58096f386706ce8296d95e3d4eaa1e090dd097447795b49837a4 -> 4.10.3-x86_64-alibaba-disk-csi-driver-operator
      sha256:f51c580925a4275362a09537aa2de0fb28e17115052694a30da6c5126ff591cd -> 4.10.3-x86_64-container-networking-plugins
      sha256:f532e7f50cadfc757a6d277ab5476a31a4f24aac0afc615da6ff9f7ce5e2b538 -> 4.10.3-x86_64-cluster-image-registry-operator
      sha256:f5a09598e8563b5d697d8a838692c0644cf16233d049e773a0a37e8f1542d64e -> 4.10.3-x86_64-ibmcloud-machine-controllers
      sha256:f835160f1dc29995853eb1c13aa9e2badbb45c8656947b6eb6faefc876aab3ee -> 4.10.3-x86_64-gcp-pd-csi-driver
      sha256:f920b01e779078c70882990e364e8f082585901685bb75c13cb8f6ad5eed445f -> 4.10.3-x86_64-prometheus-node-exporter
      sha256:f99067f5216d7b8da713c7a6843853e052f8439314d526ed1c998873e30d5763 -> 4.10.3-x86_64-service-ca-operator
      sha256:fa62c66f23f2af0f1fd3a32491e4fc5c97c31320dd985ebcbc62ba1e749dba82 -> 4.10.3-x86_64-kube-state-metrics
      sha256:fb5a1cde7e60cf4de7bf4256632bd31927a01b8ab5977b6f16c2b4df8c1c3f1e -> 4.10.3-x86_64-insights-operator
      sha256:fcbe2f72fdf333bbdcbf60e1c001d666e32046f1321b383f79176d2273a78d1c -> 4.10.3-x86_64-cluster-storage-operator
      sha256:fcc68fcb4d57bbb544f18996bb9a12859db31d2dfe02573c3b11c028dbee2a92 -> 4.10.3-x86_64-alibaba-cloud-csi-driver
      sha256:fe8d01e2de1a04428b2dfd71711a6893ea84ad22890ea29ca92185705fc11f48 -> 4.10.3-x86_64-cluster-etcd-operator
  stats: shared=5 unique=333 size=11.57GiB ratio=0.99

phase 0:
  poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com openshift4/openshift4 blobs=338 mounts=0 manifests=163 shared=5

info: Planning completed in 31.84s
uploading: poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4 sha256:edde8c98215a8e3327cf733ff3185b5ed5cb2c6410c87e0aace449b34f597d2a 28.01MiB
(...)
uploading: poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4 sha256:207b6a2aaa489392ce1a7b78263f4b2b0174cb2b892a2ca4af66e0d4d80e0c2f 30.75MiB
uploading: poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4 sha256:56d6777567c06a64b053a28b6a841cffb065ceae203c82f4182a34fecefeb17d 114.8MiB
sha256:7bf3966bb6a2b310342425f6a6c59c105f3d128ec572c3fd043b6a09f66f1f48 poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4:4.10.3-x86_64-configmap-reloader
(...)
sha256:8c3df2e8e5911ce2409c00a15376fb69d657c6626a8aa172ffdeb7ebf7fedfea poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4:4.10.3-x86_64-csi-snapshot-controller
sha256:fe8d01e2de1a04428b2dfd71711a6893ea84ad22890ea29ca92185705fc11f48 poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4:4.10.3-x86_64-cluster-etcd-operator
sha256:01a313ac0ddec713c37fbb240c57ad8378583df891b9f721a636596cefe70dc4 poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4:4.10.3-x86_64-ironic-machine-os-downloader
info: Mirroring completed in 1h14m31.94s (1.292MB/s)

Success
Update image:  poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4:4.10.3-x86_64
Mirror prefix: poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4
Mirror prefix: poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4:4.10.3-x86_64

To use the new mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - poc-registry-quay-quay-poc.apps.magic-metal-cluster.simcloud.apple.com/openshift4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
~~~

When the mirroring is completed save the output for the ImageContentSourcePolicy as that will be used wen depoying any clusterN+1s and point them to the proper registry for the OpenShift images.
