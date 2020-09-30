# openshift-pipelines

## Deploy Openshift Pipelines (Tekton) Operator

Deploy the OpenShift Pipelines operator to all namespaces in the cluster and deploy the `tkn` command line tool to your local machine or bastion host.

## Setup Application Environments

Setup steps for "employee" example application.

### Create projects

```
export BUILD_PROJECT=employee-build
export TEST_PROJECT=employee-test
export PRODUCTION_PROJECT=employee
oc new-project $BUILD_PROJECT
oc new-project $TEST_PROJECT
oc new-project $PRODUCTION_PROJECT
```

### Deploy data services into test and production environments

```
oc process openshift//mysql-ephemeral -p MYSQL_USER=demo MYSQL_PASSWORD=demo MYSQL_ROOT_PASSWORD=root MYSQL_DATABASE=employee | oc create -f - -n $TEST_PROJECT
oc process openshift//mysql-ephemeral -p MYSQL_USER=demo MYSQL_PASSWORD=demo MYSQL_ROOT_PASSWORD=root MYSQL_DATABASE=employee | oc create -f - -n $PRODUCTION_PROJECT
TODO:  automate the following:
        oc get pods
        oc rsh mysql-1-RRRRR
        mysql --user=root
        mysql> use employee
        create table emp(id INT NOT NULL AUTO_INCREMENT,name VARCHAR(30) NOT NULL,title VARCHAR(30) NOT NULL,primary key (id));
        insert into emp values(default,'Darren','CEO');
        insert into emp values(default,'Mike','ADMIN');
        insert into emp values(default,'Kenny','CONSULTANT');
        quit
        exit
```

### Deploy application into projects

```
oc new-app registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift~https://github.com/meidlinger/sdgdemoboot --name=springboot-demo -n $BUILD_PROJECT
oc new-app registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift~https://github.com/meidlinger/sdgdemoboot --name=springboot-demo -n $TEST_PROJECT
oc new-app registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift~https://github.com/meidlinger/sdgdemoboot --name=springboot-demo -n $PRODUCTION_PROJECT
oc expose svc springboot-demo -n $TEST_PROJECT
oc expose svc springboot-demo -n $PRODUCTION_PROJECT
```

### Redefine ImageStreams

```
oc delete is springboot-demo -n $TEST_PROJECT
oc delete is springboot-demo -n $PRODUCTION_PROJECT
oc tag image-registry.openshift-image-registry.svc:5000/employee-build/springboot-demo image-registry.openshift-image-registry.svc:5000/employee-build/springboot-demo:latest
oc policy add-role-to-group system:image-puller system:serviceaccounts:${TEST_PROJECT} -n $BUILD_PROJECT
oc policy add-role-to-group system:image-puller system:serviceaccounts:${PRODUCTION_PROJECT} -n $TEST_PROJECT
oc import-image image-registry.openshift-image-registry.svc:5000/${BUILD_PROJECT}/springboot-demo:latest --confirm -n $TEST_PROJECT
oc import-image image-registry.openshift-image-registry.svc:5000/${TEST_PROJECT}/springboot-demo:latest --confirm -n $PRODUCTION_PROJECT
```

### Test deployment

```
export TEST_URL="http://`oc get route springboot-demo -n $TEST_PROJECT -o jsonpath --template="{.spec.host}"`/v1/api/sdg/demo/person/checktitle?name=Kenny"
export PRODUCTION_URL="http://`oc get route springboot-demo -n $PRODUCTION_PROJECT -o jsonpath --template="{.spec.host}"`/v1/api/sdg/demo/person/checktitle?name=Kenny"
echo -e "\nTEST:        $TEST_URL\nPRODUCTION:  $PRODUCTION_URL\n"
```

## Deploy/Test Building Blocks Needed for the Pipeline

Pipeline Resources:
* Image
* Git Repository

Tasks:
* Build Container
* Promote Container to Test
* Test Container
* Promote Container to Production

### Pipeline Resources

Apply the image and repository resources to the employee application build project.

```
cat pipelineresources/employee_resources.yaml | envsubst | oc create -f - -n $BUILD_PROJECT
```

### Build Container Task

```
tkn clustertask start s2i-java-11 --inputresource source=app-repo --outputresource image=app-image --showlog --param TLSVERIFY=false  -n $BUILD_PROJECT --dry-run
```

### Promote Container to Test Task

```
tkn clustertask start openshift-client --showlog --param SCRIPT="oc import-image image-registry.openshift-image-registry.svc:5000/${BUILD_PROJECT}/springboot-demo:latest" -n $TEST_PROJECT
```

### Test Container Task

```
tkn clustertask start openshift-client --showlog --param SCRIPT="[ \"`curl -s http://springboot-demo-${TEST_PROJECT}.apps.cluster-17d8.sandbox171.opentlc.com/v1/api/sdg/demo/person/checktitle?name=KENNY`\" = \"CONSULTANT\" ] || exit 1" -n $TEST_PROJECT
```


### Promote Container to Production Task

```
tkn clustertask start openshift-client --showlog --param SCRIPT="oc import-image image-registry.openshift-image-registry.svc:5000/${TEST_PROJECT}/springboot-demo:latest" -n $PRODUCTION_PROJECT
```

## Deploy Pipeline


