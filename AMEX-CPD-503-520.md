**AMEX CPD upgrade 5.0.3 to 5.2.0**

Author: Alex Kuan (alex.kuan@ibm.com)

**From:**

CPD: 5.0.3

OCP: 4.17.31

Storage: FDF 2.9.1

Internet: proxy

Private container registry: yes

Components:
ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,db2oltp,datagate

**To:**

CPD: 5.2.0

OCP: 4.17.31

Storage: FDF 2.9.1

Internet: proxy

Private container registry: yes

Components:
ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,db2oltp,datagate

**Upgrade flow and steps**

1\. CPD 5.0.3 precheck

2\. Update cpd-cli and env variables script for 5.2.0

3\. Backup CPD 5.0.3 CRs, cpd-instance, and cpd-operators namespaces

4\. Upgrade shared cluster components
(ibm-cert-manager,ibm-licensing,scheduler)

5\. Prepare CPD upgrade (entitlement)

6\. Upgrade foundational service (cpfs)

7\. Upgrade CPD platform (cpd_platform)

8\. Upgrade CPD services (db2oltp,datagate)

9\. Potential Issues

10\. Validate CPD upgrade (customer acceptance test)

**\
**

**1. CPD 5.0.3 precheck**

1.  A client workstation RHEL 8 with internet to download OCP and CPD
    images, it can be bastion or infra node

OS level and make sure OS is RHEL 8

cat /etc/redhat-release

Test internet connection, and make sure the output from the target url
and it can be connected successfully.

curl -v https://github.com/IBM

Prepare customer\'s IBM entitlement key

1.  Log in to <https://myibm.ibm.com/products-services/containerlibrary>
    on My IBM with the IBMid and password that are associated with the
    entitled software.

2.  On the Get entitlement key tab, select Copy key to copy the
    entitlement key to the clipboard.

3.  Save the API key in a text file.

Make sure free disk space more than 500 GB (to download images and pack
the images into a tar ball)

df -lh

2.  Collect OCP and CPD cluster information

Log into OCP cluster from bastion node

oc login \$(oc whoami \--show-server) -u kubeadmin -p
\<kubeadmin-password\>

Check out OCP version

oc get clusterversion

Check out storage classes

oc get sc

Check out OCP cluster status

Make sure all nodes in ready status

oc get nodes

Make sure all mc in correct status, UPDATED all True, UPDATING all
False, DEGRADED all False

oc get mcp

Make sure all co in correct status, AVAILABLE all True, PROGRESSING all
False, DEGRADED all False

oc get co

Make sure the unhealthy pods have no pod unhealthy, if there is, please
open IBM support case to fix them.

oc get po -A -owide \| egrep -v \'(\[0-9\])/\\1\' \| egrep -v
\'Completed\'

Get CPD installed projects

oc get pod -A \| grep zen

Get CPD version and installed components

cpd-cli manage login-to-ocp \\

\--username=\${OCP_USERNAME} \\

\--password=\${OCP_PASSWORD} \\

\--server=\${OCP_URL}

or

cpd-cli manage login-to-ocp \\

\--server=\${OCP_URL} \\

\--token=\${OCP_TOKEN}

cpd-cli manage get-cr-status \\

\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS}

or

cpd-cli manage list-deployed-components
\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS}

Check the scheduling service, if it is installed but not in
ibm-common-services project, uninstall it

oc get scheduling -A

Check install plan is automatic

oc get ip -A

Check all CPD services and cr status

oc get crd \| grep cpd.ibm.com \| awk \'{print(\"\\noc get \"\$1\"
-A\"); system(\"oc get \"\$1\" -A\")}\'

Check spec of each CPD customer resource, remove patch before upgrade

oc project \${PROJECT_CPD_INST_OPERANDS}

for i in \$(oc api-resources \| grep cpd.ibm.com \| awk \'{print
\$1}\'); do echo \"\*\*\*\*\*\*\*\*\*\*\*\*\* \$i
\*\*\*\*\*\*\*\*\*\*\*\*\*\"; for x in \$(oc get \$i \--no-headers \|
awk \'{print \$1}\'); do echo \"\-\-\-\-\-\-\-\-- \$x
\-\-\-\-\-\-\-\-\-\-\--\"; oc get \$i \$x -o jsonpath={.spec} \| jq;
done ; done

Check IBM Cloud Pak foundational services (cpfs)

----

Probe IBM registry

podman login cp.icr.io -u cp -p \${IBM_ENTITLEMENT_KEY}

**\
**

**2. Update cpd-cli and env variables script for 5.2.0**

Download cpd-cli

wget
https://github.com/IBM/cpd-cli/releases/download/v14.2.0/cpd-cli-s390x-EE-14.2.0.tgz
&& tar xvf cpd-cli-s390x-EE-14.2.0.tgz && rm -rf
cpd-cli-s390x-EE-14.2.0-2081.tar

Launch olm-utils-play-v3 container

export PATH=/root/cpd-cli-s390x-EE-14.2.0-2081:\$PATH

cpd-cli manage restart-container

Update your environment variables script

export VERSION=5.2.0

source cpd_vars.sh

**\
**

**3. Backup CPD 5.0.3 CRs, cpd-instance, and cpd-operators namespaces**

Create a new directory and store the output of the following commands in
that directory:

oc project \${PROJECT_CPD_INST_OPERANDS}

for i in \$(oc api-resources \| grep cpd.ibm.com \| awk \'{print
\$1}\'); do echo \"\*\*\*\*\*\*\*\*\*\*\*\*\* \$i
\*\*\*\*\*\*\*\*\*\*\*\*\*\"; for x in \$(oc get \$i \--no-headers \|
awk \'{print \$1}\'); do echo \"\-\-\-\-\-\-\-\-- \$x
\-\-\-\-\-\-\-\-\-\-\--\"; oc get \$i \$x -oyaml \> bak-\$x.yaml; done ;
done

oc adm inspect -n \${PROJECT_CPD_INST_OPERANDS}
\--dest-dir=source-\${PROJECT_CPD_INST_OPERANDS} \$(oc api-resources
\--namespaced=true \--verbs=get,list \--no-headers -o name \| tr \'\\n\'
\',\' \| sed \'s/,\$//\')

oc adm inspect -n \${PROJECT_CPD_INST_OPERATORS}
\--dest-dir=source-\${PROJECT_CPD_INST_OPERATORS} \$(oc api-resources
\--namespaced=true \--verbs=get,list \--no-headers -o name \| tr \'\\n\'
\',\' \| sed \'s/,\$//\')

**\
**

**4. Upgrade shared cluster components
(ibm-cert-manager,ibm-licensing,scheduler)**

Find out which project the License Service is in

oc get deployment -A \| grep ibm-licensing-operator

Upgrade the Certificate manager and License Service\
\
The License Service is in the \${PROJECT_LICENSE_SERVICE} project (est.
5 minutes)

\${CPDM_OC_LOGIN}

cpd-cli manage apply-cluster-components \\

\--release=\${VERSION} \\

\--license_acceptance=true \\

\--cert_manager_ns=\${PROJECT_CERT_MANAGER} \\

\--licensing_ns=\${PROJECT_LICENSE_SERVICE}

Wait for the cpd-cli to return the following message before proceeding
to the next step:

\[SUCCESS\] \... The apply-cluster-components command ran successfully.

Confirm that the Certificate manager pods in the
\${PROJECT_CERT_MANAGER} project are Running or Completed:

oc get pods \--namespace=\${PROJECT_CERT_MANAGER}

Confirm that the License Service pods are Running or Completed

oc get pods \--namespace=\${PROJECT_LICENSE_SERVICE}

**\
**

**5. Prepare CPD upgrade (entitlements)**

Apply entitlement (est. 1 minute)

cpd-cli manage apply-entitlement \\

\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS} \\

\--entitlement=cpd-enterprise

**\
**

**6. Upgrade foundational service and cpd platform**

Before you upgrade IBM Software Hub, check for the whether the following
pods are running in this instance of IBM Software Hub.

Check whether the global search pods are running:

oc get pods \--namespace=\${PROJECT_CPD_INST_OPERANDS} \| grep
elasticsea-0ac3

-   If the command returns an empty response, proceed to the next step.

-   If the command returns a list of pods, review [Upgrades fail when
    global search is configured
    incorrectly](https://www.ibm.com/docs/en/SSNFH6_5.2.x/hub/troubleshoot/ts-upgrade-global-search.html) to
    determine whether you have any configurations that could cause
    issues during upgrade.--

Check whether the catalog-api pods are running:

oc get pods \--namespace=\${PROJECT_CPD_INST_OPERANDS} \| grep
catalog-api

-   If the command returns an empty response, you are ready to
    upgrade IBM Software Hub.

-   If the command returns a list of pods, review the following guidance
    to determine how long the catalog-api service will be down during
    upgrade.

When you upgrade the common core services to IBM Software
Hub Version 5.2, the underlying storage for the catalog-api service is
migrated to PostgreSQL.

To determine how many databases will be migrated, get the URL of the web
client and set the INSTANCE_URL variable to this value:

cpd-cli manage get-cpd-instance-details \\

\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS}

export INSTANCE_URL=\<URL\>

Get the credentials for the wdp-service:

TOKEN=\$(oc get -n \${PROJECT_CPD_INST_OPERANDS} secrets wdp-service-id
-o yaml \| grep service-id-credentials \| cut -d\':\' -f2- \| sed -e
\'s/ //g\' \| base64 -d)

Get the number of catalogs in the instance:

curl -sk -X GET
\"https://\${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=catalogs&bss_account_id=999\"
-H \'accept: application/json\' -H \"Authorization: Basic \${TOKEN}\" \|
jq -r \'.catalogs \| length\'

Get the number of projects in the instance:

curl -sk -X GET
\"https://\${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=projects&bss_account_id=999\"
-H \'accept: application/json\' -H \"Authorization: Basic \${TOKEN}\" \|
jq -r \'.catalogs \| length\'

Get the number of spaces in the instance:

curl -sk -X GET
\"https://\${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=spaces&bss_account_id=999\"
-H \'accept: application/json\' -H \"Authorization: Basic \${TOKEN}\" \|
jq -r \'.catalogs \| length\'

Add up the number of catalogs, projects, and spaces returned by the
previous commands. Then, use the following table to determine
approximately how long the service will be offline during the migration:

Databases Downtime

Up to 1,000 databases 6 minutes

1,001 - 10,000 databases 20 minutes

10,001 - 70,000 databases 60 minutes

Next, save the
[precheck_migration.sh](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=uish-upgrading-software-hub-1)
script to your workstation and run the script to determine whether you
can run an *automatic* migration of the common core services or whether
you need to configure common core services to run
a *semi-automatic* migration:

./precheck_migration.sh

Take the appropriate action based on the message returned by the script.

Now you are ready to upgrade the required operators and custom resources
for the instance (est. 60 minutes)

\${CPDM_OC_LOGIN}

cpd-cli manage get-license \\

\--release=\${VERSION} \\

\--license-type=EE

cpd-cli manage setup-instance \\

\--release=\${VERSION} \\

\--license_acceptance=true \\

\--cpd_operator_ns=\${PROJECT_CPD_INST_OPERATORS} \\

\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS}

Optionally append (can cause imagepullbackoff errors, if missing
images):

\--run_storage_tests=true

Monitor the upgrade progress of the custom resources with the following
commands:\
oc get ZenService lite-cr -n cpd-instance -o yaml\
oc get ibmcpd ibmcpd-cr -n cpd-instance -o yaml**\
**

**7. Upgrade Operators and Custom Resources**

Log the cpd-cli in to the Red Hat® OpenShift® Container
Platform cluster\
\
\${CPDM_OC_LOGIN}

Review the license terms for the software that is installed on this
instance of IBM Software Hub\
\
cpd-cli manage get-license \\

\--release=\${VERSION} \\

\--license-type=EE\
\
Upgrade the operators in the operators project (est. 15 minutes)

cpd-cli manage apply-olm \\

\--release=\${VERSION} \\

\--cpd_operator_ns=\${PROJECT_CPD_INST_OPERATORS} \\

\--upgrade=true

Wait for the cpd-cli to return the following message before proceeding
to the next step:

\[SUCCESS\]\... The apply-olm command ran successfully.

Confirm that the operator pods are Running or Completed:

oc get pods \--namespace=\${PROJECT_CPD_INST_OPERATORS}

**8. Upgrade CPD services (db2oltp, datagate)**

Stop Data Gate synchronization in the UI.

Upgrade Db2 (estimated time to run 10 minutes)

cpd-cli manage apply-cr \\

\--components=db2oltp \\

\--release=\${VERSION} \\

\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS} \\

\--license_acceptance=true \\

\--upgrade=true

Monitor the status of the db2oltp upgrade with the following command:

oc get db2oltpservice.databases.cpd.ibm.com -n cpd-instance -o yaml

Db2 is upgraded when the apply-cr command returns:

\[SUCCESS\]\... The apply-cr command ran successfully

If you want to confirm that the custom resource status is Completed, you
can run the cpd-cli manage get-cr-status command:

cpd-cli manage get-cr-status \\

\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS} \\

\--components=db2oltp

Upgrade the Db2 service instance(s) next, and if required, ensure your
Db2 license is upgraded and CPD profile are set up (est. 10 minutes):

-   [Upgrading the license before you
    deploy Db2](https://www.ibm.com/docs/en/SSNFH6_5.2.x/svc-db2/db2-update-lic.html) 

-   [Creating a profile to use the cpd-cli management
    commands](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cli-creating-cpd-profile)

Remove custom patches and override scripts, if applicable (DB2 pod will
restart)

oc login OpenShift_URL:port

oc set volume statefulset/c-\${DB2U_ID}-db2u -n
\${PROJECT_CPD_INST_OPERANDS} \--remove \--name=db2u-entrypoint-sh

\*\*\*Potential issue during the upgrade of Db2 service instance with
db2ckupgrade.sh utility\*\*\*

Db2ckupgrade.sh utility runs a job that must be scheduled to the same
node as the Db2 engine pod. To work around this issue, restrict the job
to run on the same node as the engine pod (using a cordon).

\*\*\*Potential issue during the upgrade of Db2 service instance related
to tempspace1\*\*\*

During the Db2 instance upgrade you may observe issues related to
tempspace1, "Table space access is not allowed." This problem is related
to the usage of local storage and results in missing files and missing
folder structures within the Db2u pod. To address this, you will need to
create a specific directory structure inside the Db2u pod and copy the
container tag to this location. RSH into the Db2u pod and run the
following commands to create the folder structure and copy the container
tag to the same location:

mkdir -p
/mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/NODE0000/BLUDB/T0000001/C0000000.TMP

cp SQLTAG.NAM
/mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/NODE0000/BLUDB/T0000001/C0000000.TMP

chmod 600
/mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/NODE0000/BLUDB/T0000001/C0000000.TMP/SQLTAG.NAM

chown db2inst1:db2iadm1
/mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1/NODE0000/BLUDB/T0000001/C0000000.TMP/SQLTAG.NAM

Please keep in mind that the container name will change and that these
steps assume that the file paths are identical across both systems, IPC1
and IPC2.

After these steps, confirm that Db2 can start and then once you can
connect to Db2, use the below commands for the automation to pick up the
fix.

Run the below commands as Db2 instance owner to take the backup of the
container tags, as this backup will be used to restore the container
tags when the pod starts again.

sudo rsync -rdgop \--numeric-ids \--checksum \--exclude \'\*TLB\'
\--exclude \'\*TDA\' \--exclude \'\*TBA\'
/mnt/tempts/c-db2oltp-1712862624337428-db2u/ /mnt/blumeta0/local-backup

sudo rsync -rdgop \--numeric-ids \--checksum \--exclude \'\*TLB\'
\--exclude \'\*TDA\' \--exclude \'\*TBA\'
/mnt/tempts/c-db2oltp-1712862624337428-db2u/db2inst1
/mnt/blumeta0/local-backup

Prepare for the Db2 instance upgrade by obtaining a list of Db2 service
instances:

To obtain value for --profile run cat \$HOME/.cpd-cli/config as root.

cpd-cli service-instance list \\

\--service-type=db2oltp \\

\--profile=\${CPD_PROFILE_NAME}

Set the INSTANCE_NAME to the name of the service instance:

export INSTANCE_NAME=\<instance-name\>

Check whether your Db2 service instances are in running state:

cpd-cli service-instance status \${INSTANCE_NAME} \\

\--profile=\${CPD_PROFILE_NAME} \\

\--service-type=db2oltp

Upgrade the db2oltp service instance (est. 10 minutes)\
\
cpd-cli service-instance upgrade \\\
\--service-type=db2oltp \\\
\--instance-name=\${INSTANCE_NAME} \\\
\--profile=\${CPD_PROFILE_NAME}

Verifying the service instance upgrade:

Run the following command and wait for the status to change to Ready:

oc get db2ucluster \<instance_id\> -o jsonpath=\'{.status.state}
{\"\\n\"}\'

Run the following command to check the status of your Db2 service
instances:

cpd-cli service-instance status \${INSTANCE_NAME} \\

\--profile=\${CPD_PROFILE_NAME} \\

\--service-type=db2oltp

Run the following command to check that the service instances have
updated:

cpd-cli service-instance list \\

\--profile=\${CPD_PROFILE_NAME} \\

\--service-type=db2oltp

After upgrading Db2, you need to update the configmap. Run the following
commands to patch the instance.json configmap. Run the following command
to retrieve the values for the variables, CM_NAME, NAMESPACE and
NEW_VERSION:

cpd-cli service-instance list \--profile=\${CPD_PROFILE_NAME}
\--service-type=db2oltp

CM_NAME: Db2u instance configmap name. CM_NAME is obtained like
this: \<service_type\>-\<instance_id\>-\<service_type\>-cm. For
example, db2oltp-1689782702423826-db2oltp-cm.

NAMESPACE: Cluster namespace where Db2 instance and the configmap are
installed. For example, cpd-instance.

NEW_VERSION: Upgraded Db2 instance version. For
example, 11.5.9.0-cn5-amd64.

[Copy the bash
script](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=u-upgrading-from-version-51-36)
to your local workstation and update the permissions of the bash script
using chmod.

Update the values for the CM_NAME, NAMESPACE, and NEW_VERSION variables
in the bash script and run the bash script.

Next up, continue with the upgrade of Data Gate custom resource.

Update the custom resource for Data Gate (est. 10 minutes):

cpd-cli manage apply-cr \\

\--components=datagate \\

\--release=\${VERSION} \\

\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS} \\

\--block_storage_class=\${STG_CLASS_BLOCK} \\

\--file_storage_class=\${STG_CLASS_FILE} \\

\--license_acceptance=true \\

\--upgrade=true

Validating the upgrade: Data Gate is upgraded when the apply-cr command
returns:

\[SUCCESS\]\... The apply-cr command ran successfully

If you want to confirm that the custom resource status is Completed, you
can run the cpd-cli manage get-cr-status command:

cpd-cli manage get-cr-status \\

\--cpd_instance_ns=\${PROJECT_CPD_INST_OPERANDS} \\

\--components=datagate

\*\*\*Potential issue during the upgrade of Data Gate service
instances\*\*\*

During Data Gate instance upgrade a dg-1750101262664465-backup-head-job
pod runs, but it should be scheduled to any node where Db2 is not
running -\> For this cordon the node hosting Db2, and after the
backup-head-job pod is completed, uncordon the same node

Next, upgrade the Data Gate service instances, ensure the CPD profile is
set up (est. 18 minutes):

-   [Creating a profile to use the cpd-cli management
    commands](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cli-creating-cpd-profile)

cpd-cli service-instance upgrade \\

\--all \\

\--service-type=dg \\

\--profile=\${CPD_PROFILE_NAME} \--watch

Verifying the service instance upgrade:

Run the following command and wait for the status to change to
Completed:

oc get dginstance \<instance_id\> -o
jsonpath=\'{.status.datagateInstanceStatus} {\"\\n\"}\'

Run the following command and see if the Provision status has changed to
UPGRADED:

watch cpd-cli service-instance list \--profile=\${CPD_PROFILE_NAME}
\--service-type dg

\*\*\*Potential issue after the upgrade of Data Gate service
instances\*\*\*

After Data Gate instances have been upgraded, it is possible that the
configuration settings of the Db2 target database are not correctly
migrated during the upgrade.

Missing Db2 configuration settings can cause a variety of issues, as
experienced most recently during the E3-IPC1 upgrade, in which the
DB2COMM=TCPIP,SSL was missing.

To resolve this, we had to add the missing Db2 setting by using 'db2set
DB2COMM=TCPIP,SSL'.

These configurations should persist through a Db2u pod recycle.

Keep track of your Db2 configuration settings prior to upgrading to
ensure you have a list of settings which you can restore.

You can also [change configuration
settings](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=configuration-changing-db2-settings)
after you deploy your instance.**\
**

**9. All Potential Issues**

1\. Setup-instance command will fail with 'imagepullbackoff' errors if
the storage test images are missing -\> Mirror the storage test images
ahead of time or exclude the '--run_storage_tests' flag

2\. For imagepullbackoff errors, ensure that all required images are
mirrored to the private registry, for example -\> skopeo copy
docker://icr.io/cpopen/ibm-operator-catalog@sha256:01712920b400fba751f60c71c27fbd64ca1b59b6cd325c42ae691f0b8770133d
docker://lpdza532.phx.aexp.com:5000/cpopen/ibm-operator-catalog@sha256:01712920b400fba751f60c71c27fbd64ca1b59b6cd325c42ae691f0b8770133d
\--all \--remove-signatures
\--authfile=/var/registry/oc4.7/installer/pullsecret/merged_pullsecret.json

3\. For insufficient CPU, insufficient memory, untolerated taint issues,
ensure there are enough resources available on the cluster -\> Perform a
pre-upgrade health check to ensure the cluster is in a healthy state
prior to the upgrade

4\. If the Db2 license is not upgraded to the compatible version you may
run into issues during Db2 instance upgrade. Follow the steps to
"[Upgrade the license before you deploy
Db2](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=setup-upgrading-license-before-you-deploy-db2)"

5\. During the Db2 instance upgrade, the Db2ckupgrade.sh utility runs a
job that must be scheduled to the same node as the Db2 engine pod. To
resolve this issue, restrict the job to run on the same node as the
engine pod. Use a cordon, to ensure that this job is scheduled to the
node hosting Db2. In previous upgrades, we've cordoned nodes lpqzo504,
lpqzo505, and lpqzo506, such that the job would be scheduled to lpqzo507

6\. During the Db2 instance upgrade you may observe issues related to
tempspace1, "Table space access is not allowed." This problem is related
to the usage of local storage and results in missing files and missing
folder structures within the Db2 pod. To address this, you will need to
create a specific directory structure inside the Db2u pod and copy the
container tag to this location.

7\. During Data Gate instance upgrade a
dg-1750101262664465-backup-head-job pod runs, but it should be scheduled
to any node where Db2 is not running -\> For this cordon the node
hosting Db2, and after the backup-head-job pod is completed, uncordon
the same node

8\. After Data Gate instances have been upgraded, it is possible that
the configuration settings of the Db2 target database are not correctly
migrated during the upgrade. Missing Db2 configuration settings can
cause a variety of issues, as experienced most recently during the
E3-IPC1 upgrade, in which the DB2COMM=TCPIP,SSL was missing. To resolve
this, we had to add the missing Db2 setting by using 'db2set
DB2COMM=TCPIP,SSL'. These configurations should persist through a Db2u
pod recycle. Keep track of your Db2 configuration settings prior to
upgrading to ensure you have a list of settings which you can restore.
You can also [change configuration
settings](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=configuration-changing-db2-settings)
after you deploy your instance

**10. Apply the Day 0 patch (if required)**

After you install or upgrade to IBM Software Hub Version 5.2.0, you must
apply the [Version 5.2.0 - Day 0
patch](https://www.ibm.com/support/pages/node/7236425) if you have any
services with a dependency on the common core services.

Download the script,
[5.2.0-day0-patch-v5.sh](https://www.ibm.com/support/pages/system/files/inline-files/5.2.0-day0-patch-v5.sh),
to your client workstation. Make the script executable:

chmod +x ./5.2.0-day0-patch-v5.sh

If your cluster pulls images from a private container registry, copy the
images for the Version 5.2.0 - Day 0 patch from the IBM Entitled
Registry to your private container registry:

nohup ./5.2.0-day0-patch-v5.sh \\

\--load \${PRIVATE_REGISTRY_LOCATION} \\

\--as \${PRIVATE_REGISTRY_PUSH_USER} \\

\--with \${PRIVATE_REGISTRY_PUSH_PASSWORD} \\

\--entitlement \${IBM_ENTITLEMENT_KEY} \\

\--operator \${PROJECT_CPD_INST_OPERATORS} \\

\--yes \> load_patch_images_output.log &

Apply the patch by running the following command:

nohup ./5.2.0-day0-patch-v5.sh \\

\--operator \${PROJECT_CPD_INST_OPERATORS} \\

\--yes \> patch_install_output.log &

In a duplicate terminal, monitor patch progress by watching the
patch_install_output.log file:

tail -f patch_install_output.log

The patch is successfully applied when you see the following log
message:

PASS: Apply Day 0 Patch has been successfully completed

This marks the end of the installation of IBM Software Hub, db2oltp and
Data Gate

**11. Validate CPD upgrade (customer acceptance test)**

End of document
