# **Quintessential Red Hat Quay Quickstart**

<img src="Logo-Red_Hat-Quay-A-Standard-CMYK.jpg" style="width: 1000px;" border=0/>

Red Hat Quay is a secure, private container registry that builds, analyzes and distributes container images. It provides a high level of automation and customization. Red Hat Quay is available via a deployed Operator in OpenShift or as a standalone component on a server.  In this blog we will focus on the deployment method using the operator on OpenShift and do all of the installation via the command line.  Note that this blog is not the only way to deploy Red Hat Quay but rather an opinated method that provides an example of using the command line along with the Quay API to get a Red Hat Quay up and operating quickly.

Before we dive into the steps of creating our Red Hat Quay instance we want to cover the environment we will be using for this demonstration.  The environment is a Red Hat OpenShift 4.10.10 three node compact cluster where the nodes are acting as both the control plane and workers.  For convenience we have also installed the OpenShift Data Foundation across the the three nodes to provide default storage for the cluster.  Since OpenShift Data Foundation provides Noobaa Red Hat Quay will be able to take advantage and it instead of deploying its own instance of Noobaa.

The first step to deploying Red Hat Quay is to deploy the operator and that first requires building a subscription custom resource file like the one below:

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
  startingCSV: quay-operator.v3.6.6
EOF
~~~

Once we have subscription custom reosurce file created lets go ahead and create the subscription on our cluster:

~~~bash
$ oc create -f ~/quay-operator-subscription.yaml 
subscription.operators.coreos.com/quay-operator created
~~~

We can validate that the subscription was successful by checking if the operator is running and confirming the subscription is present:

~~~bash
$ oc get pods -A|grep quay
openshift-operators                                quay-operator.v3.6.6-5945bfbbc9-kh692                             1/1     Running     0          48s

$ oc get sub -n openshift-operators quay-operator
NAME            PACKAGE         SOURCE             CHANNEL
quay-operator   quay-operator   redhat-operators   stable-3.6
~~~

With the Quay operator running we next need to create a namespace where the Red Hat Quay registry will run the require pods:

~~~bash
$ oc create namespace quay-poc
namespace/quay-poc created
~~~

Once the namespace created we can move onto generating an initial config.yaml for Red Hat Quay.   In this configuration we are going to specify a quayadmin super and enable the ability to use the Red Hat Quay API:

~~~bash
$ cat << EOF > ~/config.yaml 
FEATURE_USER_INITIALIZE: true
BROWSER_API_CALLS_XHR_ONLY: false
SUPER_USERS:
- quayadmin
FEATURE_USER_CREATION: false
EOF
~~~

Now lets take the config.yaml we created and generate a secret config bundle which we will store in the quay-poc namespace we created:

~~~bash
$ $ oc create secret generic --from-file config.yaml=config.yaml init-config-bundle-secret -n quay-poc
secret/init-config-bundle-secret created
~~~

With the init-config-bundle-secret created we will now reference it in our quay-registry custom resource file.  For this example we are keeping the custom resource file simple and allowing the operator to control every aspect of the Red Hat Quay deployment.  However if one chose to use external S3 storage or an external postgres DB this would be the file where those modifications could be made.  

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

Take the quay-registry custom resource file we created and apply it to our cluster:

~~~bash
$ oc create -f ~/quay-registry.yaml
quayregistry.quay.redhat.com/poc-registry created
~~~

After about 3-5 minutes we can validate that Quay is running by running the following command:

~~~bash
$ oc get pods -n quay-poc
NAME                                              READY   STATUS      RESTARTS        AGE
poc-registry-clair-app-6c64748b8b-6j7x6           1/1     Running     0               2m18s
poc-registry-clair-app-6c64748b8b-htt2q           1/1     Running     0               2m7s
poc-registry-clair-postgres-5d66c9856c-8ftdt      1/1     Running     1 (5m41s ago)   5m50s
poc-registry-quay-app-f9784fd67-hvtwg             1/1     Running     0               2m18s
poc-registry-quay-app-f9784fd67-v7q4n             1/1     Running     0               2m18s
poc-registry-quay-app-upgrade-jsk6k               0/1     Completed   0               2m26s
poc-registry-quay-config-editor-6b4b8df78-4kntq   1/1     Running     0               2m18s
poc-registry-quay-database-f7f95c644-nwcsf        1/1     Running     0               5m50s
poc-registry-quay-mirror-6c799c6d6f-b5nmc         1/1     Running     0               2m16s
poc-registry-quay-mirror-6c799c6d6f-n8g78         1/1     Running     0               2m5s
poc-registry-quay-postgres-init-hkjt5             0/1     Completed   0               2m29s
poc-registry-quay-redis-f8b57fc77-9t9sb           1/1     Running     0               5m50s
~~~

At this point we have a working Red Hat Quay deployment but we need to do a few configuration steps in order to establish a registry we can use to perform push/pulls against.   Before we can go through those steps however we need to ensure we have a few tools we will leverage: jq, podman, curl and oc:

~~~bash
$ which podman
/usr/bin/podman
$ which jq
/usr/bin/jq
$ which oc
/usr/local/bin/oc
$ which curl
/usr/bin/curl
~~~

If any of the tools are not installed above please refer to the documentation on how to install or obtain them for the operating system being used to step through this example in the blog.

With our tools in place we can now begin the configuration of Red Hat Quay which we will do via the Red Hat Quay API.  The Red Hat Quay application programming interface (API) is an OAuth 2 RESTful API that consists of a set of endpoints for adding, displaying, changing and deleting features for Red Hat Quay. 

The first step is to create the initial quayadmin user which also happens to be a superuser.  A superuser is a user account that has extended privileges, including the ability to manage users, organizations and service keys.  They also have the ability to view change logs, query usage logs and create globally visible user messages.  To create the user we will use curl and post to send the json data payload to Red Hat Quay.  Inside this payload will be the username, password, email and access token flag:

~~~bash
$ curl -X POST -k  https://poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/api/v1/user/initialize --header 'Content-Type: application/json' --data '{ "username": "quayadmin", "password":"password", "email": "quayadmin@schmaustech.com", "access_token": true}'

Output:
{"access_token":"OWE7BAOYNJMUGFS71DRKROXGW6RDD5UMNIHVMDSE","email":"quayadmin@schmaustech.com","encrypted_password":"usLgMuipLJyeMDxFg/soq2ZF2s6e8dhiRnQ1cii2wwbMEkH7Irein5yglpnnNQmZ","username":"quayadmin"}
~~~

When we created the quayadmin we got an access token back in the output.  This is access token is short lived but can be useful for configuring the rest of the API steps below.  Because of this lets go ahead and assign the access token to the variable TOKEN:

~~~bash
TOKEN=OWE7BAOYNJMUGFS71DRKROXGW6RDD5UMNIHVMDSE
~~~

Now that we have set the access token lets move onto creating a few more things for our Red Hat Quay registry configuration.   One item we need is a organization. Organizations provide a way of sharing repositories under a common namespace that does not belong to a single user, but rather to many users in a shared setting (such as a company).  Again we will create this via a curl and post via the API.  In this example the organization is called openshift4:

~~~bash
$  curl -X POST -k --header 'Content-Type: application/json' -H "Authorization: Bearer $TOKEN" https://poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/api/v1/organization/ --data '{"name": "openshift4", "email": "openshift4@schmaustech.com"}'

Output:
"Created"
~~~

Inside the organization we just created we can now create a repository.  The repository is created under the organization to store the actual images for the various applications one might store in a registry.  In this example we will also call the repository openshift4 and use curl and post via the API.  The json data payload in this example is the namespace(organization), repository, visibility status(public or private), a brief description which we left empty and the kind of repository:

~~~bash
$  curl -X POST -k --header 'Content-Type: application/json' -H "Authorization: Bearer $TOKEN" https://poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/api/v1/repository --data '{"namespace":"openshift4","repository":"openshift4","visibility":"public","description":"","repo_kind":"image"}'

Output:
{"namespace": "openshift4", "name": "openshift4", "kind": "image"}
~~~

In order to get images into the repository we just created we need to have a user associated to the repository.  While we have the quayadmin user already created we really should create another user that will be the user to pull/push images into our openshift4 repository.  In the example below we will create a user called openshift using curl and post via the API.  The json data payload will contain the username, and email address associated to user and the access token flag:

~~~bash
$  curl -X POST -k  https://poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/api/v1/superuser/users/ -H "Authorization: Bearer $TOKEN" --header 'Content-Type: application/json' --data '{ "username": "openshift", "email": "openshift@schmaustech.com", "access_token": true}'

Output:
{"username": "openshift", "email": "openshift@schmaustech.com", "password": "KUQ92HPRIAG9O2FQBA47879C28CDJ932", "encrypted_password": "UUNQLLSlBHq9Mfl1hKv22Ant17SZ5IdayZqzmArVXm9xqQo5to38q+uuP1zp9lUrJ4sFOZ/bYtvOhiUCrhSeSyWkBnoeEqF3vzEq89B0Brg="}
~~~

Upon creation of the user we get get an encrypted password back for that user.   We are going to assign the username and encrypted password to a couple of variables which we will use later to generate a pull secret for this Quay registry:

~~~bash
QUAYUSERNAME=openshift
QUAYPASSWORD=UUNQLLSlBHq9Mfl1hKv22Ant17SZ5IdayZqzmArVXm9xqQo5to38q+uuP1zp9lUrJ4sFOZ/bYtvOhiUCrhSeSyWkBnoeEqF3vzEq89B0Brg=
~~~

After creating our user and setting our variables we now need to associate the user to the organization we created.  We will do this using a curl and put command against the organization/owner/members path and specifying the username at the end:

~~~bash
$  curl -X PUT -k  https://poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/api/v1/organization/openshift4/team/owners/members/openshift -H "Authorization: Bearer $TOKEN" --header 'Content-Type: application/json' --data '{}'

Output:
{"name": "openshift", "kind": "user", "is_robot": false, "avatar": {"name": "openshift", "hash": "0429fc01dc57217bf6282f0a95b6b23f0a3f0bc11b871711170b0a57d65a3644", "color": "#2ca02c", "kind": "user"}, "invited": false}
~~~

We will also need to give this user a role on the repository.  Roles can be read, write or admin and in our example we are going to give the openshift user full admin rights this repository.   We will do this again using a curl and put command via the API.   We will also send along a json payload with the role defined in it:

~~~bash
$  curl -X PUT -k  https://poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/api/v1/repository/openshift4/openshift4/permissions/user/openshift -H "Authorization: Bearer $TOKEN" --header 'Content-Type: application/json' --data '{ "role": "admin"}'

Output:
{"role": "admin", "name": "openshift", "is_robot": false, "avatar": {"name": "openshift", "hash": "0429fc01dc57217bf6282f0a95b6b23f0a3f0bc11b871711170b0a57d65a3644", "color": "#2ca02c", "kind": "user"}, "is_org_member": true}
~~~

At this point we have completed all the configuration steps we needed to via the API.  Just to recap we created the quayadmin, an organization, a repository, another user and associated that user to the organization and repository.  Now we can move onto actually putting something into this registry we built.

First we need to extract the ca.crt from the new Red Hat Quay installation.  I should note though this is only required when using self-signed certs like we are using in this example.  If the cluster is deployed with certs from a real certificate of authority then this would not be required.  To extract the CA for the self siged cert create the temporary directory of quay-ca and then using the oc command extract the CA from the site and save as ca.crt:

~~~bash
$ mkdir ~/quay-ca
$ oc get secret -n openshift-ingress $(oc get deployment -n openshift-ingress router-default -o jsonpath='{ .spec.template.spec.volumes[?(@.name=="default-certificate")].secret.secretName }') -o jsonpath='{ .data.tls\.crt }' | base64 -d > ~/quay-ca/ca.crt
~~~

Lets confirm we extracted the certificates by looking at the output:

~~~bash
$ cat ~/quay-ca/ca.crt
-----BEGIN CERTIFICATE-----
MIIDbzCCAlegAwIBAgIIe066IGTo1rQwDQYJKoZIhvcNAQELBQAwJjEkMCIGA1UE
AwwbaW5ncmVzcy1vcGVyYXRvckAxNjQ4NzYyOTgzMB4XDTIyMDMzMTIxNDMwM1oX
DTI0MDMzMDIxNDMwNFowJzElMCMGA1UEAwwcKi5hcHBzLmtuaTIwLnNjaG1hdXN0
ZWNoLmNvbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALdQpmrZqfV0
9b71aX5/e9NzjCPqVipSYXom7c6EAiYDFN6HNdZB29Us+TButGJR5QQi5u2Uc0bO
JsUWjc9oi2eaksIa6WQeUda5VwqOQZNL01Q7ElW/TWGZvlU6vEv0tUaPwJdT2hJR
v/2CWnEqRPnAcu0MJ6j6c67JPmMX1x7Hw6+9HatcQownSx7QayV0mYLm+h6qqQKG
xcRgFYV137Qu6sDGdNyBsDYuzvk/0fupppxnCR8FuQ7CxthfzVR7TxhyPYZVkis0
+Xds+m9lruCqr2moVFVu2qCXpDD+U2KoqQ5V64WYPbiD9RyMxDSxaUoYC1ejOWDv
SNNYxeTNgIcCAwEAAaOBnzCBnDAOBgNVHQ8BAf8EBAMCBaAwEwYDVR0lBAwwCgYI
KwYBBQUHAwEwDAYDVR0TAQH/BAIwADAdBgNVHQ4EFgQUsRTnQi1pWk9+k0i49/xw
D9nT+AgwHwYDVR0jBBgwFoAUJZHWQoXYwZcSHNyQF0UIYIeATTIwJwYDVR0RBCAw
HoIcKi5hcHBzLmtuaTIwLnNjaG1hdXN0ZWNoLmNvbTANBgkqhkiG9w0BAQsFAAOC
AQEAfdF4/qrYdeYJuxyJVSRdIO0xK6cLtHDIAbHG6vRXhktUxpofTYivJUtVHfKS
K5/CKc6cPEo/5LXXc9kgo4NJ/+OXL9qInBjsLQ4O8Re6BMctsTwkRp/McDii/9vY
ZqSzREGGgtfDXSOl+gUoaW1lpr9xBW+mMGqgOFOKHTy4YRaVhG148l5s61mY50z4
/0b/IG085ws/b7r8PYcTIpABh8+YexbewaZNNirMdt1sRypYFvvW7NlZwg+l525+
mYteBQ5342aZIowJ2RaViy49dGUlB/gYkgFrPqcwGpNJB8H3C9Q0AlCTl6kO11Io
0Y6zkjJnZ0B3LaRwLlYGxKYYTQ==
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
MIIDDDCCAfSgAwIBAgIBATANBgkqhkiG9w0BAQsFADAmMSQwIgYDVQQDDBtpbmdy
ZXNzLW9wZXJhdG9yQDE2NDg3NjI5ODMwHhcNMjIwMzMxMjE0MzAyWhcNMjQwMzMw
MjE0MzAzWjAmMSQwIgYDVQQDDBtpbmdyZXNzLW9wZXJhdG9yQDE2NDg3NjI5ODMw
ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDARKFxZzoKLW/VExzKoe/U
D6nIgRBGNIiWNN4wlrSw61y4sC7K0qRhie4Nu8EzlUyPhafCBli99M7sf1hqSwMK
BQ6zchv53yxyspamB7YtUuUGX0+h1bCSRDptHsMpSIChYHVV4fimynhisl2gbFd+
c/BoqOc+vHGY1RyGAsOKdnLC7WABTDlfDkxxLTciNQadW0gIZ3ZCSd68PC72eU1s
VeDJOONJhGvgCpT9Iujn2g2hCJcax2pzthD9orqtxurOW0hgaav8ruIKegcUPDuc
hFG60nQz+kzTPJgQg1h7MHDlR4GcteFODITehbfiHFQYb65LfcJDB6fzAiULZSEv
AgMBAAGjRTBDMA4GA1UdDwEB/wQEAwICpDASBgNVHRMBAf8ECDAGAQH/AgEAMB0G
A1UdDgQWBBQlkdZChdjBlxIc3JAXRQhgh4BNMjANBgkqhkiG9w0BAQsFAAOCAQEA
PegMoRlTrw/q/8QFunbsN9RQz1BvCh7V+MCKZHU3IuJ+J7HYCN2ZokuTnS5DN9JI
4M52XOtFdgcMVHeJiYofkvHiNSWZ3xHQC5+Zq8c6ComQxTsMionnG82uKwBTJbKP
gETzrZzImKySeqUmiuigInDOtI0s3SteR6mTWoC30vg6BSyjOQjzq2Vp5ZCbSbxx
ml0vvL1mHRh3Ecv4i3BBVr7V49lJgYBza2M6YTdUH44ri29j7uCDnxiMlbJAJGOi
43ozs5dqLywk9ehGBuxHSXTNFMUhkkMPeoCLKGF0A/W44eeQWh+OYkKfMjvEscGy
weG2mtYGqFVksEs4fsxD2Q==
-----END CERTIFICATE-----
~~~

Next lets take the username and encrpyted password variables we set earlier and base64 encoded them into the variabled BASE64AUTH:

~~~bash
$ BASE64AUTH=`echo $QUAYUSERNAME:$QUAYPASSWORD | base64`
~~~

With the BASE64AUTH variable set lets go ahead and create a pull secret snippit of quay-secret.json:

~~~bash
$ cat << EOF > ~/quay-secret.json 
    "poc-registry-quay-quay-poc.apps.kni20.schmaustech.com": {
      "auth": "$BASE64AUTH",
      "email": "openshift@schmaustech.com"
    }
EOF
~~~

Now take the quay-secret.json and merge it with an existing pull-secret.json that was retrieved from https://console.redhat.com:

~~~bash
$ cat ~/pull-secret.json | jq ".auths += {`cat ~/quay-secret.json`}" > ~/quay-merged-pull-secret.json
~~~

Now we have a quay-merged-pull-secret.json file which contains the credentials to pull images from Quay.io for OpenShift and also have credentials to push/pull those images into our local Red Hat Quay openshift4 repository.  Before we can actually execute the mirroring though we need to set a few more variables to tell the mirror command what release we are mirroring, the local registry and repository name and where to retrieve those images from along with our quay-merged-pull-secret.json:

~~~bash
$ OCP_RELEASE=4.10.10
$ LOCAL_REGISTRY='poc-registry-quay-quay-poc.apps.kni20.schmaustech.com'
$ LOCAL_REPOSITORY='openshift4/openshift4'
$ PRODUCT_REPO='openshift-release-dev'
$ LOCAL_SECRET_JSON='quay-merged-pull-secret.json' 
$ RELEASE_NAME="ocp-release"
$ ARCHITECTURE='x86_64'
~~~

At this point we are ready to mirror the OpenShift images to our local Red Hat Quay registry.  We can do this with the oc adm release mirror command and by adding the --certificate-authority flag on the end because we are using a self signed cert.  However due to an active [BZ:2033212 ](https://bugzilla.redhat.com/show_bug.cgi?id=2033212) this method will not work yet.

~~~bash
oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --certificate-authority=./quay-ca/ca.crt
~~~

The alternate and again this is only being used because of the self signed cert involved here we can replace the --certificate-authorirty with --insecure=true to enable mirroring:

~~~bash
oc adm release mirror -a ${LOCAL_SECRET_JSON} --from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} --to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} --to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --insecure=true
info: Mirroring 162 images to poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4 ...
poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/
  openshift4/openshift4
    blobs:
      quay.io/openshift-release-dev/ocp-release sha256:39382676eb30fabb7a0616b064e142f6ef58d45216a9124e9358d14b12dedd65 1.428KiB
      quay.io/openshift-release-dev/ocp-release sha256:3373f848a7cb7ca66de1d9190b99e63acfe0644ed837da756ca1168365013312 1.73KiB
      quay.io/openshift-release-dev/ocp-release sha256:4dee79d6f4cb4d991a8c44cbc5f7ce90bc70672c8ab1f901af6ace6ea4c39244 598.3KiB

 (...)
      sha256:fdb8e07238367f8e086049f0255e5f7d0edfcfe2f28bc72bfbb3cfa02d176f2c -> 4.10.10-x86_64-operator-marketplace
      sha256:fee261289761341aaa662801cff6689eb7e47ad1b492ca6288af18a964ab2406 -> 4.10.10-x86_64-vsphere-csi-driver-operator
      sha256:ff507b711bc99f95cdb847a4ec936ea44aa4abf738d2e8dc1b0f4a443731cdf3 -> 4.10.10-x86_64-powervs-machine-controllers
  stats: shared=5 unique=327 size=11.73GiB ratio=0.99

phase 0:
  poc-registry-quay-quay-poc.apps.kni20.schmaustech.com openshift4/openshift4 blobs=332 mounts=0 manifests=162 shared=5

info: Planning completed in 31.52s
uploading: poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4 sha256:66ab094148e45140732dcaabf86556d54f557433f1c04f2023c07361c8d1c374 10.64MiB
uploading: poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4 sha256:4dee79d6f4cb4d991a8c44cbc5f7ce90bc70672c8ab1f901af6ace6ea4c39244 598.3KiB
uploading: poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4 sha256:237bfbffb5f297018ef21e92b8fede75d3ca63e2154236331ef2b2a9dd818a02 79.51MiB
(...)
sha256:054fbaac8295dada04effbbef8d8793dc0e52f71183b5d76a7396ff362180f41 poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4:4.10.10-x86_64-ironic
sha256:3888e5e253b8caf99bc6517f98f4de6f2b39548c1bcf6b3a92b0ad0ac0ba72ed poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4:4.10.10-x86_64-machine-os-content
sha256:e5b50f23be5b64f771e3e1cc965dccb06d097394cbf8d75cb59e37c91e332eb6 poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4:4.10.10-x86_64-multus-networkpolicy
info: Mirroring completed in 18m11.38s (11.54MB/s)

Success
Update image:  poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4:4.10.10-x86_64
Mirror prefix: poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4
Mirror prefix: poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4:4.10.10-x86_64

To use the mirrored repository to install, add the following section to the install-config.yaml:

imageContentSources:
- mirrors:
  - poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev


To use the new mirrored repository for upgrades, use the following to create an ImageContentSourcePolicy:

apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: example
spec:
  repositoryDigestMirrors:
  - mirrors:
    - poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4
    source: quay.io/openshift-release-dev/ocp-release
  - mirrors:
    - poc-registry-quay-quay-poc.apps.kni20.schmaustech.com/openshift4/openshift4
    source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
~~~

When the mirroring is completed save the output for the ImageContentSourcePolicy as that will be used when depoying any clusters and point them to the Red Hat Quay registry for the OpenShift 4.10.10 images.

Hopefully this blog provided some ideas on what is possible when deploying Red Hat Quay from a command line scenario and while this blog does not encapsulate all customer use cases and scenarios it does provide a good base to build upon for those advanced scenarios.
