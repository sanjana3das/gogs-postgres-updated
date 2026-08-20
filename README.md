# Gogs + PostgreSQL on Kubernetes — Complete Practical

## 1. Objective

Deploy **Gogs** and **PostgreSQL** on Kubernetes with:

- PostgreSQL and Gogs in different namespaces
- Kubernetes Secrets for database credentials
- PostgreSQL exposed internally using a `ClusterIP` Service
- Gogs exposed using a `NodePort` Service
- Gogs connected to PostgreSQL using Kubernetes DNS
- Gogs repository created from the GUI
- Repository cloned using the clone URL copied from the Gogs GUI
- Local Git repository initialized/committed and pushed to Gogs
- Troubleshooting of common clone, Git identity, and authentication problems

---

# 2. Final Architecture

```text
                         Kubernetes Cluster
                                |
              +-----------------+-----------------+
              |                                   |
       namespace: postgres                 namespace: gogs
              |                                   |
       +--------------+                    +--------------+
       | PostgreSQL   |                    |    Gogs      |
       |     Pod      |                    |     Pod      |
       |              |                    |              |
       |   :5432      |                    |   :3000      |
       +------+-------+                    +------+-------+
              |                                   |
       +------+-------+                           |
       | PostgreSQL   |                           |
       |   Service    |<--------------------------+
       |   :5432      |
       +--------------+

External Git Client / Browser
              |
              | HTTPS
              v
       Gogs NodePort :30030
              |
              v
          Gogs Pod
              |
              v
      PostgreSQL Service
              |
              v
       PostgreSQL Pod
```

---

# 3. Important Concepts

## Namespace

We intentionally separated the applications:

```text
postgres namespace
    -> PostgreSQL

gogs namespace
    -> Gogs
```

Namespaces provide logical separation.

---

## Secret

A Secret stores sensitive values such as:

```text
username
password
database name
database host
```

The Secret itself does **not** create a database connection.

It only provides configuration/credentials to the application.

---

## Environment Variables

Secrets are injected into containers as environment variables.

Conceptually:

```text
Secret
   |
   v
Environment Variables
   |
   v
Gogs Application
```

---

## Service

A Service gives an application a stable network endpoint.

We do NOT connect Gogs directly to the changing PostgreSQL Pod IP.

Instead:

```text
Gogs
  |
  v
PostgreSQL Service
  |
  v
PostgreSQL Pod
```

The PostgreSQL Service selects the PostgreSQL Pod using labels.

Example:

```yaml
selector:
  app: postgres
```

and the Pod has:

```yaml
labels:
  app: postgres
```

---

# 4. PostgreSQL Namespace

Create namespaces:

```bash
kubectl create namespace postgres
kubectl create namespace gogs
```

Verify:

```bash
kubectl get ns
```

---

# 5. PostgreSQL Secret

File:

```text
postgres-secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-secret
  namespace: postgres
type: Opaque
stringData:
  POSTGRES_USER: gogs
  POSTGRES_PASSWORD: gogs123
  POSTGRES_DB: gogs
```

Apply:

```bash
kubectl apply -f postgres-secret.yaml
```

Check:

```bash
kubectl get secret -n postgres
```

---

# 6. PostgreSQL Deployment

File:

```text
postgres-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: postgres

spec:
  replicas: 1

  selector:
    matchLabels:
      app: postgres

  template:
    metadata:
      labels:
        app: postgres

    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine

          ports:
            - containerPort: 5432

          env:
            - name: POSTGRES_USER
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_USER

            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_PASSWORD

            - name: POSTGRES_DB
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: POSTGRES_DB
```

Apply:

```bash
kubectl apply -f postgres-deployment.yaml
```

Check:

```bash
kubectl get pods -n postgres
```

---

# 7. PostgreSQL Service

File:

```text
postgres-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: postgres

spec:
  type: ClusterIP

  selector:
    app: postgres

  ports:
    - port: 5432
      targetPort: 5432
```

Apply:

```bash
kubectl apply -f postgres-service.yaml
```

Check:

```bash
kubectl get svc -n postgres
```

Check Service endpoints:

```bash
kubectl get endpoints -n postgres
```

The endpoint should point to the PostgreSQL Pod on port `5432`.

---

# 8. PostgreSQL Connection Test

Check PostgreSQL:

```bash
kubectl get pods -n postgres
```

Connect using `psql`:

```bash
kubectl exec -it -n postgres deployment/postgres -- \
psql -U gogs -d gogs
```

If this gives:

```text
gogs=#
```

PostgreSQL is working.

Exit:

```sql
\q
```

---

# 9. PostgreSQL Kubernetes DNS

The PostgreSQL Service is:

```text
postgres
```

The namespace is:

```text
postgres
```

Therefore the full Kubernetes DNS name is:

```text
postgres.postgres.svc.cluster.local
```

With PostgreSQL port:

```text
postgres.postgres.svc.cluster.local:5432
```

Breakdown:

```text
postgres
    -> Service name

postgres
    -> Namespace

svc
    -> Kubernetes Service domain

cluster.local
    -> Kubernetes cluster DNS domain
```

This is how Gogs can find PostgreSQL even though Gogs and PostgreSQL are in different namespaces.

---

# 10. Gogs Secret

File:

```text
gogs-secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gogs-secret
  namespace: gogs
type: Opaque
stringData:
  DB_TYPE: postgres
  DB_HOST: postgres.postgres.svc.cluster.local:5432
  DB_NAME: gogs
  DB_USER: gogs
  DB_PASSWORD: gogs123
  SECRET_KEY: change-this-to-a-long-random-secret
```

Apply:

```bash
kubectl apply -f gogs-secret.yaml
```

Check:

```bash
kubectl get secret -n gogs
```

---

# 11. Gogs Deployment

File:

```text
gogs-deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gogs
  namespace: gogs

spec:
  replicas: 1

  selector:
    matchLabels:
      app: gogs

  template:
    metadata:
      labels:
        app: gogs

    spec:
      containers:
        - name: gogs
          image: gogs/gogs:0.14

          ports:
            - containerPort: 3000
            - containerPort: 22

          env:
            - name: DB_TYPE
              valueFrom:
                secretKeyRef:
                  name: gogs-secret
                  key: DB_TYPE

            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: gogs-secret
                  key: DB_HOST

            - name: DB_NAME
              valueFrom:
                secretKeyRef:
                  name: gogs-secret
                  key: DB_NAME

            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: gogs-secret
                  key: DB_USER

            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: gogs-secret
                  key: DB_PASSWORD

            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: gogs-secret
                  key: SECRET_KEY

          volumeMounts:
            - name: gogs-data
              mountPath: /data

      volumes:
        - name: gogs-data
          emptyDir: {}
```

Apply:

```bash
kubectl apply -f gogs-deployment.yaml
```

Check:

```bash
kubectl get pods -n gogs
```

Check logs:

```bash
kubectl logs -n gogs deployment/gogs
```

---

# 12. Gogs Service

File:

```text
gogs-service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gogs
  namespace: gogs

spec:
  type: NodePort

  selector:
    app: gogs

  ports:
    - name: http
      port: 3000
      targetPort: 3000
      nodePort: 30030

    - name: ssh
      port: 22
      targetPort: 22
      nodePort: 30022
```

Apply:

```bash
kubectl apply -f gogs-service.yaml
```

Check:

```bash
kubectl get svc -n gogs
```

---

# 13. Verify Gogs Environment Variables

Get the Pod:

```bash
kubectl get pods -n gogs
```

Enter the Pod:

```bash
kubectl exec -it -n gogs <GOGS-POD-NAME> -- sh
```

Check DB variables:

```bash
env | grep DB_
```

Expected values should correspond to:

```text
DB_TYPE=postgres
DB_HOST=postgres.postgres.svc.cluster.local:5432
DB_NAME=gogs
DB_USER=gogs
DB_PASSWORD=...
```

Exit:

```bash
exit
```

---

# 14. Test DNS from Gogs Pod

Inside the Gogs Pod:

```bash
getent hosts postgres.postgres.svc.cluster.local
```

If an IP is returned, Kubernetes DNS resolution is working.

Conceptual flow:

```text
Gogs Pod
   |
   | postgres.postgres.svc.cluster.local
   v
Kubernetes DNS
   |
   v
postgres Service
   |
   v
PostgreSQL Pod
```

---

# 15. Access Gogs GUI

In Killercoda, Gogs was exposed through the NodePort/proxy.

Example external URL:

```text
https://88ad023846e5-10-244-5-204-30030.spca.r.killercoda.com
```

The important part is:

```text
30030
```

which corresponds to the Kubernetes NodePort.

---

# 16. Create a Gogs User

For the first user, use a username such as:

```text
Username: sanjana
Email: sanjana@example.com
Password: <your chosen Gogs password>
```

Important:

**Gogs login username and Git commit identity are different things.**

Gogs login:

```text
Username + Gogs password
```

Git commit identity:

```text
git config user.name
git config user.email
```

---

# 17. Create Repository from Gogs GUI

In Gogs:

```text
+ New Repository
```

Repository name:

```text
demo-project
```

Create it.

The repository page provides a Clone URL.

---

# 18. Important Clone URL Problem We Hit

The Gogs GUI originally displayed:

```text
http://localhost:3000/sanjana/demo-project.git
```

We tried:

```bash
git clone http://localhost:3000/sanjana/demo-project.git
```

It failed:

```text
fatal: unable to access ...
Failed to connect to localhost port 3000
```

### Why?

`localhost` means:

```text
the machine/container where the Git command is being executed
```

It does NOT automatically mean the Gogs Pod.

Our Gogs was actually exposed through the Killercoda URL.

The working clone URL was:

```text
https://88ad023846e5-10-244-5-204-30030.spca.r.killercoda.com/sanjana/demo-project.git
```

Clone:

```bash
git clone https://88ad023846e5-10-244-5-204-30030.spca.r.killercoda.com/sanjana/demo-project.git
```

Result:

```text
Cloning into 'demo-project'...
warning: You appear to have cloned an empty repository.
```

### Important

This warning is NOT a clone failure.

It means:

```text
Gogs repository exists
        +
Git can reach Gogs
        +
Repository currently has no commits
```

---

# 19. Enter the Repository

```bash
cd demo-project
```

Create a file:

```bash
echo "Hello from Gogs on Kubernetes" > README.md
```

Stage it:

```bash
git add README.md
```

---

# 20. Git Commit Identity

When committing, Git initially returned:

```text
Author identity unknown

Please tell me who you are.
```

This happened because Git did not know the local commit author's name/email.

Configure:

```bash
git config --global user.name "sanjana"
git config --global user.email "sanjana@example.com"
```

Check:

```bash
git config --global --list
```

Then commit:

```bash
git commit -m "Initial commit"
```

Expected:

```text
[master (root-commit) ...] Initial commit
 1 file changed, 1 insertion(+)
 create mode 100644 README.md
```

---

# 21. Check Branch

```bash
git branch
```

In our practical:

```text
* master
```

Therefore push:

```bash
git push -u origin master
```

---

# 22. Authentication Failure We Hit

The push asked:

```text
Username:
sanjana

Password:
```

Then returned:

```text
fatal: Authentication failed
```

This is a separate issue from the earlier clone problem.

The clone was successful, proving:

```text
Git Client
    |
    v
Gogs external URL
    |
    v
Gogs Service
    |
    v
Gogs Pod
```

The push authentication failure means the credentials supplied during `git push` were not accepted by Gogs.

### Do not confuse these two:

```text
Git commit identity
        !=
Gogs login credentials
```

`git config` only identifies the author of a commit.

It does NOT set the Gogs account password.

---

# 23. Verify the Remote

```bash
git remote -v
```

Expected:

```text
origin  https://88ad023846e5-10-244-5-204-30030.spca.r.killercoda.com/sanjana/demo-project.git (fetch)
origin  https://88ad023846e5-10-244-5-204-30030.spca.r.killercoda.com/sanjana/demo-project.git (push)
```

If the remote is wrong:

```bash
git remote set-url origin https://88ad023846e5-10-244-5-204-30030.spca.r.killercoda.com/sanjana/demo-project.git
```

Then:

```bash
git push -u origin master
```

---

# 24. Complete Git Flow

The complete Git part we performed:

```text
Gogs GUI
    |
    | Create Repository
    v
demo-project
    |
    | Copy Clone URL
    v
Git Client
    |
    | git clone
    v
Local Repository
    |
    | create README.md
    v
git add
    |
    v
git commit
    |
    v
git push
    |
    v
Gogs Repository
```

---

# 25. Most Important Troubleshooting Lessons

## Problem 1 — localhost clone failed

Wrong:

```bash
git clone http://localhost:3000/sanjana/demo-project.git
```

Reason:

```text
localhost = current machine
```

Use the externally reachable Gogs URL instead.

---

## Problem 2 — Empty repository warning

```text
warning: You appear to have cloned an empty repository.
```

This is normal for a newly created repository with no commits.

---

## Problem 3 — Git commit failed

```text
Author identity unknown
```

Fix:

```bash
git config --global user.name "sanjana"
git config --global user.email "sanjana@example.com"
```

---

## Problem 4 — Git push authentication failed

```text
fatal: Authentication failed
```

This is a Gogs authentication problem, not a Git commit identity problem.

Verify:

1. Correct Gogs username
2. Correct Gogs password
3. Correct repository URL
4. Correct Gogs account
5. Gogs logs if necessary

---

# 26. One-Line Mental Model

Remember this:

```text
Secret = credentials/configuration
Service = stable network address
DNS = finds Service by name
Namespace = logical separation
Deployment = manages Pods
Pod = runs the application
Gogs = Git server/application
PostgreSQL = Gogs database
Git Client = talks to Gogs, not directly to PostgreSQL
```

And the database connection:

```text
Gogs
 |
 | DB_HOST
 | postgres.postgres.svc.cluster.local:5432
 v
PostgreSQL Service
 |
 v
PostgreSQL Pod
```

The Git connection:

```text
Git Client
 |
 | HTTPS
 v
Gogs Service / external URL
 |
 v
Gogs Pod
 |
 v
Repository
```

---

# 27. Practical Verification Commands

```bash
kubectl get pods -n postgres
kubectl get svc -n postgres
kubectl get endpoints -n postgres

kubectl get pods -n gogs
kubectl get svc -n gogs
kubectl logs -n gogs deployment/gogs

kubectl get secrets -n postgres
kubectl get secrets -n gogs
```

Git:

```bash
git status
git branch
git remote -v
git log --oneline
```

---

# 28. Production Improvements

This practical uses:

```text
emptyDir
```

for Gogs data only to keep the learning lab simple.

For production, use:

```text
PersistentVolume
PersistentVolumeClaim
StorageClass
```

Also use stronger credentials and proper Secret management.

For external access, normally use:

```text
Ingress
TLS/HTTPS
DNS
```

instead of directly exposing NodePort.

For Git authentication, production environments should use an appropriate secure authentication mechanism rather than putting passwords into command history or scripts.

---

# 29. Final Architecture to Remember

```text
                    Developer
                        |
                        | git clone / push
                        v
                 Gogs Service
                    :30030
                        |
                        v
                   Gogs Pod
                        |
             +----------+----------+
             |                     |
       Git Repository         DB connection
                                   |
                                   | DNS
                                   v
                       postgres.postgres.svc.cluster.local
                                   |
                                   v
                           PostgreSQL Service
                                   |
                                   v
                           PostgreSQL Pod
                                   |
                                   v
                              Database
                                gogs
```

This is the complete **Gogs + PostgreSQL + Kubernetes + Secrets + Services + Git Client** practical.
