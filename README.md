
Deployment guide for a Guacamole server on OpenShift.

---

## What is Guacamole Server?

Guacamole is a web application that is made up of many parts. Users connect to a Guacamole server with their web browser. The Guacamole client is the web server within the Guacamole server. Once the webserver is loaded, this client connects back to the server over HTTP using the Guacamole protocol.

## What is Guacamole Protocol?

Guacamole protocol is basically the HTTP requests the client is making and sending back to the server — the requests represent the actions we want to do on the remote desktop: every mouse click/movement, keyboard input, and so on.

The webserver sends these actions to the Guacamole server, which reads these Guacamole protocol actions and forwards them to Guacd. Guacd is the proxy that interprets the Guacamole protocol into actual actions on various remote desktop connections such as RDP / VNC.

### Why OpenShift?

OpenShift is our cloud services platform — for managing containers, VMs, etc.

In order to enable access to our simulation machines regardless of network firewall settings (some don't allow Remote Desktop Connections), we want to have a Guacamole server up and running in our container platform.

> **Note:** This project is a learning project — it will walk you through the application troubleshooting process. That means there will be issues during this deployment — you have the choice of looking at the solution or trying to figure your way out of the error.
>
> **Important:** This project is based on RedHat cloud tools such as Quay (for the image registry) and OpenShift (for the container platform), plus some Docker commands.
>
> This guide is based mainly on the KRDesigns blog, which explains how to deploy Guacamole with Docker, plus some more tips from various DevOps folks who wrote about implementing this on the OpenShift platform. This is the final guide that will hopefully walk you through every step.

---

## 1. Deploying the Database for Guacamole

In order to save machine usernames and passwords, we need a database. The chosen DB (the one mostly used for Guacamole) is MariaDB. We use the RHEL image so it can run in OpenShift (other versions run as root, and therefore can't run in OpenShift).

### Get the init script

Deploying the DB first requires running the Guacamole image locally and retrieving the `initdb.sql` file. `initdb.sql` is an automation script that creates the Guacamole DB with the relevant tables.

We should already have the Guacamole image locally from before, so run it and retrieve the file:

```bash
docker run --rm guacamole/guacamole:1.4.0 /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
```

After running this command there should be no Docker container running, since the `--rm` flag removes it.

We then need to copy this file into the DB container (see below).

### Deploy the DB in OpenShift

In OpenShift → **Developer** → **Add** → **Container Image**, fill in these fields:

- **Image path** — from Quay
- **Resource limits** (scroll down to add manually):
  - CPU: request `512` / limit `2048`
  - Memory: request `1 Gi` / limit `1 Gi`
- **Deployment env vars** (scroll down to add manually):
  - `MYSQL_ROOT_PASSWORD`: `MariaDBRootPass`
  - `MYSQL_DATABASE`: `guacamole_db`
  - `MYSQL_USER`: `guacamole_user`
  - `MYSQL_PASSWORD`: `MariaDBUserPass`

Click **Create**, then switch to the **Administrator** view and check **Deployments → mariadb** inside the pods — you should see an image error at this point. To fix it, apply the two post-deploy steps below.

### Post-deploy: create storage and load the init script

Once MariaDB is running with no image errors:

1. **Create a volume for the container** (a volume is storage). Switch to **Administrator** → **Storage** → **PersistentVolumeClaim** → **Create**. Choose any storage class, set size to `2 Gi`, and name the PVC `db-data`.
2. **Mount it to the deployment.** In the MariaDB deployment, under **Actions** (top right), click **Add storage**, choose the PVC you just created, and mount it to `/var/lib/mysql`.
3. **Copy `initdb.sql` into the container** using `kubectl` (should already be installed) from your local machine. First, in OpenShift, find the **login command** and log in via the CLI. Then run:

   ```bash
   kubectl cp "<path_to_local_initdb.sql>" "<your_namespace>/<your_pod_name>:<mounted_directory>" -c "<your_container_name>"
   ```

   Example:

   ```bash
   kubectl cp "Downloads\guacamole\renana_is_the_queen\initdb.sql" mafat-world-server/mariadb-102-rhel7-649949db88-87c46:/var/lib/mysql -c mariadb-102-rhel7
   ```

   Check via the pod terminal (using `cd`, `ls`, etc.) that the copy worked.

4. **Start the DB.** Inside the container, navigate to the mounted path and run:

   ```bash
   cat ./initdb.sql | mysql -u root guacamole_db
   ```

   This signs in to the database and runs the `initdb.sql` script, which creates the DB tables.

That's it — the database deployment is done.

---

## 2. Deploying the Guacamole Web App Server

Now let's deploy the Guacamole web app. We need to build a new image, so grab the `start.sh` file and the `Dockerfile` and build it locally.

> **Important:** the `Dockerfile` and `start.sh` script must be in the same folder.

```bash
docker build -t <give_it_a_name> .
```

After building the image, tag it and push it to Quay (see the Quay/tagging guide, if you have one).

### Deploy in OpenShift

In OpenShift → **Developer** → **Add** → **Container Image**, fill in these fields:

0. Make sure to name your image properly.
1. **Image path** — from Quay
2. **Resource limits** (scroll down to add manually):
   - CPU: request `512` / limit `2048`
   - Memory: request `1 Gi` / limit `1 Gi`
3. **Deployment env vars** (scroll down to add manually):
   - `GUACD_HOSTNAME`: `guacd`
   - `MYSQL_HOSTNAME`: `mariadb-102-rhel7`
   - `MYSQL_DATABASE`: `guacamole_db`
   - `MYSQL_USER`: `guacamole_user`
   - `MYSQL_PASSWORD`: `MariaDBUserPass`
   - `TOTP_ENABLED`: `true`
   - `GUACD_PORT`: `4822`

> **Important:** whenever an environment variable is named `xxx_HOSTNAME`, it refers to another container (such as the DB or Guacd). It must match the actual name of your `guacd` and DB containers.

Click **Deploy**, and make sure to complete the two post-deploy image steps described in the [Image Pull Secret](#3-openshift-application-deployment-using-a-container-image) section below.

After the deployment finishes, edit it and add a command to the deployment YAML (under `spec` → `containers`) that runs `start.sh` inside the Guacamole container:

```yaml
command:
  - /bin/sh
  - "-c"
  - /opt/guacamole/bin/start.sh
```

> Keep the indentation (tabs/spaces) consistent!

That's it — the Guacamole web app should be up and running. To check it, see [Using Guacamole](#4-using-the-guacamole-web-app) below.

---

## 3. Deploying the Guacamole Proxy Server (Guacd)

This deployment is the easiest one. After deploying MariaDB, deploy Guacd in a very similar way.

Under **Developer** → **Add** → **Container Image**: use the same resource settings, but choose the `guacd` image this time. Add an environment variable:

- `HOME`: `/tmp`

Make sure to complete the two post-deploy image steps below. After that, Guacd should run without issues.

---

## 3.1 OpenShift Application Deployment Using a Container Image

This section covers pulling images from your Quay repo and running them in OpenShift.

Steps:

1. Load images from the zip file
2. Push images to Quay
3. Pull images from Quay into OpenShift
4. Run (deploy) the images

### Load, push, and pull the images

When opening a project in OpenShift, also open a repository designated for that project — this is where the container images for the project will be stored.

Guacamole needs three images:

- **mariadb** — stores user authentication for the machines
- **guacamole/guacamole** — the webserver
- **guacamole/guacd** — the Guacamole proxy, creates the connections

Start by grabbing the project images (e.g. from an `images.zip` folder). Load all three images to your local Docker Desktop:

```bash
docker load < image.tar
```

> You don't need to load `mariadb` this way — it's already available in Quay as `rhscl-mariadb-102-rhel7`. We use the RHEL image because it can run on OpenShift (it belongs to RHEL/Red Hat), while the standard image is built to run as root, which OpenShift does not allow.

After loading the images locally, retag them in the Quay/CTS format and push them:

```bash
docker tag <current_image> <registry.marganit-1.idf.cts>/<YOUR_REGISTRY>/<NEW_NAME>
docker push <image>
```

> Note: you may need to log in first via `docker login` (credentials available in Quay).

Verify all your images now appear in your Quay registry.

### Create an Image Pull Secret

After creating a repo and storing the images inside, in order to pull an image into an OpenShift project, we need an **Image Pull Secret** — the credential OpenShift uses to pull images from a private repo in Quay.

**How to create it:**

1. **Quay:** `<your_repo>` → **Robot Accounts** → **pull_all** → **Kubernetes Secret** → **Download**.
   (Inside the robot account there should be two secrets — choose the **pull** one.)
2. Copy the contents of the downloaded file.
3. **OpenShift:** **Administrator** → **Workloads** → **Secrets** → **Create From YAML** → paste and save.

Now you have an image pull secret ready to pull any image from your Quay registry.

Two more image-related steps remain, to be applied **after** creating each deployment:

1. Attach the image pull secret to every deployment you create.
2. Update the deployment's ImageStream.

### Deployment

In OpenShift, choose **Developer** → **Add** → **Container Image**, and provide the image path to pull from. Repeat this for each image, in this order:

**Database → Guacd → Guacamole**

### Post-deploy (image error fix)

If your pod isn't running due to an image error, apply these two remaining steps:

1. **Attach the pull secret.** In the deployment YAML, add your image pull secret under `spec`, above `containers`:

   ```yaml
   spec:
     imagePullSecrets:
       - name: <your-image-pull-secret-name>
   ```

2. **Fix the ImageStream.** OpenShift automatically creates an ImageStream when you start a deployment. Find it under **Builds** → navigate to the ImageStream for your deployment, then in its YAML go to `spec` → `tags` → `referencePolicy` → `type`, and change it from `Local` to `Source`.

After this, there should be no more image-related errors.

---

## 4. Using the Guacamole Web App

First, find the application's URL. In OpenShift, under **Networking** → **Routes**, you should see a new route named `guacamole`. Click the URL and add the extension `/guacamole`.

You should see a screen like this:

![Guacamole login screen](guide-images/slide8-app-screen.jpeg)

---

## Appendix: Additional Screenshots

Reference screenshots from the original deck (OpenShift deploy-image forms, storage/PVC setup, and related configuration screens):

![Deploy image / resource & env config](guide-images/slide9-1.jpeg)
![Deploy image form](guide-images/slide9-2.jpeg)
![Deployment configuration](guide-images/slide9-3.jpeg)
![Deployment configuration](guide-images/slide9-4.jpeg)
![Deployment configuration](guide-images/slide9-5.jpeg)
![Deployment configuration](guide-images/slide9-6.jpeg)
![OpenShift configuration](guide-images/slide10-1.jpeg)
![OpenShift configuration](guide-images/slide10-2.jpeg)
![OpenShift configuration](guide-images/slide10-3.jpeg)
![OpenShift configuration](guide-images/slide10-4.jpeg)
