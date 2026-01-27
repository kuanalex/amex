## AMEX CPD upgrade 5.2.0 to 5.2.2
## Author: Alex Kuan (alex.kuan@ibm.com)

From:
```
CPD: 5.2.0
OCP: 4.17.31
Storage: FDF 2.9.1
Internet: proxy
Private container registry: yes
Components: ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,db2oltp,datagate
```

To:
```
CPD: 5.2.2
OCP: 4.17.31
Storage: FDF 2.9.1
Internet: proxy
Private container registry: yes
Components: ibm-cert-manager,ibm-licensing,cpfs,cpd_platform,db2oltp,datagate
```

Upgrade flow and steps
```
1. CPD 5.2.0 pre-check
2. Prepare to run upgrades in a restricted network using a private container registry
3. Backup CPD 5.2.0 CRs, cpd-instance, and cpd-operators namespaces
4. Upgrade shared cluster components (ibm-cert-manager,ibm-licensing,scheduler)
5. Prepare to upgrade an instance of IBM Software Hub
6. Upgrade an instance of IBM Software Hub
7. Upgrade CPD services (db2oltp,datagate)
8. Potential Issues To Be Aware Of
9. Validate CPD upgrade (customer acceptance test)
```


## 1. CPD 5.2.0 pre-check

### Use a client workstation with internet (bastion or infra node) to download OCP and CPD images, and confirm the OS level, ensuring the OS is RHEL 8/9
```
cat /etc/redhat-release
```

Test internet connection, and make sure the output from the target URL and it can be connected successfully:
```
curl -v https://github.com/IBM
```


### Prepare your IBM entitlement key

Log in to <https://myibm.ibm.com/products-services/containerlibrary> with the IBMid and password that are associated with the entitled software.

On the Get entitlement key tab, select Copy key to copy the entitlement key to the clipboard.

Save the API key in a text file.


### Make sure free disk space more than 500 GB (to download images and pack the images into a tar ball)
```
df -lh
```


### Collect OCP and CPD cluster information

Log into OCP cluster from bastion node
```
oc login $(oc whoami --show-server) -u kubeadmin -p <kubeadmin-password>
```


### Review OCP version
```
oc get clusterversion
```


### Review storage classes
```
oc get sc
```


### Review OCP cluster status and make sure all nodes are in ready status
```
oc get nodes
```


### Make sure all mc are in correct status, UPDATED all True, UPDATING all False, DEGRADED all False
```
oc get mcp
```


### Make sure all co are in correct status, AVAILABLE all True, PROGRESSING all False, DEGRADED all False
```
oc get co
```


### Make sure there are no unhealthy pods, if there are, please open an IBM support case to fix them.
```
oc get po -A -owide | egrep -v '([0-9])/\1' | egrep -v 'Completed' 
```


### Get CPD installed projects
```
oc get pod -A | grep zen
```


### Get CPD version and installed components
```
cpd-cli manage login-to-ocp \
--username=${OCP_USERNAME} \
--password=${OCP_PASSWORD} \
--server=${OCP_URL}
```

```
cpd-cli manage get-cr-status --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

or
```
cpd-cli manage list-deployed-components --cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```


### Check the scheduling service, if it is installed but not in ibm-common-services project, uninstall it
```
oc get scheduling -A
```


### Check install plan is automatic
```
oc get ip -A
```


### Check the spec of each CPD custom resource, remove any patches before upgrading
```
oc project ${PROJECT_CPD_INST_OPERANDS}
```

```
for i in $(oc api-resources | grep cpd.ibm.com | awk '{print $1}'); do echo "************* $i *************"; for x in $(oc get $i --no-headers | awk '{print $1}'); do echo "--------- $x ------------"; oc get $i $x -o jsonpath={.spec} | jq; done ; done
```

```
export PATH=/root/cpd-cli-linux-SE-14.2.2-2727:$PATH
```

Update your environment variables script
```
vi cpd_vars.sh
```

Update the Version field and save your changes
```
VERSION=5.2.2
```

Source your environment variables
```
source cpd_vars.sh
```


### Save the existing configuration
```
oc get datagateinstanceservices dg1734441701891261 -o yaml > dg1734441701891261.yaml 
oc get db2uclusters db2oltp-1734440804666892 -o yaml > db2oltp-1734440804666892.yaml 
oc get cm -n cpd-instance c-db2oltp-1734440804666892-db2dbconfig -o yaml > c-db2oltp-1734440804666892-db2dbconfig.yaml 
oc get cm -n cpd-instance c-db2oltp-1734440804666892-db2dbmconfig -o yaml > c-db2oltp-1734440804666892-db2dbmconfig.yaml
```


### Obtaining the olm-utils-v3 image (skip, if not required)
```
cd /apps/cpdcli/cpd-cli-linux-SE-14.2.2-2727
```
```
cpd-cli manage restart-container
```
```
podman ps
```


### STOP DATA GATE SYNCHRONIZATION

In the CP4D UI, disable data gate synchroinzation using the sync toggle


## 3. Backup CPD 5.2.0 CRs, cpd-instance and cpd-operators namespaces

Create a new directory and store the output of the following commands in that directory
```
mkdir cpdbackup && cd cpdbackup && oc project ${PROJECT_CPD_INST_OPERANDS}
for i in $(oc api-resources | grep cpd.ibm.com | awk '{print $1}'); do echo "************* $i *************"; for x in $(oc get $i --no-headers | awk '{print $1}'); do echo "--------- $x ------------"; oc get $i $x -oyaml > bak-$x.yaml; done ; done
```

**Note: The following 'oc adm' commands can be time-consuming and should be collected during the pre-upgrade Health Check activity**

Backup the current state of operands in your backup folder of choice:
```
mkdir operandsbackup && cd operandsbackup && oc adm inspect -n ${PROJECT_CPD_INST_OPERANDS} --dest-dir=source-${PROJECT_CPD_INST_OPERANDS} $(oc api-resources --namespaced=true --verbs=get,list --no-headers -o name | tr '\n' ',' | sed 's/,$//')
```

Backup the current state of operators in your backup folder of choice:
```
mkdir operatorsbackup && cd operatorsbackup && oc adm inspect -n ${PROJECT_CPD_INST_OPERATORS} --dest-dir=source-${PROJECT_CPD_INST_OPERATORS} $(oc api-resources --namespaced=true --verbs=get,list --no-headers -o name | tr '\n' ',' | sed 's/,$//')
```


## 4. Upgrade shared cluster components (ibm-cert-manager,ibm-licensing)

Determine which project the License Service is in
```
oc get deployment -A | grep ibm-licensing-operator
```

Upgrade the Certificate manager and License Service

The License Service is in the ${PROJECT_LICENSE_SERVICE} or ${PROJECT_CS_CONTROL} project

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster
```
${CPDM_OC_LOGIN}
```

Upgrade shared cluster components
```
cpd-cli manage apply-cluster-components \
--release=${VERSION} \
--license_acceptance=true \
--cert_manager_ns=${PROJECT_CERT_MANAGER} \
--licensing_ns=${PROJECT_CS_CONTROL} \
--case_download=false
```

Wait for the cpd-cli to return the following message before proceeding to the next step:

[SUCCESS] ... The apply-cluster-components command ran successfully

Confirm that the Certificate manager pods in the ${PROJECT_CERT_MANAGER} project are Running or Completed:
```
oc get pods --namespace=${PROJECT_CERT_MANAGER}
```

Confirm that the License Service pods are Running or Completed
```
oc get pods --namespace=${PROJECT_CS_CONTROL}
```


## 5. Prepare to upgrade an instance of IBM Software Hub (est. 1-2 minutes)

### Validate the health of your cluster, nodes, operators, and operands before proceeding with the upgrade:
```
cpd-cli health operators \
--control_plane_ns=${PROJECT_CPD_INST_OPERANDS} \
--operator_ns=${PROJECT_CPD_INST_OPERATORS}
```

Results should read "SUCCESS..."
```
cpd-cli health operands \
--control_plane_ns=${PROJECT_CPD_INST_OPERANDS} \
--include_ns=${PROJECT_CPD_INST_OPERATORS}
```

Results should read "SUCCESS..."
```
cpd-cli health cluster
```

Results should read "SUCCESS..."
```
cpd-cli health nodes
```

Results should read "SUCCESS..."


### [Next, apply entitlements](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=puish-applying-your-entitlements-3) (est. 1-2 minutes):

Run the apply-entitlement command for each solution that is installed or that you plan to install in this instance

Apply entitlements for CP4D:
```
cpd-cli manage apply-entitlement \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--entitlement=cpd-enterprise
```


### Disable Bedrock customiztion (skip, if not required)
```
oc get sts -n cpd-instance |grep db2u
```

Observe the output
```
c-db2oltp-1736767772824325-db2u
```

Run the set volume command
```
oc set volume statefulset/c-db2oltp-1736767772824325-db2u -n cpd-instance --remove --name=db2u-entrypoint-sh
```

If you see this error message, you can proceed normally
```
error: statefulsets/c-db2oltp-1736767772824325-db2u volume 'db2u-entrypoint-sh' not found
```


## 6. Upgrade an instance of IBM Software Hub


### Before you upgrade to IBM Software Hub, check whether the following common core services pods are running in this instance of IBM Cloud Pak for Data:

Check whether the global search pods are running:
```
oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep elasticsea-0ac3
```

If the command returns an empty response, proceed to the next step.

If the command returns a list of pods, review [Upgrades fail when global search is configured incorrectly](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=tiu-upgrades-fail-when-global-search-is-configured-incorrectly) to determine whether you have any configurations that could cause issues during upgrade.

Check whether the catalog-api pods are running:
```
oc get pods --namespace=${PROJECT_CPD_INST_OPERANDS} | grep catalog-api
```

If the command returns an empty response, you are ready to upgrade IBM Software Hub.

If the command returns a list of pods, review the following guidance to determine how long the catalog-api service will be down during upgrade.

When you upgrade the common core services to IBM Software Hub Version 5.2, the underlying storage for the catalog-api service is migrated to PostgreSQL.

During the final stages of the migration, the catalog-api service is offline, and services that are dependent on the service are not available. The duration of the migration depends on the number of assets and relationships that are stored in the instance. The duration of the outage depends on the number of databases (projects, catalogs, and spaces) in the instance. In a typical upgrade scenario, the outage should be significantly shorter than the overall migration.

To determine how many databases will be migrated follow these steps:

Set the INSTANCE_URL environment variable to the URL of IBM Software Hub
```
export INSTANCE_URL=<URL>
```

To get the URL of the web client, run the following command:
```
cpd-cli manage get-cpd-instance-details \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS}
```

Get the credentials for the wdp-service:
```
TOKEN=$(oc get -n ${PROJECT_CPD_INST_OPERANDS} secrets wdp-service-id -o yaml | grep service-id-credentials | cut -d':' -f2- | sed -e 's/ //g' | base64 -d)
```

Get the number of catalogs in the instance:
```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=catalogs&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Get the number of projects in the instance:
```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=projects&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Get the number of spaces in the instance:
```
curl -sk -X GET "https://${INSTANCE_URL}/v2/catalogs?limit=10001&skip=0&include=spaces&bss_account_id=999" -H 'accept: application/json' -H "Authorization: Basic ${TOKEN}" | jq -r '.catalogs | length'
```

Add up the number of catalogs, projects, and spaces returned by the previous commands. Then, use the following table to determine approximately how long the service will be offline during the migration:
```
Databases                   Downtime for migration (approximate)
Up to 1,000 databases	    6 minutes
1,001 - 10,000 databases	20 minutes
10,001 - 70,000 databases	60 minutes
```

Save the following script on the client workstation as a file named precheck_migration.sh:
```
#!/bin/bash

# Default ranges for couchdb size
SMALL=30
LARGE=200

echo "Performing pre-migration checks"

check_resources(){
        scale_config=$1
        pvc_size=$(oc get pvc -n ${PROJECT_CPD_INST_OPERANDS} database-storage-wdp-couchdb-0 --no-headers | awk '{print $4}')
        size=$(awk '{print substr($0, 1, length($0)-2)}' <<< "$pvc_size")

        if [[ "$size" -le "$SMALL" ]];then
          echo "The system is ready for migration. Upgrade your cluster as usual."
        elif [ "$size" -ge "$SMALL" ] && [ "$size" -le "$LARGE" ] && [[ $scale_config == "small" ]];then
          echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 6,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "3", "ephemeral-storage": "10Mi", "memory": "4Gi"},
    "limits": {"cpu": "8", "ephemeral-storage": "4Gi", "memory": "8Gi"}}
}}'
EOF
          echo
          echo "The system is ready for migration. Upgrade your cluster as usual."
        elif [[ $scale_config == "medium" ]];then
          echo -e "Run the following command to increase the CPU and memory:\n"
          cat << EOF
oc patch ccs ccs-cr -n ${PROJECT_CPD_INST_OPERANDS} --type merge --patch '{"spec": {
  "catalog_api_postgres_migration_threads": 8,
  "catalog_api_migration_job_resources": { 
    "requests": {"cpu": "6", "ephemeral-storage": "10Mi", "memory": "6Gi"},
    "limits": {"cpu": "10", "ephemeral-storage": "6Gi", "memory": "10Gi"}}
}}'
EOF
          echo
          echo "Before you can start the upgrade, you must prepare the system for migration."
        fi
}

check_upgrade_case(){     
        scale_config=$(oc get ccs -n ${PROJECT_CPD_INST_OPERANDS} ccs-cr -o json | jq -r '.spec.scaleConfig')

        # Default case, scale config is set to small
        if [[ -z "${scale_config}" ]];then
          scale_config=small
        fi

        if [[ $scale_config == "large" ]];then
          echo "Before you can start the upgrade, you must prepare the system for migration."
        elif [[ $scale_config == "small" ]] || [[ $scale_config == "medium" ]];then
          check_resources $scale_config
        fi
}

check_upgrade_case
```

Run the precheck_migration.sh to determine whether you can run an automatic migration of the common core services or whether you need to configure common core services to run a semi-automatic migration:
```
./precheck_migration.sh
```

Take the appropriate action based on the message returned by the script:

Option 1 - "The system is ready for migration" -> Upgrade your cluster as usual

Option 2 - "Run the following command to increase the CPU and memory" -> Run the patch command returned by the script, upgrade IBM Software Hub

Option 3 - "The script returns both of the following messages: Run the following command to increase the CPU and memory AND Before you can start the upgrade, you must prepare the system for migration." -> Run the patch command returned by the script, then run the following command to enable semi-automatic migration:
```
oc patch ccs ccs-cr \
-n ${PROJECT_CPD_INST_OPERANDS} \
--type merge \
--patch '{"spec": {"use_semi_auto_catalog_api_migration": true}}'
```


### Proceed with the upgrade of IBM Software Hub

Upgrade the required operators and custom resources for the instance:

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster
```
${CPDM_OC_LOGIN}
```

Review the license terms for the software that you plan to install:
```
cpd-cli manage get-license \
--release=${VERSION} \
--license-type=EE
```


### Upgrade the required operators and custom resources for the instance (est. 60 minutes):
```
cpd-cli manage setup-instance \
--release=${VERSION} \
--license_acceptance=true \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--file_storage_class=${STG_CLASS_FILE} \
--run_storage_tests=false \
--case_download=false
```

Monitor the upgrade progress of the custom resources with the following commands:
```
watch -n 5 'oc get pods -A -o wide | grep -Ev "([0-9]+)/\1|Completed"; echo; oc get ZenService -n cpd-instance -o yaml | grep "progress"'
```

```
oc get ZenService lite-cr -n cpd-instance -o yaml
```

```
oc get ibmcpd ibmcpd-cr -n cpd-instance -o yaml
```


### Upgrade the operators in the operators project (est. 15 minutes):
```
cpd-cli manage apply-olm \
--release=${VERSION} \
--cpd_operator_ns=${PROJECT_CPD_INST_OPERATORS} \
--upgrade=true \
--case_download=false
```

Wait for the cpd-cli to return the following message before proceeding to the next step:

[SUCCESS]... The apply-olm command ran successfully

Confirm that the operator pods are Running or Completed:
```
oc get pods --namespace=${PROJECT_CPD_INST_OPERATORS}
```


## 7. Upgrade CPD services (db2oltp,datagate)

In order to have more visibility into each service upgrade, it is recommended to upgrade the services in sequential order, as follows:


### Upgrade Db2 custom resource (est. 15 minutes)

**IMPORTANT: BEFORE YOU PROCEED, MAKE SURE TO STOP DATA GATE SYNCHRONIZATION**

Before you upgrade the Db2 custom resource, check the status of the zen-metastore-edb pods
```
oc get pods -n ${PROJECT_CPD_INST_OPERANDS} -l k8s.enterprisedb.io/cluster=zen-metastore-edb
```

For example
```
NAME                  READY   STATUS    RESTARTS      AGE
zen-metastore-edb-1   1/1     Running   0             XdXh
zen-metastore-edb-2   1/1     Running   0             XdXh
```

If there are no issues with zen-metastore-edb pods, proceed with upgrading the Db2 custom resource

Log the cpd-cli in to the Red Hat® OpenShift® Container Platform cluster:
```
${CPDM_OC_LOGIN}
```

Update the custom resource for Db2:
```
cpd-cli manage apply-cr \
--components=db2oltp \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--license_acceptance=true \
--upgrade=true \
--case_download=false
```

If you want to confirm that the custom resource status is Completed, you can run the cpd-cli manage get-cr-status command:
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=db2oltp
```

You can also monitor the status of the Db2 upgrade with the following command:
```
watch -n 5 'oc get pods -A -o wide | grep -Ev "([0-9]+)/\1|Completed"; echo; oc get Db2oltpService -n cpd-instance -o yaml | grep "progress"'
```

```
oc get db2oltpservice.databases.cpd.ibm.com -n cpd-instance -o yaml
```

Db2 is upgraded when the apply-cr command returns:
```
[SUCCESS]... The apply-cr command ran successfully
```


### Upgrade Db2 instances (est. 10 minutes)

**Before you proceed, ensure you switch DB2 to Archive Logging mode, Db2 license is upgraded, and CPD profile are set up**

### [Switch DB2 to Archive Logging mode](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=gate-target-database-in-backup-pending-state)


Remote into the db2u engine pod with db2inst1 user
```
oc exec -it ${DB2U_POD_NAME} -n ${PROJECT_CPD_INST_OPERANDS} su - db2inst1
```

Set dbConfig LOGARCHMETH1 value to OFF
```
db2 update db cfg for bludb using LOGARCHMETH1 OFF
```


### [Upgrading the license before you deploy Db2](https://www.ibm.com/docs/en/SSNFH6_5.2.x/svc-db2/db2-update-lic.html)

Retrieve the license file: DB2 Standard Edition 12.1: db2std_vpc.lic

Encode the license
```
base64 -w0 db2std_vpc.lic
```

```
oc patch Db2oltpservice db2oltp-cr -n cpd-instance --type merge -p "{\"spec\":{\"license\":{\"license\":\"Standard\", \"licenseValue\":{\"value64\":\"
ZTNlNTQ1NTM5YTQ2NzdlMDM3ODU4Mzg4ODliYjk0ZDRlNDI1MmQyNjc4MDhlOGZmODNjZWJhNjQzMTM3MjdkMQ0KRGVyaXZlZExpY2Vuc2VBZ2dy
ZWdhdGVEdXJhdGlvbj0NCkRlcml2ZWRMaWNlbnNlRW5kRGF0ZT0NCkRlcml2ZWRMaWNlbnNlU3RhcnREYXRlPQ0KTGljZW5zZUR1cmF0aW9uPTY3
MTYNCkxpY2Vuc2VFbmREYXRlPTEyLzMxLzIwMzcNCkxpY2Vuc2VTdGFydERhdGU9MDgvMTMvMjAxOQ0KUHJvZHVjdEFubm90YXRpb249MTI3IDE0
MyAyNTUgMjU1IDk0IDI1NSAxIDAgMCAwLTI3OyMwIDEyOCAxNiAwIDANClByb2R1Y3RJRD0xNDA1DQpQcm9kdWN0TmFtZT1EQjIgU3RhbmRhcmQg
RWRpdGlvbg0KUHJvZHVjdFZlcnNpb249MTIuMQ0K\"}}}}"
```

```
oc get Db2oltpService db2oltp-cr -n cpd-instance -o jsonpath='{.status.db2oltpStatus} {"\n"}'
```


### [Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=cli-creating-cpd-profile)

Connect to the Cloud Pak user interface and retrieve an API key

Click on Profile and settings

Click on Generate new key

Prepare the environment variables
```
export API_KEY="2wpXlj1rm69uZTjnvlQOqVFOJloQmK0D2PuoPHkV" 
export CPD_USERNAME="l318711" 
export LOCAL_USER="user_devme" 
export CPD_PROFILE_NAME="prof_devme" 
export CPD_PROFILE_URL="https://cpd-cpd-instance.apps.oc001b000004.dev.echonet"
```

Configure the user
```
cd /apps/cpdcli/cpd-cli-linux-SE-14.2.2-2727
cpd-cli config users set ${LOCAL_USER} --username ${CPD_USERNAME} --apikey ${API_KEY}
```

Configure the profile
```
cpd-cli config profiles set ${CPD_PROFILE_NAME} --user ${LOCAL_USER} --url ${CPD_PROFILE_URL}
```

Confirm the profile is working
```
cpd-cli service-instance list --profile=${CPD_PROFILE_NAME}
```

Set the INSTANCE_NAME to the name of the service instance ahead of db2oltp service instance upgrade:
```
export INSTANCE_NAME=<instance-name>
```


### Upgrade the db2oltp service instance (est. 10 minutes)

**IMPORTANT: Potential issue during the upgrade of Db2 service instance with db2ckupgrade.sh utility**

Db2ckupgrade.sh utility runs a job that must be scheduled to the same node as the Db2 engine pod

To work around this issue, restrict the job to run on the same node as the engine pod (using a cordon)

Initiate the upgrade of db2oltp services instances
```
cpd-cli service-instance upgrade \
--service-type=db2oltp \
--instance-name=${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME} \
--watch
```

Verify the service instance upgrade by running the following command and waiting for the status to change to Ready:
```
oc get db2ucluster <instance_id> -o jsonpath='{.status.state} {"\n"}'
```

Run the following command to check the status of your Db2 service instances:
```
cpd-cli service-instance status ${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME} \
--service-type=db2oltp
```

Run the following command to check that the service instances have updated:
```
cpd-cli service-instance list \
--profile=${CPD_PROFILE_NAME} \
--service-type=db2oltp
```

Check whether your Db2 service instances are in running state:
```
cpd-cli service-instance status ${INSTANCE_NAME} \
--profile=${CPD_PROFILE_NAME} \
--service-type=db2oltp
```

Continue with the upgrade of Data Gate custom resource


### Upgrade the Data Gate custom resource (est. 10 minutes):

Complete the following tasks before you run the actual Data Gate upgrade


### [Activating the Db2 Connect Unlimited Edition license](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrade-activating-db2-connect-unlimited)

As a prerequisite for all upgrades, the license for your JDBC driver must be activated

This step requires access to the DB2 Z database. It is probably necessary to request assistance from a DB2 DBA


### [Cleaning up the data-gate-api container](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=upgrade-cleaning-up-data-gate-api)

Open a shell in the data-gate-api container of the Data Gate pod
```
oc get pod -n cpd-instance | grep data-gate
```

Identify the Data Gate pod
```
dg-1737478278318519-data-gate-f4fff8f58-rp72p
```

```
oc exec -it -c data-gate-api dg-1737478278318519-data-gate-f4fff8f58-rp72p -- bash
```

Run the following command
```
rm -rf /head/clone-api/work/jetty-0_0_0_0-8188-clone-api_war-_clone_system-any-/webapp/* 2> /dev/null
```

Exit the container shell and proceed with the next check

Ensure that the datagate instance is not in an "InMaintenance" state, due to the parameter "ignoreForMaintenance":"true"
```
oc get datagateinstanceservice -n cpd-instance
```

If the datagate instance status is "InMaintenance" apply this patch
```
oc patch datagateinstanceservices $(oc get datagateinstanceservice -n cpd-instance | tail -n 1 | cut -d ' cpd-instance --patch '{"spec":{"ignoreForMaintenance":"false"}}' --type=merge ' -f1) -n
```


### Upgrade the custom resource for Data Gate (est. 5 minutes):
```
cpd-cli manage apply-cr \
--components=datagate \
--release=${VERSION} \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--block_storage_class=${STG_CLASS_BLOCK} \
--file_storage_class=${STG_CLASS_FILE} \
--license_acceptance=true \
--upgrade=true \
--case_download=false
```

Validating the upgrade: Data Gate is upgraded when the apply-cr command returns:

[SUCCESS]... The apply-cr command ran successfully

Monitor the progress of Datagate custom resource upgrade with the following command:
```
watch -n 5 'oc get pods -A -o wide | grep -Ev "([0-9]+)/\1|Completed"; echo; oc get DatagateService -n cpd-instance -o yaml | grep "progress"'
```

If you want to confirm that the custom resource status is Completed, you can run the cpd-cli manage get-cr-status command:
```
cpd-cli manage get-cr-status \
--cpd_instance_ns=${PROJECT_CPD_INST_OPERANDS} \
--components=datagate
```


### Upgrade the Data Gate service instances (est. 18 minutes):

**IMPORTANT: Potential issue after initiating Data Gate service instance upgrade**

During Data Gate instance upgrade a dg-1750101262664465-backup-head-job pod runs, but it should be scheduled to any node where Db2 is not running

For this cordon the node hosting Db2, and after the backup-head-job pod is completed, uncordon the same node

If the node hosting Db2u engine pod has a taint, this should also prevent this issue from occurring

**IMPORTANT: Potential issue after the upgrade of Data Gate service instances**

After Data Gate instances have been upgraded, it is possible that the configuration settings of the Db2 target database are not correctly migrated during the upgrade

Missing Db2 configuration settings can cause a variety of issues, in which the 'DB2COMM=TCPIP,SSL' was missing

To resolve this, we had to add the missing Db2 setting by using 'db2set DB2COMM=TCPIP,SSL'

These configurations should persist through a Db2u pod recycle

Keep track of your Db2 configuration settings prior to upgrading to ensure you have a list of settings which you can restore

For example
```
oc rsh c-db2oltp-1764799139398922-db2u-0 su - db2inst1
```

```
db2set
```

```
DB2_WAITFORDATA_LIBNAME=/mnt/blumeta0/home/db2inst1/config_db2u/waitfordata/libs/libwaitForDataSharedLib.so
DB2_ENABLE_MULTIPLE_CCSIDS_FOR_LOB_COLUMNS=YES [DB2_WORKLOAD]
DB2_REMOTE_EXTTAB_PIPE_PATH=/db2u/tmp
DB2_ENABLE_LOCAL_DATE_FORMAT=YES [DB2_WORKLOAD]
DB2_4K_DEVICE_SUPPORT=ON
DB2_ENABLE_CCSID_SEMANTICS=YES [DB2_WORKLOAD]
DB2_ENABLE_MULTIPLE_CCSIDS=YES [DB2_WORKLOAD]
DB2_FMP_RUN_AS_CONNECTED_USER=NO
DB2_STATISTICS=USCC:0;DISCOVER:ON;CGS_SAMPLE_RATE_ADJUST:0;RAND_COLGROUPID:Y;LEN21_COLGROUPNAME:Y;AUTO_SAMPLING_IMPRV:ON
DB2_OVERRIDE_THREADING_DEGREE=2
DB2_OVERRIDE_NUM_CPUS=3
DB2_RESTORE_GRANT_ADMIN_AUTHORITIES=ON
DB2_ATS_ENABLE=YES
DB2_WORKLOAD=ANALYTICS_ACCELERATOR
DB2RSHCMD=/bin/ssh
DB2AUTH=OSAUTHDB,ALLOW_LOCAL_FALLBACK,PLUGIN_AUTO_RELOAD
DB2_USE_ALTERNATE_PAGE_CLEANING=ON [DB2_WORKLOAD]
DB2_APPENDERS_PER_PAGE=1
DB2_LOAD_COPY_NO_OVERRIDE=COPY YES TO /mnt/bludata0/db2/copy
DB2_SELECTIVITY=ALL,AJSEL,UNIQUE_COL_FF ON
DB2_ANTIJOIN=EXTEND [DB2_WORKLOAD]
DB2MAXFSCRSEARCH=1
DB2CHECKCLIENTINTERVAL=100
DB2LIBPATH=/mnt/blumeta0/home/db2inst1/config_db2u/waitfordata/libs
DB2LOCK_TO_RB=STATEMENT
DB2COMM=TCPIP,SSL
```

You can also [change configuration settings](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=configuration-changing-db2-settings) after you deploy your instance

Before you proceed, make sure the CPD profile is set up

### [Creating a profile to use the cpd-cli management commands](https://www.ibm.com/docs/en/software-hub/5.1.x?topic=cli-creating-cpd-profile)

Proceed with Upgrading Datagate Instances
```
cpd-cli service-instance upgrade \
--all \
--service-type=dg \
--profile=${CPD_PROFILE_NAME} \
--watch
```

Verifying the service instance upgrade

Run the following command and wait for the status to change to Completed:
```
oc get dginstance <instance_id> -o jsonpath='{.status.datagateInstanceStatus} {"\n"}'
```

Run the following command and see if the Provision status has changed to UPGRADED:
```
watch cpd-cli service-instance list --profile=${CPD_PROFILE_NAME} --service-type dg
```


### Restart Data Gate synchronization

**IMPORTANT: From the CP4D UI, re-enable Data Gate synchronization**


## 8. Potential Issues To Be Aware Of

1. Setup-instance command will fail with 'imagepullbackoff' errors if the storage test images are missing. Mirror the storage test images ahead of time or exclude the '--run_storage_tests' flag

2. For imagepullbackoff errors, ensure that all required images are mirrored to the private registry, for example:
```
skopeo copy docker://icr.io/cpopen/ibm-operator-catalog@sha256:01712920b400fba751f60c71c27fbd64ca1b59b6cd325c42ae691f0b8770133d docker://lpdza532.phx.aexp.com:5000/cpopen/ibm-operator-catalog@sha256:01712920b400fba751f60c71c27fbd64ca1b59b6cd325c42ae691f0b8770133d \--all \--remove-signatures \--authfile=/var/registry/oc4.7/installer/pullsecret/merged_pullsecret.json
```

```
skopeo copy docker://icr.io/cpopen/cpd/olm-utils-v3@sha256:5a5b9b563756b3a41b22b2c9b6e940d3ceed6e12100e351809eb7f09ed058905 docker://lppza417.gso.aexp.com:5000/cpopen/cpd/olm-utils-v3@sha256:5a5b9b563756b3a41b22b2c9b6e940d3ceed6e12100e351809eb7f09ed058905 --all --remove-signatures --authfile=/root/merged_pullsecret.json
```

3. For insufficient CPU, insufficient memory, untolerated taint issues, ensure there are enough resources available on the cluster. Perform a pre-upgrade health check to ensure the cluster is in a healthy state
prior to the upgrade

4. If the Db2 license is not upgraded to the compatible version you may run into issues during Db2 instance upgrade. Follow the steps to "[Upgrade the license before you deploy Db2](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=setup-upgrading-license-before-you-deploy-db2)"

5. During the Db2 instance upgrade, the Db2ckupgrade.sh utility runs a job that must be scheduled to the same node as the Db2 engine pod. To resolve this issue, restrict the job to run on the same node as the
engine pod. Use a cordon, to ensure that this job is scheduled to the node hosting Db2. In previous upgrades, we've cordoned nodes lpqzo504, lpqzo505, and lpqzo506, such that the job would be scheduled to lpqzo507

6. During the Db2 instance upgrade you may observe issues related to tempspace1, "Table space access is not allowed." This problem is related to the usage of local storage and results in missing files and missing
folder structures within the Db2 pod. To address this, you will need to create a specific directory structure inside the Db2u pod and copy the container tag to this location.

7. During Data Gate instance upgrade a dg-1750101262664465-backup-head-job pod runs, but it should be scheduled to any node where Db2 is not running. For this cordon the node hosting Db2, and after the backup-head-job pod is completed, uncordon the same node

8. After Data Gate instances have been upgraded, it is possible that the configuration settings of the Db2 target database are not correctly migrated during the upgrade. Missing Db2 configuration settings can
cause a variety of issues, as experienced most recently during the E3-IPC1 upgrade, in which the DB2COMM=TCPIP,SSL was missing. To resolve this, we had to add the missing Db2 setting by using 'db2set
DB2COMM=TCPIP,SSL'. These configurations should persist through a Db2u pod recycle. Keep track of your Db2 configuration settings prior to upgrading to ensure you have a list of settings which you can restore.
You can also [change configuration settings](https://www.ibm.com/docs/en/software-hub/5.2.x?topic=configuration-changing-db2-settings) after you deploy your instance

This marks the end of the upgrade of IBM Software Hub, Db2oltp, Datagate

## 9. Validate CPD upgrade 

Perform final technical and functional checks

End of document
