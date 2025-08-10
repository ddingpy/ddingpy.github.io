---
layout: default
parent: Contents
date: 2025-07-08
nav_exclude: true
---
# How to Configure a Global External HTTPS Load Balancer for Google Cloud Run
- TOC
{:toc}

Of course. Here's a detailed, step-by-step guide on how to configure a Global External HTTPS Load Balancer for a serverless application running on Google Cloud Run.

### Introduction

Using a Global External HTTPS Load Balancer in front of your Cloud Run service provides several key benefits for production applications. It allows you to use a **custom domain** with a **Google-managed SSL certificate**, which Google automatically provisions and renews for you. This setup also integrates seamlessly with **Cloud CDN** to cache content closer to your users, improving latency. Additionally, it enhances security by enabling services like **Google Cloud Armor** to protect your application from DDoS attacks and other web-based threats.

***

### Prerequisites

* A deployed Cloud Run service. This guide will assume the service is named `my-cloud-run-service` and is located in the `asia-northeast1` region.
* The Google Cloud SDK (`gcloud`) installed and authenticated on your local machine.
* A registered domain name that you own (e.g., `your-cool-domain.com`).

***

### Step-by-Step Configuration

This section provides instructions using both the `gcloud` command-line tool and the Google Cloud Console.

#### **Step 1: Reserve a Global Static IP Address**

A static IP address is essential because it provides a stable, unchanging endpoint for your domain's 'A' record. Unlike an ephemeral IP, it won't change if your load balancer is updated.

**gcloud Command:**
```bash
gcloud compute addresses create my-lb-ip \
    --global
```

**Google Cloud Console:**
1.  Navigate to **VPC network** > **IP addresses**.
2.  Click **RESERVE EXTERNAL STATIC IP ADDRESS**.
3.  **Name**: `my-lb-ip`
4.  **Network Service Tier**: Premium
5.  **IP version**: IPv4
6.  **Type**: Global
7.  Click **RESERVE**. Note the IP address that is provisioned.

---

#### **Step 2: Create a Serverless Network Endpoint Group (NEG)**

A Serverless NEG acts as a backend for the load balancer. It's a special type of NEG that points to a serverless service like Cloud Run, allowing the load balancer to route traffic to it.

**gcloud Command:**
```bash
gcloud compute network-endpoint-groups create my-serverless-neg \
    --region=asia-northeast1 \
    --network-endpoint-type=serverless \
    --cloud-run-service=my-cloud-run-service
```

**Google Cloud Console:**
1.  Navigate to **Network services** > **Load balancing** and click on **Backends**.
2.  Select the **NETWORK ENDPOINT GROUPS** tab and click **CREATE NETWORK ENDPOINT GROUP**.
3.  **Name**: `my-serverless-neg`
4.  **Network endpoint group type**: Serverless network endpoint group
5.  **Region**: `asia-northeast1`
6.  Under **Serverless source**, select **Cloud Run**, and then choose `my-cloud-run-service` as the service name.
7.  Click **CREATE**.

---

#### **Step 3: Create a Google-Managed SSL Certificate**

This certificate will secure your custom domain with HTTPS. Google handles the entire lifecycle of this certificate, including provisioning and renewal, at no extra cost.

**gcloud Command:**
```bash
gcloud compute ssl-certificates create my-ssl-cert \
    --domains=[YOUR_DOMAIN] \
    --global
```
*Replace `[YOUR_DOMAIN]` with your domain name (e.g., `www.your-cool-domain.com`).*

**Google Cloud Console:**
1.  Navigate to **Network services** > **Load balancing** and click on **Certificates**.
2.  Click **CREATE SSL CERTIFICATE**.
3.  **Name**: `my-ssl-cert`
4.  **Create mode**: Create Google-managed certificate
5.  **Domains**: Enter your domain name (e.g., `www.your-cool-domain.com`).
6.  Click **CREATE**.

The certificate status will be `PROVISIONING` until you point your domain's DNS records to the load balancer's IP.

---

#### **Step 4: Configure the Backend Service**

The backend service is the component that directs traffic from the load balancer to the appropriate backend, in this case, your Serverless NEG. Health checks are not necessary because Cloud Run's native infrastructure already manages service health.

**gcloud Command:**
```bash
# Create the backend service
gcloud compute backend-services create my-backend-service \
    --global

# Add the Serverless NEG as the backend
gcloud compute backend-services add-backend my-backend-service \
    --global \
    --network-endpoint-group=my-serverless-neg \
    --network-endpoint-group-region=asia-northeast1

# (Optional) Enable Cloud CDN
gcloud compute backend-services update my-backend-service \
    --enable-cdn \
    --global
```

**Google Cloud Console:**
1.  Navigate to **Network services** > **Load balancing** and click on **Backends**.
2.  Click **CREATE A BACKEND SERVICE**.
3.  **Name**: `my-backend-service`
4.  **Backend type**: Serverless network endpoint group
5.  Under **Backends**, click **New backend**.
6.  Select the Serverless NEG you created (`my-serverless-neg`).
7.  **(Optional)** Check the box for **Enable Cloud CDN**.
8.  Leave the rest of the settings as default and click **CREATE**.

---

#### **Step 5: Configure the Frontend (Forwarding Rule)**

The frontend consists of the forwarding rule and the target proxy. The forwarding rule binds your reserved IP address and a specific port to a target proxy, which then uses a URL map to direct requests to the correct backend service.

**gcloud Commands:**
```bash
# Create the URL map
gcloud compute url-maps create my-url-map \
    --default-service my-backend-service

# Create the HTTPS target proxy
gcloud compute target-https-proxies create my-https-proxy \
    --url-map=my-url-map \
    --ssl-certificates=my-ssl-cert

# Create the global forwarding rule for HTTPS (port 443)
gcloud compute forwarding-rules create my-https-forwarding-rule \
    --address=my-lb-ip \
    --target-https-proxy=my-https-proxy \
    --global \
    --ports=443
```

**Google Cloud Console:**
1.  Navigate to **Network services** > **Load balancing** and click **CREATE LOAD BALANCER**.
2.  Under **Application Load Balancer (HTTPS/2)**, click **START CONFIGURATION**.
3.  Select **From Internet to my VMs or serverless services** and **Global external Application Load Balancer**. Click **CONTINUE**.
4.  **Name**: `my-cloud-run-lb`
5.  **Frontend configuration**:
    * **Protocol**: HTTPS (includes HTTP/2)
    * **IP address**: Select `my-lb-ip`.
    * **Certificate**: Select `my-ssl-cert`.
    * Check **Enable HTTP to HTTPS Redirect**. This will automatically create the necessary HTTP components and rules.
    * Click **DONE**.
6.  **Backend configuration**:
    * Click **Backend services & backend buckets** and select **Backend services**.
    * Select the backend service you created (`my-backend-service`).
7.  **Review and finalize**:
    * Review the configuration and click **CREATE**.

---

#### **Step 6: Update DNS Records**

The final step is to tell the internet where to find your application by pointing your custom domain to the load balancer's static IP address.

1.  Find the static IP address you reserved in **Step 1**.
2.  Go to your domain registrar's website (e.g., Google Domains, Namecheap, GoDaddy).
3.  Navigate to the DNS management page for your domain.
4.  Create a new **'A' record** with the following details:
    * **Type**: `A`
    * **Host/Name**: `@` (for the root domain) or `www` (if you configured the certificate for a subdomain).
    * **Value/Points to**: The reserved static IP address of your load balancer.
    * **TTL (Time to Live)**: You can leave this at the default setting (often 1 hour).

***

### Testing and Verification

After you've updated your DNS records, you need to wait for the changes to propagate across the internet. This can take anywhere from a few minutes to 48 hours, though it's typically on the faster end.

Once propagation is complete, you can test your setup:

1.  Open a web browser and navigate to `https://[YOUR_DOMAIN]`.
2.  You should see your Cloud Run application's content served securely over HTTPS.
3.  Check the certificate information in your browser; it should show the Google-managed certificate you created.
4.  Try navigating to `http://[YOUR_DOMAIN]`; you should be automatically redirected to the HTTPS version.

***

### Diagram

Here is a simple diagram illustrating the final architecture of your serverless application with a global load balancer.

```
      +------+
      | User |
      +--+---+
         |
         | (https://your-cool-domain.com)
         v
    +----------+
    |   DNS    |
    +----+-----+
         |
         | (Points to Load Balancer IP)
         v
+--------+--------------------------+
|  Global Load Balancer             |
|  (Forwarding Rule - Static IP)    |
|  (Target Proxy + SSL Cert)        |
+--------+--------------------------+
         |
         v
+--------+--------------------------+
|  Backend Service                  |
|  (with optional Cloud CDN)        |
+--------+--------------------------+
         |
         v
+--------+--------------------------+
|  Serverless NEG                   |
+--------+--------------------------+
         |
         v
+--------+--------------------------+
|  Cloud Run Service                |
|  (my-cloud-run-service)           |
+-----------------------------------+
```