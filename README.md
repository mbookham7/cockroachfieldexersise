# Deploy cockroachdb via a Helm chart
Add the Helm repo and update
```
helm repo add cockroachdb https://charts.cockroachdb.com/
helm repo update
```
Install the Helm chart with some overiding values.
```
helm install my-release --values my-values.yaml cockroachdb/cockroachdb
```
Check the database pods are up and running.
```
kubectl get pods
```
Check to see is the persistent volumes are up and running
```
kubectl get pv
```

Connect to the database server using a dedicated pod. Create a database to ensure that cockroach is running.

Connect to the UI. Open and new terminal
```
kubectl port-forward my-release-cockroachdb-0 8080
```
Then you can access the admin UI at http://localhost:8080/ in your web browser.

```
kubectl run cockroachdb -it \
--image=cockroachdb/cockroach:v21.1.2 \
--rm \
--restart=Never \
-- sql \
--insecure \
--host=my-release-cockroachdb-public
```
Create a database
```
CREATE DATABASE bank;
```
Create a table.
```
CREATE TABLE bank.accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
      balance DECIMAL
  );
```
Insert a table called accounts.
```
INSERT INTO bank.accounts (balance)
  VALUES
      (1000.50), (20000), (380), (500), (55000);
```
Display table
```
SELECT * FROM bank.accounts;
```
Quit
```
\q
```

## Run a test workload

First we need to grab the Cluster IP. As we are running in Kubernetes we can use this as our LB address.
```
kubectl get svc
```
Output
```
NAME                            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
kubernetes                      ClusterIP   10.96.0.1      <none>        443/TCP              3h50m
my-release-cockroachdb          ClusterIP   None           <none>        26257/TCP,8080/TCP   3h49m
my-release-cockroachdb-public   ClusterIP   10.97.92.150   <none>        26257/TCP,8080/TCP   3h49m
```

Exec into one of the db pods and run the following.
```
kubectl exec my-release-cockroachdb-0 -it bash
```

Prepare the pod to run the test
```
cockroach workload init bank \
'postgresql://root@<clusterIP>:26257?sslmode=disable'
```

Run a 60 minuite test
```
cockroach workload run bank \
--duration=60m \
'postgresql://root@<clusterIP>:26257?sslmode=disable'
```

## Start our scaling tests

Update the Helm deployment to 4 replicas
```
helm upgrade \
my-release \
cockroachdb/cockroachdb \
--set statefulset.replicas=4 \
--reuse-values
```

Performance is hit, number of inserts dropped and latency increased. Insert never get back to the previous level. Log commit latency also increased.

Gracefully remove nodes, Get a list of replicas.
```
kubectl run cockroachdb -it \
--image=cockroachdb/cockroach:v21.1.2 \
--rm \
--restart=Never \
-- node status \
--insecure \
--host=my-release-cockroachdb-public
```
Remove replica 4.
```
kubectl run cockroachdb -it \
--image=cockroachdb/cockroach:v21.1.2 \
--rm \
--restart=Never \
-- node decommission 4 \
--insecure \
--host=my-release-cockroachdb-public
```
Update the Helm deployment to 3 replicas
```
helm upgrade \
my-release \
cockroachdb/cockroachdb \
--set statefulset.replicas=3 \
--reuse-values
```

Next forcibly remove a replica....
```
kubectl delete pod my-release-cockroachdb-2
```
Admin console reporting suspect replica, kubernetes replaces pod with new replica attached the original storage. Load test stops "server is not accepting clients".
```
kubectl get pod my-release-cockroachdb-2
```

Remove all but 1 replica.
```
kubectl scale statefulsets my-release-cockroachdb --replicas=1
```
Lost connection to the Database.

# Questions to answer


1. As you were adding and removing nodes from the cluster, how did that impact
performance? What kinds of metrics were you tracking to identify that impact?
Answer - Number of inserts dropped and SQL Service Latency Increased.

2. What other kinds of behavior did you witness as you were changing the cluster
topology? How did the system handle the hard node failure differently than the graceful
shutdown?
My load test stopped and had to be restarted when I forcibly removed nodes. Where as it carried on during graceful activities.

3. When you killed all of the nodes but one, what happened to the database?
I lost connection to the database.

4. Did the platform behave differently than you would expect in any of the above
scenarios? If so please describe.
I was surprised I lost connection even though I had one node remaining.

Scale the StatefulSet back up to 3 replicas.
```
kubectl scale statefulsets my-release-cockroachdb --replicas=3
```
Next create an application to connect to the database.
Start the built-in SQL client:

```
kubectl run cockroachdb -it \
--image=cockroachdb/cockroach:v21.1.2 \
--rm \
--restart=Never \
-- sql \
--insecure \
--host=my-release-cockroachdb-public
```

In the SQL shell, issue the following statements to create the maxroach user and bank database:
```
DROP DATABASE bank CASCADE;
```

```
CREATE USER IF NOT EXISTS maxroach;
CREATE DATABASE bank;
```
Give the maxroach user the necessary permissions:
```
GRANT ALL ON DATABASE bank TO maxroach;
```
Exit the SQL shell:
```
\q
```
Run a pod where we will run our go application.
```
kubectl run -i -t golang --image=golang:1.12-alpine --restart=Never
```

```
apk add -U build-base git
go get -u github.com/lib/pq
```
```
vi basic-sample.go
```
Copy code into the file....
```
package main

import (
    "database/sql"
    "fmt"
    "log"

    _ "github.com/lib/pq"
)

func main() {
    // Connect to the "bank" database.
    db, err := sql.Open("postgres", "postgresql://maxroach@my-release-cockroachdb-public:26257/bank?sslmode=disable")
    if err != nil {
        log.Fatal("error connecting to the database: ", err)
    }

    // Create the "accounts" table.
    if _, err := db.Exec(
        "CREATE TABLE IF NOT EXISTS accounts (id INT PRIMARY KEY, balance INT)"); err != nil {
        log.Fatal(err)
    }

    // Insert two rows into the "accounts" table.
    if _, err := db.Exec(
        "INSERT INTO accounts (id, balance) VALUES (1, 1000), (2, 250)"); err != nil {
        log.Fatal(err)
    }

    // Print out the balances.
    rows, err := db.Query("SELECT id, balance FROM accounts")
    if err != nil {
        log.Fatal(err)
    }
    defer rows.Close()
    fmt.Println("Initial balances:")
    for rows.Next() {
        var id, balance int
        if err := rows.Scan(&id, &balance); err != nil {
            log.Fatal(err)
        }
        fmt.Printf("%d %d\n", id, balance)
    }
}
```

Run the application
```
go run basic-sample.go
```
Run second example..
```
vi txn-sample.go
```
Copy code below into the file
```
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"

    "github.com/cockroachdb/cockroach-go/crdb"
)

func transferFunds(tx *sql.Tx, from int, to int, amount int) error {
    // Read the balance.
    var fromBalance int
    if err := tx.QueryRow(
        "SELECT balance FROM accounts WHERE id = $1", from).Scan(&fromBalance); err != nil {
        return err
    }

    if fromBalance < amount {
        return fmt.Errorf("insufficient funds")
    }

    // Perform the transfer.
    if _, err := tx.Exec(
        "UPDATE accounts SET balance = balance - $1 WHERE id = $2", amount, from); err != nil {
        return err
    }
    if _, err := tx.Exec(
        "UPDATE accounts SET balance = balance + $1 WHERE id = $2", amount, to); err != nil {
        return err
    }
    return nil
}

func main() {
    db, err := sql.Open("postgres", "postgresql://maxroach@my-release-cockroachdb-public:26257/bank?sslmode=disable")
    if err != nil {
        log.Fatal("error connecting to the database: ", err)
    }

    // Run a transfer in a transaction.
    err = crdb.ExecuteTx(context.Background(), db, nil, func(tx *sql.Tx) error {
        return transferFunds(tx, 1 /* from acct# */, 2 /* to acct# */, 100 /* amount */)
    })
    if err == nil {
        fmt.Println("Success")
    } else {
        log.Fatal("error: ", err)
    }
}
```
Run the application 
```
go get -d github.com/cockroachdb/cockroach-go
go run txn-sample.go
```
Exit go pod.
```
exit
```


To verify that funds were transferred from one account to another, use the built-in SQL client:
```
kubectl exec my-release-cockroachdb-0 -it bash
```
Run the following command inside the pod
```
cockroach sql --insecure -e 'SELECT id, balance FROM accounts' --database=bank
```
# Clean Up

```
helm uninstall my-release
```










