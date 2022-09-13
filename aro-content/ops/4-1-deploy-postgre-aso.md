## Deploy Database for Minesweeper application through ASO
Azure Service Operator(ASO) is an open-source project by Microsoft Azure. ASO gives you the ability to provision and manages Azure resources within the Kubernetes plane by using familiar Kubernetes tooling and primitives. ASO consists of:
1. Custom Resource Definitions (CRDs) for each of the Azure services that a Kubernetes user can provision.
2. A Kubernetes controller that manages the Azure resources represented by the user-specified Custom Resources. The controller attempts to synchronize the desired state in the user-specified Custom Resource with the actual state of that resource in Azure, creating it if it doesn't exist, updating it if it has been changed, or deleting it.

In this task, we use ASO to provision a PostgreSQL DB and connect applications to Azure resources from within Kubernetes

### Prerequisites

* an ARO cluster

* oc cli

* Azure Service Operator(ASO) operator v2
  
  


## Provision DB for Minesweeper APP

to provision a PostgreSQL DB you need to create the following objects in your cluster:
 - ResourceGroup  
 - FlexibleServer  
 - FlexibleServersDatabase 
 - FlexibleServersFirewallRule

 **ResourceGroup**  **(if you don't have Resource Group)**
```
cat <<EOF | oc apply -f -
apiVersion: resources.azure.com/v1beta20200601
kind: ResourceGroup
metadata:
  name: wksp-rg
  namespace: default
spec:
  location: eastus
EOF
```
 **Provision PostgreSQL flexible server**

1. **create a secret for the DB server**
```
cat <<EOF | oc apply -f -
apiVersion : v1
kind : Secret
metadata : 
  name : server-admin-pw
  namespace : default
data:
  password: aGFja2F0aG9uUGFzcw==
type: Opaque
EOF
```
      
2. **create DB server**
      
 ```
 cat <<EOF | oc apply -f -
apiVersion: dbforpostgresql.azure.com/v1beta20210601
kind: FlexibleServer
metadata:
  name: wksp-pqslserver
  namespace: default
spec:
  location: eastus
  owner:
    name: wksp-rg
  version: "13"
  sku:
    name: Standard_B1ms
    tier: Burstable
  administratorLogin: myAdmin
  administratorLoginPassword: # This is the name/key of a Kubernetes secret in the same namespace
    name: server-admin-pw
    key: password
  storage:
    storageSizeGB: 32
EOF
 ```

3.  **create Server configuration**
```
cat  <<EOF | oc apply -f -
apiVersion: dbforpostgresql.azure.com/v1beta20210601
kind: FlexibleServersConfiguration
metadata:
  name: pgaudit
  namespace: default
spec:
  owner:
    name: wksp-pqslserver
  azureName: pgaudit.log
  source: user-override
  value: READ
EOF
```
4. **create a firewall rule for the database**
```
cat  <<EOF | oc apply -f -
apiVersion: dbforpostgresql.azure.com/v1beta20210601
kind: FlexibleServersFirewallRule
metadata:
  name: wksp-fw-rule
  namespace: default
spec:
  owner:
    name: wksp-pqslserver
  startIpAddress: 0.0.0.0
  endIpAddress: 255.255.255.255
EOF
```

**Note: it takes about 10 minutes for the database to be operational and running** 

 
5. **create a sample DB**
```
cat  <<EOF | oc apply -f -
apiVersion: dbforpostgresql.azure.com/v1beta20210601
kind: FlexibleServersDatabase
metadata:
  name: wksp-db
  namespace: default
spec:
  owner:
    name: wksp-pqslserver
  charset: utf8

EOF
```


6. **check connection to DB server**

```
psql "host=wksp-pqslserver.postgres.database.azure.com port=5432 dbname=wksp-db user=myAdmin password=<password> sslmode=require"
```