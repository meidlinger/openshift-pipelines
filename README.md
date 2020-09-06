# openshift-pipelines

## Setup

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

### Test deployment

```
export TEST_URL="http://`oc get route springboot-demo -n $TEST_PROJECT -o jsonpath --template="{.spec.host}"`/v1/api/sdg/demo/person/checktitle?name=Kenny"
export PRODUCTION_URL="http://`oc get route springboot-demo -n $PRODUCTION_PROJECT -o jsonpath --template="{.spec.host}"`/v1/api/sdg/demo/person/checktitle?name=Kenny"
echo -e "\nTEST:        $TEST_URL\nPRODUCTION:  $PRODUCTION_URL\n"
```
