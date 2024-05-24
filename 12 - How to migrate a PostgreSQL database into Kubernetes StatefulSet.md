# How to migrate a PostgreSQL database into Kubernetes StatefulSet

## Introduction

Migrating an existing PostgreSQL database into a Kubernetes StatefulSet is an important task that can provide several benefits, including high availability, scalability, portability, and consistent deployment across different environments. It is particularly useful for running production databases with high availability requirements, deploying databases for microservices or cloud-native applications, migrating legacy databases to a more modern, cloud-native architecture, or consolidating multiple databases into a single, centrally managed Kubernetes cluster.

## Prereqs

- A running Kubernetes cluster
- `kubectl` installed and configured to access the Kubernetes cluster in desired namespace.
- `helm` installed
- A database with data to be migrated. Alternatively, you can use a [sample database dump from here](https://www.postgresqltutorial.com/wp-content/uploads/2019/05/dvdrental.zip).

## Step 1: Deploy PostgreSQL database server into your Kubernetes cluster using Helm

To deploy a PostgreSQL database server into your Kubernetes cluster, you can use the Bitnami PostgreSQL Helm chart, which provides a convenient way to package and deploy the necessary resources.

- **Add the Bitnami Helm repository:**

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

- **Install the PostgreSQL Helm chart:**

```sh
helm install my-postgresql bitnami/postgresql --version 15.4.0
```

This will deploy a PostgreSQL StatefulSet, along with associated resources like Services, Persistent Volumes, and Persistent Volume Claims.

- **Customize the deployment (optional):**
By default, the Helm chart creates an 8Gi Persistent Volume Claim for data storage. If you need to customize the storage size, use specific volumes or storage classes, or adjust other configuration values, you can override the default values using a custom `values.yaml` file or by setting individual values with the `--set` flag during installation.

- **Verify the deployment:**
Ensure that the StatefulSet, Persistent Volumes, Persistent Volume Claims, Pods, and Services have been successfully deployed and are running by checking their status using `kubectl` commands like `kubectl get statefulsets`, `kubectl get pv`, `kubectl get pvc`, `kubectl get pods`, and `kubectl get svc`.

- **Retrieve the PostgreSQL password:**
The PostgreSQL admin password is generated randomly during installation. Retrieve it from the Helm release notes:

```sh
helm get notes my-postgresql
```

## Step 2: Connect to the database server with a PostgreSQL Client; Create user; Create database

To connect to the newly deployed PostgreSQL server and set up the necessary users and databases for your migration, you'll need a PostgreSQL client.

- **Build a custom PostgreSQL client image (recommended):**
The Bitnami PostgreSQL client image provided in the Helm chart notes is quite limited in terms of additional utilities. It's recommended to build a custom image with the PostgreSQL client and other useful tools like `wget`, `curl`, `zip`, and `unzip`. This will make it easier to fetch the database dump file and import it later.

Here's an example `Dockerfile` for a custom PostgreSQL client image:

```Dockerfile
FROM ubuntu:latest

# Set the working directory to /app
WORKDIR /app

# Copy the database dump file into the image (optional)
# COPY dump.sql /app/dump.sql

# Install required packages
RUN apt-get update && \
    apt-get install -y postgresql-client-16 zip unzip wget && \
    rm -rf /var/lib/apt/lists/*

# Add PostgreSQL 16.2 bin directory to PATH
ENV PATH="/usr/lib/postgresql/16/bin:${PATH}"

# Verify PostgreSQL client installation and PATH
RUN echo "PostgreSQL client version:" && pg_config --version && \
    echo "PATH: $PATH"

# Run the shell
CMD ["/bin/bash"]
```

Build and push the image to a container registry:

```sh
docker build -t username/mypg-client:1.0 .
docker push username/mypg-client:1.0
```

- **Run the PostgreSQL client:**
You can run the PostgreSQL client as a one-off Pod or create a long-running Pod for subsequent use.

For a one-off Pod:

```sh
kubectl run my-postgresql-client --rm --tty -i --restart='Never' --image docker.io/username/mypg-client:1.0 --env="PGPASSWORD=$POSTGRES_PASSWORD" --command -- psql --host my-postgresql-postgresql -U postgres -d postgres -p 5432
```

For a long-running Pod:

```sh
kubectl run my-postgresql-client --image docker.io/username/mypg-client:1.0 --env="PGPASSWORD=$POSTGRES_PASSWORD"
kubectl exec my-postgresql-client -it -- psql --host my-postgresql-postgresql -U postgres -d postgres -p 5432
```

- **Create a new user and database:**
Once connected to the PostgreSQL server, create a new user and database for your migration:

```sql
CREATE USER databaseuser WITH PASSWORD 'password';
CREATE DATABASE incoming_db;
GRANT ALL PRIVILEGES ON DATABASE incoming_db TO databaseuser;
```

- **Exit the PostgreSQL client console:**
Exit the console, but keep the container running for later use:

```sh
\q
```

## Step 3: Prepare a dump of the current database

To migrate your existing PostgreSQL database, you'll need to create a dump file containing the data and schema.

- **Dump the database using a GUI tool or `pg_dump`:**
You can use a graphical database management tool like DBeaver or TablePlus to export your database as a dump file, or use the `pg_dump` command-line utility:

```sh
pg_dump -h old-db-host -p 5432 -U postgres old_db -Ft -f /tmp/dumpfile.tar
```

This command creates a compressed archive file (`/tmp/dumpfile.tar`) containing the dump of the `old_db` database.

- **Upload the dump file (optional):**
If you plan to import the dump file into the Kubernetes cluster from outside, you can upload it to a cloud storage service or leave it on your local machine for later transfer.

## Step 4: Import the dump using `pg_restore`

With the database dump file ready, you can import it into the new PostgreSQL database running in your Kubernetes cluster.

- **Get the dump file into the PostgreSQL client container:**
If you uploaded the dump file to a cloud storage service, you can use `wget` to download it into the container:

```sh
wget https://cloud-storage.com/dumpfile.tar
```

> *Use [this guide](https://chemicloud.com/blog/download-google-drive-files-using-wget) if you have to use `wget` to download the dump file from Google Drive.*

Alternatively, if the dump file is on your local machine, you can copy it directly into the container using `kubectl cp`:

```sh
kubectl cp /path/to/dumpfile.tar namespace/pod-name:/path/in/container
```

> Add the `--container=container-name` flag if there are multiple containers in the Pod to specify the container to copy into.

- **Import the database dump:**
Once the dump file is in the container, use the `pg_restore` command to import it into the new database:

> *If you are using the sample database provide above, note that database file is in zipformat ( dvdrental.zip) so you need to extract it to  dvdrental.tar before loading the sample database into the PostgreSQL database server. Use `unzip dvdrental.zip` to extract the `.tar` dump.*

```sh
pg_restore --host my-postgresql-postgresql -U databaseuser -d incoming_db -p 5432 dumpfile.tar
```

- **Verify the import:**
After the import is complete, you can verify that the data was imported successfully by connecting to the new database and checking the tables:

```sh
psql --host my-postgresql-postgresql -U databaseuser -d incoming_db -p 5432
\dt
```

## Troubleshooting

- **Connection issues:** If you encounter connection issues, double-check the PostgreSQL service name, port, username, and password. Also, ensure that the service is accessible from within the cluster.
- **Permissions issues:** If you encounter permission issues while importing the dump, make sure that the user you're using has the necessary privileges on the target database.
- **Resource limitations:** If the import process fails due to resource limitations (e.g., memory, CPU), you may need to adjust the resource requests and limits for the PostgreSQL StatefulSet or the client Pod.
