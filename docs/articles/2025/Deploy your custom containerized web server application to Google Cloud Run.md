---
layout: default
title: Deploy your custom containerized web server application to Google Cloud Run
parent: Contents
date: 2025-07-01
nav_exclude: true
---

# Deploy your custom containerized web server application to Google Cloud Run.
- TOC
{:toc}

-----

## 1\. Prerequisites 🧑‍💻

Before you begin, ensure you have the following set up:

  * **Google Cloud Project:**
      * A Google Cloud Project with **billing enabled**.
      * The following APIs **enabled** within your project:
          * Cloud Run API (`run.googleapis.com`)
          * Cloud Build API (`cloudbuild.googleapis.com`) (if using Cloud Build)
          * Artifact Registry API (`artifactregistry.googleapis.com`) OR Container Registry API (`containerregistry.googleapis.com`)
          * Secret Manager API (`secretmanager.googleapis.com`) (if managing secrets)
            You can enable APIs via the Google Cloud Console under "APIs & Services" \> "Library".
  * **`gcloud` Command-Line Tool:**
      * Installed and configured locally. If not, [install the Google Cloud SDK](https://cloud.google.com/sdk/docs/install).
      * Authenticated with your Google Cloud account:
        ```bash
        gcloud auth login
        gcloud config set project YOUR_PROJECT_ID
        ```
        Replace `YOUR_PROJECT_ID` with your actual project ID.
  * **Docker:**
      * Docker installed and running locally. Refer to the [Docker documentation](https://docs.docker.com/get-docker/) for installation instructions.
  * **`Dockerfile`:**
      * A `Dockerfile` must be present in the root directory of your application code. This file defines how to build your container image.
      * Ensure your application within the container listens on the port specified by the `$PORT` environment variable (Cloud Run injects this automatically, typically `8080`).
  * **Application Container Image:**
      * You need a container image of your web application.
      * **Option 1 (Recommended): Build with Cloud Build and store in Artifact Registry.** Cloud Build can automatically build your image from your `Dockerfile` and push it to Artifact Registry.
      * **Option 2: Build locally and push to Artifact Registry/Container Registry.** You can build the image on your local machine using Docker and then push it.

-----

## 2\. Containerization and Pushing to Registry 🐳

If your container image is not already built and pushed to a registry:

  * **Build the Docker Image:**
    Open a terminal in your application's root directory (where the `Dockerfile` is located).

    ```bash
    docker build -t YOUR_IMAGE_NAME .
    ```

    Replace `YOUR_IMAGE_NAME` with a name for your image (e.g., `my-web-app`).

  * **Tag the Docker Image:**
    Tag your image for Artifact Registry or Container Registry.

    **For Artifact Registry (Recommended):**

    ```bash
    docker tag YOUR_IMAGE_NAME YOUR_REGION-docker.pkg.dev/YOUR_PROJECT_ID/YOUR_REPOSITORY_NAME/YOUR_IMAGE_NAME:latest
    ```

    Example: `docker tag my-web-app us-central1-docker.pkg.dev/my-gcp-project/my-app-repo/my-web-app:latest`
    Replace `YOUR_REGION`, `YOUR_PROJECT_ID`, `YOUR_REPOSITORY_NAME`, and `YOUR_IMAGE_NAME` accordingly. Ensure the Artifact Registry repository exists or create one.

    **For Container Registry (Legacy):**

    ```bash
    docker tag YOUR_IMAGE_NAME gcr.io/YOUR_PROJECT_ID/YOUR_IMAGE_NAME:latest
    ```

    Example: `docker tag my-web-app gcr.io/my-gcp-project/my-web-app:latest`

  * **Authenticate Docker to Google Cloud:**
    Configure Docker to use `gcloud` as a credential helper for the chosen registry.

    **For Artifact Registry:**

    ```bash
    gcloud auth configure-docker YOUR_REGION-docker.pkg.dev
    ```

    Example: `gcloud auth configure-docker us-central1-docker.pkg.dev`

    **For Container Registry:**

    ```bash
    gcloud auth configure-docker gcr.io
    ```

  * **Push the Tagged Image:**

    **For Artifact Registry:**

    ```bash
    docker push YOUR_REGION-docker.pkg.dev/YOUR_PROJECT_ID/YOUR_REPOSITORY_NAME/YOUR_IMAGE_NAME:latest
    ```

    **For Container Registry:**

    ```bash
    docker push gcr.io/YOUR_PROJECT_ID/YOUR_IMAGE_NAME:latest
    ```

Alternatively, to build and submit directly using **Cloud Build** (pushes to Artifact Registry by default if configured, or Container Registry):

```bash
gcloud builds submit --tag YOUR_REGION-docker.pkg.dev/YOUR_PROJECT_ID/YOUR_REPOSITORY_NAME/YOUR_IMAGE_NAME:latest .
```

Or for Container Registry:

```bash
gcloud builds submit --tag gcr.io/YOUR_PROJECT_ID/YOUR_IMAGE_NAME:latest .
```

-----

## 3\. Deploying to Cloud Run 🚀

You can deploy your containerized application using the `gcloud` CLI or the Google Cloud Console.

### Using the `gcloud` Command-Line Tool

The primary command is `gcloud run deploy`.

```bash
gcloud run deploy YOUR_SERVICE_NAME \
    --image YOUR_REGION-docker.pkg.dev/YOUR_PROJECT_ID/YOUR_REPOSITORY_NAME/YOUR_IMAGE_NAME:latest \
    --platform managed \
    --region YOUR_CLOUD_RUN_REGION \
    --port CONTAINER_PORT \
    --cpu CPU_ALLOCATION \
    --memory MEMORY_LIMIT \
    --concurrency CONCURRENCY_VALUE \
    --min-instances MIN_INSTANCES \
    --max-instances MAX_INSTANCES \
    --set-env-vars KEY1=VALUE1,KEY2=VALUE2 \
    --allow-unauthenticated
```

**Key Configuration Options Explained:**

  * `YOUR_SERVICE_NAME`: Choose a name for your Cloud Run service (e.g., `my-web-service`).
  * `--image`: The full URL of your container image in Artifact Registry or Container Registry.
  * `--platform managed`: Specifies the fully managed Cloud Run platform.
  * `--region YOUR_CLOUD_RUN_REGION`: The Google Cloud region where you want to deploy your service (e.g., `us-central1`).
  * `--port CONTAINER_PORT`: (Optional) The port your container listens on. If not specified, Cloud Run defaults to `8080`. Your application should listen on the port defined by the `$PORT` environment variable. If your container listens on a different hardcoded port, specify it here.
  * `--cpu CPU_ALLOCATION`: (Optional) CPU to allocate per instance (e.g., `1`, `2`, `0.5`). Default is `1`.
  * `--memory MEMORY_LIMIT`: (Optional) Memory to allocate per instance (e.g., `512Mi`, `1Gi`). Default is `512Mi`.
  * `--concurrency CONCURRENCY_VALUE`: (Optional) Number of concurrent requests an instance can handle. Default is `80`.
  * `--min-instances MIN_INSTANCES`: (Optional) Minimum number of instances to keep running. Default is `0` (scales to zero). Set to `1` or higher for a warm instance.
  * `--max-instances MAX_INSTANCES`: (Optional) Maximum number of instances. Default is `100`.
  * `--set-env-vars KEY1=VALUE1,KEY2=VALUE2`: (Optional) Set environment variables for your service.
  * `--update-secrets=KEY1=SECRET_NAME:VERSION,KEY2=SECRET_NAME2:latest`: (Optional) Mount secrets from Secret Manager as environment variables. `SECRET_NAME` is the name of the secret in Secret Manager, and `VERSION` is the specific version or `latest`.
  * `--allow-unauthenticated`: (Optional) Allows public access to your service. For private services, omit this and configure IAM invoker permissions (`gcloud run services add-iam-policy-binding`).
  * `--ingress`: (Optional) Controls network traffic. Options: `all` (default), `internal`, `internal-and-cloud-load-balancing`.

**Example:**

```bash
gcloud run deploy my-public-api \
    --image us-central1-docker.pkg.dev/my-gcp-project/my-app-repo/my-web-app:latest \
    --platform managed \
    --region us-central1 \
    --allow-unauthenticated \
    --port 8080 \
    --memory 1Gi \
    --max-instances 5
```

During the first deployment, you'll be prompted for some of these options if not provided.

### Using the Google Cloud Console

1.  Go to the [Cloud Run page](https://console.cloud.google.com/run) in the Google Cloud Console.
2.  Click **"Create service"**.
3.  **Service Settings:**
      * **Container image URL:** Select your image from Artifact Registry or manually enter the image URL from Container Registry.
      * **Service name:** Enter a name for your service.
      * **Region:** Select the region for your service.
4.  **Authentication:**
      * Choose **"Allow unauthenticated invocations"** for a public web server.
      * Choose **"Require authentication"** for a private service.
5.  Expand **"Container, Connections, Security"** for more advanced settings:
      * **Container:**
          * **Container port:** Specify the port your container listens on (defaults to `8080`, respects `$PORT`).
          * **CPU allocation and memory:** Configure resources per instance.
          * **Capacity:**
              * **Concurrency:** Set the number of concurrent requests per instance.
              * **CPU is always allocated:** Choose if CPU should be always allocated or only during request processing (affects pricing).
          * **Environment variables:** Add environment variables.
          * **Secrets:** Reference secrets from Secret Manager. Click "Reference a secret", select the secret, version, and how it should be exposed (e.g., as an environment variable).
      * **Connections:**
          * **Networking:** Configure VPC connectors if needed.
          * **HTTP/2:** Enable or disable.
      * **Security:**
          * **Container service account:** Choose the identity the container runs with.
          * **Execution environment:** Choose the execution environment.
6.  Expand **"Autoscaling"**:
      * **Minimum number of instances:** Set to `0` to scale to zero (most cost-effective for idle services) or `1` or more to keep instances warm.
      * **Maximum number of instances:** Set the upper limit for scaling.
7.  Click **"Create"**.

### Setting up a Custom Domain

After deployment, you can map a custom domain to your Cloud Run service.

1.  Go to your service in the Cloud Run console.
2.  Select the **"Custom Domains"** tab (or "Triggers" then "Add custom domain" in some views).
3.  Click **"Add mapping"**.
4.  Follow the instructions to verify domain ownership and update your DNS records (usually `A`, `AAAA`, or `CNAME` records).
    This process involves your DNS provider. For more details, see the [official documentation on mapping custom domains](https://cloud.google.com/run/docs/mapping-custom-domains).

-----

## 4\. Verification and Testing ✅

  * **Find the Service URL:**

      * **`gcloud` CLI:** After successful deployment, the CLI will output the service URL. You can also retrieve it using:
        ```bash
        gcloud run services describe YOUR_SERVICE_NAME --platform managed --region YOUR_CLOUD_RUN_REGION --format='value(status.url)'
        ```
      * **Google Cloud Console:** Navigate to your service in the Cloud Run section. The URL is displayed at the top of the service details page.

  * **Test the Web Server:**

      * **Browser:** Open the service URL in a web browser.
      * **`curl`:** Use `curl` in your terminal:
        ```bash
        curl YOUR_SERVICE_URL
        ```
        Replace `YOUR_SERVICE_URL` with the actual URL.

  * **Check Logs in Cloud Logging:**
    Cloud Run services automatically stream logs to Cloud Logging.

      * **`gcloud` CLI:**
        ```bash
        gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=YOUR_SERVICE_NAME AND resource.labels.location=YOUR_CLOUD_RUN_REGION" --limit 50 --format "value(textPayload)"
        ```
      * **Google Cloud Console:**
        1.  Go to your service in the Cloud Run console.
        2.  Click the **"Logs"** tab.
        3.  You can filter logs by severity and search for specific terms.

  * **Common Deployment Issues and Basic Troubleshooting:**

      * **Image Pull Errors:**
          * Ensure the image URL is correct and the image exists in the registry.
          * Verify Cloud Run service agent (`service-YOUR_PROJECT_NUMBER@gcp-sa-run.iam.gserviceaccount.com`) has permissions to pull from the registry (usually "Artifact Registry Reader" or "Storage Object Viewer" for Container Registry). This is typically set up automatically.
      * **Application Crashes / Port Issues:**
          * Check container logs for error messages.
          * Ensure your application listens on the port specified by the `$PORT` environment variable (Cloud Run sets `$PORT` to `8080` by default if your `Dockerfile` doesn't `EXPOSE` a different port explicitly and you don't override `--port`).
          * If your application needs to write to the filesystem, it can only use in-memory `/tmp`. For persistent storage, use services like Cloud Storage, Firestore, or Cloud SQL.
      * **Permission Denied Errors:**
          * If accessing other Google Cloud services, ensure the runtime service account for your Cloud Run service has the necessary IAM permissions. By default, it uses the Compute Engine default service account. It's better to assign a dedicated service account with least privilege.
      * **Health Check Failures:**
          * Cloud Run sends requests to the container root path (`/`) to check its health. If your application doesn't respond successfully to `/` or takes too long to start, deployments might fail or instances might be restarted.
      * **Quotas:**
          * You might hit regional quotas for CPU, memory, or number of instances. Check "Quotas" in the IAM & Admin section of the console.

-----

## 5\. Pricing Estimation 💰

Cloud Run pricing is based on the actual resources your service consumes.

  * **Key Cloud Run Pricing Components:**

      * **vCPU-time:** Charged for the vCPU allocated to your instances, only while they are processing requests or if `min-instances` is set \> 0 (or CPU is always allocated). Measured in vCPU-seconds.
      * **Memory-time:** Charged for the memory allocated to your instances, under the same conditions as vCPU. Measured in GiB-seconds.
      * **Number of requests:** Charged per million requests.
      * **Outbound networking:** Data egress to the internet is charged per GiB. Egress to other Google Cloud services in the same region is often free or discounted.
      * **Build Time (if using Cloud Build):** Cloud Build has its own pricing based on build minutes.

  * **Always-Free Tier:**
    Cloud Run includes a generous **perpetual free tier** for vCPU-time, memory-time, requests, and outbound networking. This often covers small applications or development/testing workloads. Check the [official Cloud Run pricing page](https://cloud.google.com/run/pricing) for current free tier amounts as they can change.

  * **Google Cloud Pricing Calculator:**
    To estimate costs for your specific configuration and expected traffic:

    1.  Go to the [Google Cloud Pricing Calculator](https://cloud.google.com/products/calculator).
    2.  Search for and select "Cloud Run".
    3.  Enter your anticipated usage:
          * Region
          * Memory and vCPU per instance
          * Number of requests per month
          * Average request duration (or concurrency and CPU utilization)
          * Billable instance time (consider min instances)
          * Outbound internet traffic

  * **Tips for Cost Optimization:**

      * **Scale to Zero:** Set `min-instances` to `0` (default) if your application can tolerate cold starts. This means you pay nothing when there are no requests.
      * **Right-size Instances:** Allocate only the necessary CPU and memory. Over-provisioning increases costs. Monitor resource utilization to fine-tune.
      * **Optimize Concurrency:** Adjust concurrency settings. Higher concurrency can mean fewer instances but might impact request latency if instances are overloaded.
      * **CPU Allocation:** For workloads that are idle between requests, choose "CPU is only allocated during request processing" (default). If your service performs background work without requests, you might need "CPU is always allocated", which costs more.
      * **Region Selection:** Costs can vary slightly by region.
      * **Caching:** Use caching (e.g., Cloud CDN, in-application caching) to reduce the number of requests hitting your Cloud Run service.
