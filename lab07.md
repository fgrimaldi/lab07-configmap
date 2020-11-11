# Provisioning Kube Configuration

By the end of this exercise, you should be able to:

 - write kube yaml describing secrets and config maps, and associate them with pods and deployments.

## Provisioning ConfigMaps

1.  Create a file on `node-0` called `env-config` with the following content:

    ```bash
    user=moby
    db=mydb
    ```

2.  Create a configMap that parses each line in this file as a separate key/value pair:

    ```bash
    [centos@node-0 ~]$ kubectl create configmap dbconfig --from-env-file=env-config
    ```

3.  Ask kubectl to repeat this configMap back to us, in yaml:

    ```bash
    [centos@node-0 ~]$ kubectl get configmap dbconfig -o yaml

    apiVersion: v1
    data:
      db: mydb
      user: moby
    kind: ConfigMap
    metadata:
      creationTimestamp: "2019-01-08T15:41:31Z"
      name: dbconfig
      namespace: default
      resourceVersion: "709821"
      selfLink: /api/v1/namespaces/default/configmaps/dbconfig
      uid: df14e7a7-135b-11e9-87ee-0242ac11000a
    ```

    The key/value pairs parsed from `env-config` are visible under the `data` key in this file; we could have created the same configMap declaratively via `kubectl create -f <yaml filename>` like we've been doing so far, using this yaml.

    > **Imperative vs. Declarative kubectl commands**: `kubectl` supports two syntaxes for most actions: *imperative*, which specifies the action to take (like `kubectl create pod ...`, `kubectl describe deployment ...` etc), and *declarative*, which specifies objects in a yaml file and creates them with `kubectl create -f <yaml filename>`. Which to use is a matter of preference; in general, I recommend *declarative* (ie file based) commands for *any action that changes the state of the system* (ie create / update / destroy operations) so that these changes can be based off of easily tracked and version controlled config files, and *imperative* commands only for gathering information without changing anything (`kubectl get ...`, `kubectl decribe ...`). These are not strict rules (we just saw a convenient example of the imperative `kubectl create configmap` above, useful since an env-file specification is so much easier to write than the corresponding yaml), but lend themselves well to good record-keeping, automation and version control for your workloads.

4.  Describe a postgres database in a pod with the following `postgres.yaml`:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: dbdemo
      namespace: default
    spec:
      containers:
      - name: pg
        image: postgres:9.6
        env:
          - name: POSTGRES_USER
            valueFrom:
              configMapKeyRef:
                name: dbconfig
                key: user
          - name: POSTGRES_DB
            valueFrom:
              configMapKeyRef:
                name: dbconfig
                key: db
    ```

    Here we're populating the environment variables `POSTGRES_USER` and `POSTGRES_DB` from our configMap, under the `containers:env` specification. Notice that the pod definition itself makes no reference to the literal values of these environment variables; we can reconfigure our database (say for deployment in a different environment) by swapping out our `dbconfig` configMap, and leaving our pod definition untouched.

    Deploy your pod as usual with `kubectl create -f postgres.yaml`.

5.  Describe your `dbdemo` pod:

    ```bash
    [centos@node-0 ~]$ kubectl describe pod dbdemo

    ...

    Environment:
      POSTGRES_USER:  <set to the key 'user' of config map 'dbconfig'>  Optional: false
      POSTGRES_DB:    <set to the key 'db' of config map 'dbconfig'>    Optional: false

    ...
    ```

    You should see a block like the above, indicating that the listed environment variables are being populated from the configMap, as expected.

6.  Attach to your postgres database using the username and database name you specified in `config.yaml`, to prove to yourself the configMap information actually got consumed:

    ```bash
    [centos@node-0 ~]$ kubectl exec -it -c pg dbdemo -- psql -U moby -d mydb

    mydb=# \du
                                       List of roles
     Role name |                         Attributes                         | Member of
    -----------+------------------------------------------------------------+-----------
     moby      | Superuser, Create role, Create DB, Replication, Bypass RLS | {}

    mydb=# \q
    ```

    The user and database got created with the names we defined.

7.  Delete your pod with `kubectl delete -f postgres.yaml`.

8.  So far, we've used the environment variables postgres looks for for setting user and database names. Some config, however, is expected as a file rather than an environment variable; configMaps can provision config files as well as environment config. Create a database initialization script `db-init.sh`:

    ```
    #!/bin/bash
    set -e    

    psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
        CREATE TABLE PRODUCTS(PRICE FLOAT, NAME TEXT);
        INSERT INTO PRODUCTS VALUES('18.95', 'widget');
        INSERT INTO PRODUCTS VALUES('1.45', 'sprocket');
    EOSQL
    ```

9.  Turn this entire file into a configMap:

    ```bash
    [centos@node-0 ~]$ kubectl create configmap dbinit --from-file=db-init.sh
    ```

10. We'll provision this config file to our postgres container by mounting it in as a volume to the correct path; postgres will run all `.sh` files found at the path `/docker-entrypoint-initdb.d` upon initialization. Change your `postgres.yaml` file to look like this:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: dbdemo
      namespace: default
    spec:
      containers:
      - name: pg
        image: postgres:9.6
        volumeMounts:
        - name: dbinit-vol
          mountPath: /docker-entrypoint-initdb.d
        env:
          - name: POSTGRES_USER
            valueFrom:
              configMapKeyRef:
                name: dbconfig
                key: user
          - name: POSTGRES_DB
            valueFrom:
              configMapKeyRef:
                name: dbconfig
                key: db
      volumes:
        - name: dbinit-vol
          configMap:
            name: dbinit
    ```

    Here we've added the `volumeMounts` key describing which volume (`dbinit-vol`) to mount in the container and at what path (`/docker-entrypoint-initdb.d`). We've also added the `volumes` key to define the volumes themselves; we create one volume named `dbinit-vol`, populated from the files contained in the configMap `dbinit` we just created.

11. Deploy postgres with this configuration, and check that the database initialization script actually worked:

    ```bash
    [centos@node-0 ~]$ kubectl create -f postgres.yaml
    [centos@node-0 ~]$ kubectl exec -it -c pg dbdemo -- psql -U moby -d mydb

    mydb=# SELECT * FROM products;

     price |   name   
    -------+----------
     18.95 | widget
      1.45 | sprocket
    (2 rows)

    mydb=# \q    
    ```

    Our table was initialized via our config file, as expected. After exiting your pod, delete it with `kubectl delete -f postgres.yaml`.

## Provisioning Secrets

So far, we've provisioned non-sensitive data to our pod, but often we want options for added security when provisioning things like passwords or other access tokens. For this, Kubernetes maintains a separate config provisioning tool, *secrets*.

1.  Let's set a custom password for our database using a Kubernetes secret. Create a file `secret.yaml` with the following content:

    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: postgres-pwd
      namespace: default
    type: Opaque
    stringData:
      password: "mypassword"
    ```

    Create the secret via `kubectl create -f secret.yaml`, This will create a secret called `postgres-pwd` that encodes our password.

    > Note: Of course it's not recommended to leave your secret unencrypted in a file like `secret.yaml`; in practice, we'd delete this file as soon as the secret is created.

2.  Update your `postgres.yaml` definition to look like this:

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: dbdemo
      namespace: default
    spec:
      containers:
      - name: pg
        image: postgres:9.6
        volumeMounts:
        - name: dbinit-vol
          mountPath: /docker-entrypoint-initdb.d
        env:
          - name: POSTGRES_USER
            valueFrom:
              configMapKeyRef:
                name: dbconfig
                key: user
          - name: POSTGRES_DB
            valueFrom:
              configMapKeyRef:
                name: dbconfig
                key: db
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                name: postgres-pwd
                key: password
      volumes:
        - name: dbinit-vol
          configMap:
            name: dbinit
    ```

    This is exactly the same as above, but adds the block at the bottom which populates the `POSTGRES_PASSWORD` environment variable in the `pg` container with the value found under the `password` key in the `postgres-pwd` secret.

3.  Create your pod, and dump its postgres environment variables:

    ```bash
    [centos@node-0 ~]$ kubectl create -f postgres.yaml
    [centos@node-0 ~]$ kubectl exec -it -c pg dbdemo -- env | grep POSTGRES

    POSTGRES_USER=moby
    POSTGRES_DB=mydb
    POSTGRES_PASSWORD=mypassword
    ```

    The `POSTGRES_PASSWORD` has been provisioned from your Kube secret.

4.  Note that anyone with `kubectl get secret` permissions can recover your secret as follows:

    ```bash
    [centos@node-0 ~]$ kubectl get secret postgres-pwd -o yaml

    apiVersion: v1
    data:
      password: bXlwYXNzd29yZA==
    kind: Secret
    metadata:
      creationTimestamp: "2019-01-07T18:42:01Z"
      name: postgres-pwd
      namespace: default
      resourceVersion: "604412"
      selfLink: /api/v1/namespaces/default/secrets/postgres-pwd
      uid: ebb6f645-12ab-11e9-87ee-0242ac11000a
    type: Opaque

    [centos@node-0 ~]$ echo 'bXlwYXNzd29yZA==' | base64 --decode

    mypassword
    ```

    *Challenge*: the secret password can also be recovered in plain text on the node hosting the postgres pod. Can you find it?

5.  As usual, clean up by deleting the pods, configMaps, and secret you created in this exercise.

## Conclusion

In this exercise, we looked at configMaps and secrets, two tools for provisioning information to your deployments. When deciding where to place configuration, it can help to prioritize designing for reusability; in the postgres example we saw, we separated out all the environment config from the pod definition, so the same pod yaml could be migrated from environment to environment with no changes; all the environment specific data was captured in the configMaps and secret. If you find yourself having to do heavy reconfiguration of your pods and deployments (or even images) as you migrate from one environment to another, consider if it would be possible to separate this configuration from your pod definition using a configMap or secret.

[Back](../../lab07.md)
