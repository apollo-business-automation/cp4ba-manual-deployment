# Cloud Pak for Business Automation Production deployment manual installation ✍️<!-- omit in toc -->

For version 22.0.1 IF004

Installs BAW Authoring environment.

- [Disclaimer ✋](#disclaimer-)
- [Prerequisites](#prerequisites)
- [Needed tooling](#needed-tooling)
- [Access info after deployment](#access-info-after-deployment)
- [Install preparation](#install-preparation)
- [Command line preparation in install Pod](#command-line-preparation-in-install-pod)
- [Prerequisite software](#prerequisite-software)
  - [OpenLdap](#openldap)
  - [PostgreSql](#postgresql)
- [Cloud Pak for Business Automation](#cloud-pak-for-business-automation)
  - [Get certified kubernetes folder](#get-certified-kubernetes-folder)
  - [Cluster setup](#cluster-setup)
  - [Prepare property files](#prepare-property-files)
  - [Generate, update and apply SQL and Secret files and validate the connectivity](#generate-update-and-apply-sql-and-secret-files-and-validate-the-connectivity)
  - [Create and deploy CP4BA CR](#create-and-deploy-cp4ba-cr)
  - [Post-installation](#post-installation)
- [Contacts](#contacts)
- [Notice](#notice)

## Disclaimer ✋

This is **not** an official IBM documentation.  
Absolutely no warranties, no support, no responsibility for anything.  
Use it on your own risk and always follow the official IBM documentations.  
It is always your responsibility to make sure you are license compliant when using this repository to install IBM Cloud Pak for Business Automation.

Please do not hesitate to create an issue here if needed. Your feedback is appreciated.

Not for production use. Suitable for Demo and PoC environments - but with Production deployment.  

## Prerequisites

- Empty OpenShift cluster
- With direct internet connection
- File RWX StorageClass - in this case managed-nfs-storage is used everywhere, feel free to find and replace
- Block RWO StorageClass (optional) - in this case managed-nfs-storage is used which is not Block. When asked for RWO class use your own Block based.
- cluster admin user
- IBM entitlement key from https://myibm.ibm.com/products-services/containerlibrary

## Needed tooling

Provided in the install Pod.

- podman
- openjdk9
- helm
- oc
- kubectl
- yq

## Access info after deployment

OpenLdap
- https://phpldapadmin-openldap.<ocp_apps_domain>/
- cn=admin,dc=cp,dc=local / Password

PostgreSql
- See Project postgresql, Service postgresql for assigned NodePort 
- cpadmin / Password
- In PostgreSql Pod terminal `psql postgresql://cpadmin:Password@localhost:5432/postgresdb`

Cloud Pak for Business Automation
- https://cpd-cp4ba.<ocp_apps_domain>/
- cpadmin / Password
- Additional capabilities based info in Project cp4ba in ConfigMap icp4adeploy-cp4ba-access-info

## Install preparation

In you OCP cluster create Project install using OpenShift console.

Create PVC where all files used for installation would be kept if you would need to re-instantiate the install Pod.
```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: install
  namespace: install
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: managed-nfs-storage
  volumeMode: Filesystem
```

Create a pod from which the install will be performed with the following definition.  
It is ready when the message *Install pod - Ready* is in the Log.
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: install
  namespace: install
spec:
  containers:
    - name: install
      securityContext:
        privileged: true    
      image: ubi9/ubi:9.0.0
      command: ["/bin/bash"]
      args:
        ["-c","cd /usr;
          yum install podman -y;
          yum install ncurses -y;
          curl -O https://download.java.net/java/GA/jdk9/9/binaries/openjdk-9_linux-x64_bin.tar.gz;
          tar -xvf openjdk-9_linux-x64_bin.tar.gz;
          ln -fs /usr/jdk-9/bin/java /usr/bin/java;
          ln -fs /usr/jdk-9/bin/keytool /usr/bin/keytool;
          curl -O https://get.helm.sh/helm-v3.6.0-linux-amd64.tar.gz;
          tar -zxvf helm-v3.6.0-linux-amd64.tar.gz linux-amd64/helm;
          mv linux-amd64/helm helm;
          chmod u+x helm;
          ln -fs /usr/helm /usr/bin/helm;
          curl -k https://mirror.openshift.com/pub/openshift-v4/clients/ocp/stable/openshift-client-linux.tar.gz --output oc.tar;
          tar -xvf oc.tar oc;
          chmod u+x oc;
          ln -fs /usr/oc /usr/bin/oc;
          curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl;
          chmod u+x kubectl;
          ln -fs /usr/kubectl /usr/bin/kubectl;
          curl -LO https://github.com/mikefarah/yq/releases/download/v4.30.5/yq_linux_amd64.tar.gz;
          tar -xzvf yq_linux_amd64.tar.gz;
          ln -fs /usr/yq_linux_amd64 /usr/bin/yq;
          podman -v;
          java -version;
          keytool;
          helm version;
          oc version;
          kubectl version;
          yq --version;
          while true;
          do echo 'Install pod - Ready - Enter it via Terminal and \"bash\"';
          sleep 300; done"]      
      imagePullPolicy: IfNotPresent
      volumeMounts:
        - name: install
          mountPath: /usr/install
  volumes:
    - name: install
      persistentVolumeClaim:
        claimName: install
```

## Command line preparation in install Pod

This needs to be done if you don't perform this in one go and you come back to resume.

Open Terminal window of the newly created pod.

Enter bash.
```
bash
```

Login with your OC user using your own token and server address.
```bash
oc login --token=sha256~kwZnxyz1Uekt-IWQJKsXmvabca4LAJtpiBlFiMEPLZ_U \
--server=https://c108-e.eu-gb.containers.cloud.ibm.com:31624
```

## Prerequisite software

CP4BA needs at least LDAP and Database. In this deployment OpenLDAP and postgresql are used.

You would normally use your own production ready instances.

Please note that OpenLDAP is working but is not officially supported.

### OpenLdap

Add helm repository for OpenLdap
```bash
helm repo add helm-openldap https://jp-gouin.github.io/helm-openldap/
```

Create openldap-values.yaml file which defines the whole deployment.
```bash 
cat <<EOF > /usr/install/openldap-values.yaml
replicaCount: 1
env:
  LDAP_ORGANISATION: "cp.local"
  LDAP_DOMAIN: "cp.local"
global:  
  adminPassword: 'Password'
  configPassword: 'Password'
customLdifFiles:
  01-default-users.ldif: |-
    # Units
    dn: ou=Users,dc=cp,dc=local
    objectClass: organizationalUnit
    ou: Users

    dn: ou=Groups,dc=cp,dc=local
    objectClass: organizationalUnit
    ou: Groups

    # Users
    dn: uid=cpadmin,ou=Users,dc=cp,dc=local
    objectClass: inetOrgPerson
    objectClass: top
    cn: cpadmin
    sn: cpadmin
    uid: cpadmin
    mail: cpadmin@cp.local
    userpassword:: UGFzc3dvcmQ=
    employeeType: admin

    dn: uid=cpuser,ou=Users,dc=cp,dc=local
    objectClass: inetOrgPerson
    objectClass: top
    cn: cpuser
    sn: cpuser
    uid: cpuser
    mail: cpuser@cp.local
    userpassword:: UGFzc3dvcmQ=

    # Groups
    dn: cn=cpadmins,ou=Groups,dc=cp,dc=local
    objectClass: groupOfNames
    objectClass: top
    cn: cpadmins
    member: uid=cpadmin,ou=Users,dc=cp,dc=local

    dn: cn=cpusers,ou=Groups,dc=cp,dc=local
    objectClass: groupOfNames
    objectClass: top
    cn: cpusers
    member: uid=cpadmin,ou=Users,dc=cp,dc=local
    member: uid=cpuser,ou=Users,dc=cp,dc=local
replication:
  enabled: false
persistence:
  enabled: true
  accessModes:
    - ReadWriteMany
  size: 8Gi
  storageClass: "managed-nfs-storage"
livenessProbe:
  initialDelaySeconds: 60
readinessProbe:
  initialDelaySeconds: 60
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
ltb-passwd:
  enabled: false
phpldapadmin:
  enabled: true
  ingress:
    enabled: false
  env:
    PHPLDAPADMIN_LDAP_HOSTS: "openldap.openldap.svc.cluster.local"

EOF
```

Create openldap Project
```bash
echo "
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: openldap
" | oc apply -f -
```

Add required anyuid premission
```bash
echo "
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: openldap-anyuid
  namespace: openldap
subjects:
  - kind: ServiceAccount
    name: default
    namespace: openldap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:anyuid
" | oc apply -f -
```

Install OpenLdap
```bash
helm install openldap helm-openldap/openldap-stack-ha \
-f /usr/install/openldap-values.yaml -n openldap --version 3.0.2
```

Add Route for phpLdapAdmin
```bash
echo "
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: phpldapadmin
  namespace: openldap
spec:
  to:
    kind: Service
    name: openldap-phpldapadmin
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
  wildcardPolicy: None
" | oc apply -f -
```

Wait for all pods in openldap Project to be Ready 1/1.

### PostgreSql

Create postgresql Project
```bash
echo "
kind: Project
apiVersion: project.openshift.io/v1
metadata:
  name: postgresql
" | oc apply -f -
```

Add required privileged premission
```bash
echo "
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: postgresql-privileged
  namespace: postgresql
subjects:
  - kind: ServiceAccount
    name: default
    namespace: postgresql
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:openshift:scc:privileged
" | oc apply -f -
```

Create Secret for config
```bash
echo "
kind: Secret
apiVersion: v1
metadata:
  name: postgresql-config
  namespace: postgresql
stringData:
  POSTGRES_DB: postgresdb
  POSTGRES_USER: cpadmin
  POSTGRES_PASSWORD: Password
" | oc apply -f -
```

Create PVC for data
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-data
  namespace: postgresql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: managed-nfs-storage
  volumeMode: Filesystem
" | oc apply -f -
```

Create PVC for table spaces as they should not be in PGDATA
```bash
echo "
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgresql-tablespaces
  namespace: postgresql
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: managed-nfs-storage
  volumeMode: Filesystem
" | oc apply -f -
```

Create Deployment
```bash
echo "
kind: Deployment
apiVersion: apps/v1
metadata:
  name: postgresql
  namespace: postgresql  
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          securityContext:
            privileged: true
          image: postgres:13.9-alpine3.16
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
          envFrom:
            - secretRef:
                name: postgresql-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgresql-data
            - mountPath: /pgsqldata
              name: postgresql-tablespaces
      volumes:
        - name: postgresql-data
          persistentVolumeClaim:
            claimName: postgresql-data
        - name: postgresql-tablespaces
          persistentVolumeClaim:
            claimName: postgresql-tablespaces
" | oc apply -f -
```

Create Service
```bash
echo "
kind: Service
apiVersion: v1
metadata:
  name: postgresql
  namespace: postgresql    
  labels:
    app: postgresql
spec:
  type: NodePort
  ports:
    - port: 5432
  selector:
    app: postgresql
" | oc apply -f -
```

Wait for pod in postgresql Project to become Ready 1/1.

Set max transactions
```bash
# Set the value
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb \
-c "ALTER SYSTEM SET max_prepared_transactions = 200;"'

# Restart the pod
oc --namespace postgresql delete pod \
$(oc get pods --namespace postgresql -o name | cut -d"/" -f2)
```

Wait for pod in postgresql Project to become Ready 1/1.

## Cloud Pak for Business Automation

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=deployments-installing-cp4ba-multi-pattern-production-deployment

### Get certified kubernetes folder

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=deployment-preparing-client-connect-cluster
```bash
# Download the package
curl https://raw.githubusercontent.com/IBM/cloud-pak/master/repo/case/\
ibm-cp-automation/4.0.4/ibm-cp-automation-4.0.4.tgz \
--output /usr/install/ibm-cp-automation-4.0.4.tgz

# Extract the package
tar xzvf /usr/install/ibm-cp-automation-4.0.4.tgz -C /usr/install

# Extract cert-kubernetes
tar xvf /usr/install/ibm-cp-automation/inventory/\
cp4aOperatorSdk/files/deploy/crs/cert-k8s-22.0.1.tar \
-C /usr/install
```

### Cluster setup

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=cluster-setting-up-by-running-script

```bash
/usr/install/cert-kubernetes/scripts/cp4a-clusteradmin-setup.sh
```

```text
Select the cloud platform to deploy: 
1) RedHat OpenShift Kubernetes Service (ROKS) - Public Cloud
2) Openshift Container Platform (OCP) - Private Cloud
3) Other ( Certified Kubernetes Cloud Platform / CNCF)
Enter a valid option [1 to 3]: 1 # Or 2 based on your platform

What type of deployment is being performed?
1) Starter
2) Production
Enter a valid option [1 to 2]: 2

Do you want CP4BA Operator support 'All Namespaces'? (Yes/No, default: No) No

Where do you want to deploy Cloud Pak for Business Automation?
Enter the name for a new project or an existing project (namespace): cp4ba

The Cloud Pak for Business Automation Operator (Pod, CSV, Subscription) not found in cluster
Continue....

Using project cp4ba...

Here are the existing users on this cluster: 
1) Cluster Admin
2) Your User
Enter an existing username in your cluster, valid option [1 to 3], non-admin is suggested: 2
ATTENTION: When you run cp4a-deployment.sh script, please use cluster admin user.


Follow the instructions on how to get your Entitlement Key: 
https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=images-getting-access-from-public-entitled-registry

Do you have a Cloud Pak for Business Automation Entitlement Registry key (Yes/No, default: No): Yes

Enter your Entitlement Registry key: # it is OK that when you paste nothing is seen - it is for security.
Verifying the Entitlement Registry key...
Login Succeeded!
Entitlement Registry key is valid.

Please use available storage classes.

The existing storage classes in the cluster: 
...omitted

Creating docker-registry secret for Entitlement Registry key in project cp4ba...
secret/ibm-entitlement-key created
Done
ibm-cp4a-operator-catalog  ibm-cp4a-operator  grpc   IBM  26h
Found existing ibm operator catalog source, updating it
catalogsource.operators.coreos.com/ibm-cp4a-operator-catalog unchanged
catalogsource.operators.coreos.com/ibm-cp-automation-foundation-catalog unchanged
catalogsource.operators.coreos.com/ibm-automation-foundation-core-catalog unchanged
catalogsource.operators.coreos.com/opencloud-operators unchanged
catalogsource.operators.coreos.com/bts-operator unchanged
catalogsource.operators.coreos.com/cloud-native-postgresql-catalog unchanged
IBM Operator Catalog source updated!
Waiting for CP4A Operator Catalog pod initialization
CP4BA Operator Catalog is running ibm-cp4a-operator-catalog-g624g  1/1   Running  0  26h
operatorgroup.operators.coreos.com/ibm-cp4a-operator-catalog-group created
CP4BA Operator Group Created!
subscription.operators.coreos.com/ibm-cp4a-operator-catalog-subscription created
CP4BA Operator Subscription Created!

Waiting for CP4BA operator pod initialization
No resources found in cp4ba namespace.

CP4A operator is running ibm-cp4a-operator-76c4c85584-pjxrf  1/1   Running   0     17s

Waiting for CP4BA Content operator pod initialization
CP4A Content operator is running ibm-content-operator-58fb9986d7-ccpzk   1/1   Running   0     32s


Label the default namespace to allow network policies to open traffic to the ingress controller using a namespaceSelector...namespace/default not labeled
Done

Storage classes are needed to run the deployment script. For the Starter deployment scenario, you may use one (1) storage class.  For an Production deployment, the deployment script will ask for three (3) storage classes to meet the slow, medium, and fast storage for the configuration of CP4A components.  If you don't have three (3) storage classes, you can use the same one for slow, medium, or fast.  Note that you can get the existing storage class(es) in the environment by running the following command: oc get storageclass. Take note of the storage classes that you want to use for deployment. 
```

Wait until the script finishes.

### Prepare property files

Generate properties for components that you would like to deploy.
```bash
/usr/install/cert-kubernetes/scripts/cp4a-prerequisites.sh -m property
```

Answer as follows.
```text
Tips:Press [ENTER] to accept the default (None of the patterns is selected)
Enter a valid option [1 to 4, 5a, 5b, 6, 7a, 7b]: 5a

Hit Enter to continue.

Do you want to enable Business Automation Application Data Persistence? (Yes/No): No

Pattern "(a) Workflow Authoring": Select optional components: 
1) Business Automation Insights 

Tips: Press [ENTER] to accept the default (None of the components is selected)
Enter a valid option [1 to 1 or ENTER]: 

Hit Enter to continue.

What is the LDAP type used for this deployment? 
1) Microsoft Active Directory
2) Tivoli Directory Server / Security Directory Server
Enter a valid option [1 to 2]: 2

What is the Database type used for this deployment? 
1) IBM Db2 Database
2) Oracle
3) Microsoft SQL Server
4) PostgreSQL
Enter a valid option [1 to 4]: 4

Please input the alias name(s) for database server(s)/instance(s) which will be used by CP4BA deployment.
(NOTES: NOT host name of database server, and CAN NOT include dot[.] character)
(NOTE: This key supports comma-separated lists (for example: dbserver1,dbserver2,dbserver3)
The alias name(s): postgresql
```

Update values for cp4ba_LDAP.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property /usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property.bak

# Update generated file with real values
sed -i \
-e 's/LDAP_SERVER="<Required>"/LDAP_SERVER="openldap.openldap.svc.cluster.local"/g' \
-e 's/LDAP_PORT="<Required>"/LDAP_PORT="389"/g' \
-e 's/LDAP_BASE_DN="<Required>"/LDAP_BASE_DN="dc=cp,dc=local"/g' \
-e 's/LDAP_BIND_DN="<Required>"/LDAP_BIND_DN="cn=admin,dc=cp,dc=local"/g' \
-e 's/LDAP_BIND_DN_PASSWORD="<Required>"/LDAP_BIND_DN_PASSWORD="Password"/g' \
-e 's/LDAP_SSL_ENABLED="True"/LDAP_SSL_ENABLED="False"/g' \
-e 's/LDAP_USER_NAME_ATTRIBUTE="<Required>"/LDAP_USER_NAME_ATTRIBUTE="*:cn"/g' \
-e 's/LDAP_USER_DISPLAY_NAME_ATTR="<Required>"/LDAP_USER_DISPLAY_NAME_ATTR="cn"/g' \
-e 's/LDAP_GROUP_BASE_DN="<Required>"/LDAP_GROUP_BASE_DN="ou=Groups,dc=cp,dc=local"/g' \
-e 's/LDAP_GROUP_NAME_ATTRIBUTE="<Required>"/LDAP_GROUP_NAME_ATTRIBUTE="*:cn"/g' \
-e 's/LDAP_GROUP_DISPLAY_NAME_ATTR="<Required>"/LDAP_GROUP_DISPLAY_NAME_ATTR="cn"/g' \
-e 's/LDAP_GROUP_MEMBERSHIP_SEARCH_FILTER="<Required>"/LDAP_GROUP_MEMBERSHIP_SEARCH_FILTER="(|(\&(objectclass=groupOfNames)(member={0}))(\&(objectclass=groupofuniquenames)(uniquemember={0})))"/g' \
-e 's/LDAP_GROUP_MEMBER_ID_MAP="<Required>"/LDAP_GROUP_MEMBER_ID_MAP="groupOfNames:member"/g' \
-e 's/LC_USER_FILTER="<Required>"/LC_USER_FILTER="(\&(uid=%v)(objectclass=inetOrgPerson))"/g' \
-e 's/LC_GROUP_FILTER="<Required>"/LC_GROUP_FILTER="(\&(cn=%v)(|(objectclass=groupOfNames)(objectclass=groupofuniquenames)(objectclass=groupofurls)))"/g' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_LDAP.property
```

Update values for cp4ba_db_name_user.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property /usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property.bak

# Update generated file with real values
sed -i \
-e 's/postgresql.GCD_DB_USER_NAME="<youruser1>"/postgresql.GCD_DB_USER_NAME="gcd"/g' \
-e 's/postgresql.GCD_DB_USER_PASSWORD="<yourpassword>"/postgresql.GCD_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.BAWDOCS_DB_USER_NAME="<youruser1>"/postgresql.BAWDOCS_DB_USER_NAME="bawdocs"/g' \
-e 's/postgresql.BAWDOCS_DB_USER_PASSWORD="<yourpassword>"/postgresql.BAWDOCS_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.BAWDOS_DB_USER_NAME="<youruser1>"/postgresql.BAWDOS_DB_USER_NAME="bawdos"/g' \
-e 's/postgresql.BAWDOS_DB_USER_PASSWORD="<yourpassword>"/postgresql.BAWDOS_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.BAWTOS_DB_USER_NAME="<youruser1>"/postgresql.BAWTOS_DB_USER_NAME="bawtos"/g' \
-e 's/postgresql.BAWTOS_DB_USER_PASSWORD="<yourpassword>"/postgresql.BAWTOS_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.CHOS_DB_USER_NAME="<youruser1>"/postgresql.CHOS_DB_USER_NAME="chos"/g' \
-e 's/postgresql.CHOS_DB_USER_PASSWORD="<yourpassword>"/postgresql.CHOS_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.ICN_DB_USER_NAME="<youruser1>"/postgresql.ICN_DB_USER_NAME="icn"/g' \
-e 's/postgresql.ICN_DB_USER_PASSWORD="<yourpassword>"/postgresql.ICN_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.APP_ENGINE_DB_USER_NAME="<youruser1>"/postgresql.APP_ENGINE_DB_USER_NAME="aae"/g' \
-e 's/postgresql.APP_ENGINE_DB_USER_PASSWORD="<yourpassword>"/postgresql.APP_ENGINE_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.AUTHORING_DB_USER_NAME="<youruser1>"/postgresql.AUTHORING_DB_USER_NAME="bawau"/g' \
-e 's/postgresql.AUTHORING_DB_USER_PASSWORD="<yourpassword>"/postgresql.AUTHORING_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.APP_PLAYBACK_DB_USER_NAME="<youruser1>"/postgresql.APP_PLAYBACK_DB_USER_NAME="app"/g' \
-e 's/postgresql.APP_PLAYBACK_DB_USER_PASSWORD="<yourpassword>"/postgresql.APP_PLAYBACK_DB_USER_PASSWORD="Password"/g' \
-e 's/postgresql.STUDIO_DB_USER_NAME="<youruser1>"/postgresql.STUDIO_DB_USER_NAME="bas"/g' \
-e 's/postgresql.STUDIO_DB_USER_PASSWORD="<yourpassword>"/postgresql.STUDIO_DB_USER_PASSWORD="Password"/g' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_name_user.property
```

Update values for cp4ba_db_server.property
```bash
# Backup generated file
cp /usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property /usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property.bak

# Update generated file with real values
sed -i \
-e 's/postgresql.DATABASE_SERVERNAME="<Required>"/postgresql.DATABASE_SERVERNAME="postgresql.postgresql.svc.cluster.local"/g' \
-e 's/postgresql.DATABASE_PORT="<Required>"/postgresql.DATABASE_PORT="5432"/g' \
-e 's/postgresql.DATABASE_SSL_ENABLE="True"/postgresql.DATABASE_SSL_ENABLE="False"/g' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/propertyfile/cp4ba_db_server.property
```

### Generate, update and apply SQL and Secret files and validate the connectivity

Generate SQL and Secrets
```bash
/usr/install/cert-kubernetes/scripts/cp4a-prerequisites.sh -m generate
```
PostgreSql instance configured in a way that tablespace path in some of the generated SQL create scripts doesn't need updating.

Copy create scripts to PostgreSQL instance
```bash
oc cp /usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/dbscript \
postgresql/$(oc get pods --namespace postgresql -o name | cut -d"/" -f2):/usr/dbscript
```

Execute create scripts with table space directory creation
```bash
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/ae/postgresql/postgresql/create_ae_playback_db.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/ae/postgresql/postgresql/create_app_engine_db.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/icndb; chown postgres:postgres /pgsqldata/icndb;'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/ban/postgresql/postgresql/createICNDB.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/icndb; chown postgres:postgres /pgsqldata/icndb;'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/bas/postgresql/postgresql/create_bas_studio_db.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/baw-authoring/postgresql/postgresql/create_baw_db.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/bawdocs; chown postgres:postgres /pgsqldata/bawdocs;'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/fncm/postgresql/postgresql/createBAWDOCS.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/bawdos; chown postgres:postgres /pgsqldata/bawdos;'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/fncm/postgresql/postgresql/createBAWDOS.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/bawtos; chown postgres:postgres /pgsqldata/bawtos;'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/fncm/postgresql/postgresql/createBAWTOS.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/chos; chown postgres:postgres /pgsqldata/chos;'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/fncm/postgresql/postgresql/createCHOS.sql'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'mkdir /pgsqldata/gcd; chown postgres:postgres /pgsqldata/gcd;'
oc --namespace postgresql exec deploy/postgresql -- /bin/bash -c \
'psql postgresql://cpadmin:Password@localhost:5432/postgresdb --file=/usr/dbscript/fncm/postgresql/postgresql/createGCDDB.sql'
```

Update secrets
```bash
# ban
yq -i '.stringData.appLoginUsername = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/ban/ibm-ban-secret.yaml
yq -i '.stringData.appLoginPassword = "Password"' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/ban/ibm-ban-secret.yaml
yq -i 'del(.stringData.jMailUsername)' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/ban/ibm-ban-secret.yaml
yq -i 'del(.stringData.jMailPassword)' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/ban/ibm-ban-secret.yaml
yq -i '.stringData.ltpaPassword = "Password"' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/ban/ibm-ban-secret.yaml
yq -i '.stringData.keystorePassword = "Password"' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/ban/ibm-ban-secret.yaml
# fncm
yq -i '.stringData.appLoginUsername = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/fncm/ibm-fncm-secret.yaml
yq -i '.stringData.appLoginPassword = "Password"' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/fncm/ibm-fncm-secret.yaml
yq -i '.stringData.ltpaPassword = "Password"' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/fncm/ibm-fncm-secret.yaml
yq -i '.stringData.keystorePassword = "Password"' \
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/secret_template/fncm/ibm-fncm-secret.yaml
```

Apply secrets
```bash
oc project cp4ba
/usr/install/cert-kubernetes/scripts/cp4ba-prerequisites/create_secret.sh
```

Validate connectivity
```bash
/usr/install/cert-kubernetes/scripts/cp4a-prerequisites.sh -m validate
```
Look for all green checkmarks.

### Create and deploy CP4BA CR

Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=deployment-creating-production

Run deployment script  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=deployment-generating-custom-resource-script
```bash
/usr/install/cert-kubernetes/scripts/cp4a-deployment.sh
```

```text
Enter for licence acceptance
accept Yes
Content No
type 2
enter to confirm selection
profiel 1 small
platform 1 ROKS
zip file none - enter
slow storage managed-nfs-storage
medium storage managed-nfs-storage
fast storage managed-nfs-storage
RWO managed-nfs-storage
proceed yes
```

Update CR file  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=deployment-checking-completing-your-custom-resource
```bash
yq -i '.spec.shared_configuration.sc_deployment_fncm_license = "non-production"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.shared_configuration.sc_deployment_baw_license = "non-production"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.shared_configuration.sc_deployment_license = "non-production"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.bastudio_configuration.admin_user  = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.bastudio_configuration.playback_server.admin_user = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.application_engine_configuration[0].admin_user = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.workflow_authoring_configuration.admin_user = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_ldap_creation.ic_ldap_admin_user_name[0] = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_ldap_creation.ic_ldap_admins_groups_name[0] = "cpadmins"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[0].oc_cpe_obj_store_admin_user_groups[0] = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[0].oc_cpe_obj_store_admin_user_groups[1] = "cpadmins"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[1].oc_cpe_obj_store_admin_user_groups[0] = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[1].oc_cpe_obj_store_admin_user_groups[1] = "cpadmins"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[2].oc_cpe_obj_store_workflow_data_tbl_space = "bawtos_tbs"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[2].oc_cpe_obj_store_workflow_admin_group = "cpadmins"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[2].oc_cpe_obj_store_workflow_config_group = "cpadmins"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[2].oc_cpe_obj_store_workflow_pe_conn_point_name = "pe_conn_batos"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[2].oc_cpe_obj_store_admin_user_groups[0] = "cpadmin"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
yq -i '.spec.initialize_configuration.ic_obj_store_creation.object_stores[2].oc_cpe_obj_store_admin_user_groups[1] = "cpadmins"' \
/usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

Apply CR  
Based on https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=deployment-deploying-custom-resource-you-created-script
```bash
oc apply -f /usr/install/cert-kubernetes/scripts/generated-cr/ibm_cp4a_cr_final.yaml
```

Wait for the deployment to be completed. Can be determined by looking in Kind ICP4ACluster, instance named icp4adeploy to have the following conditions:
```yaml
  conditions:
    - message: Running reconciliation
      reason: Running
      status: 'True'
      type: Running
    - message: Prerequisites execution done.
      reason: Successful
      status: 'True'
      type: PrereqReady
    - message: ''
      reason: Successful
      status: 'True'
      type: Ready
```      

### Post-installation

Follow https://www.ibm.com/docs/en/cloud-paks/cp-biz-automation/22.0.1?topic=deployment-completing-post-installation-tasks as needed.

## Contacts

Jan Dusek  
jdusek@cz.ibm.com  
Business Automation Technical Specialist  
IBM Czech Republic

## Notice

© Copyright IBM Corporation 2022.
