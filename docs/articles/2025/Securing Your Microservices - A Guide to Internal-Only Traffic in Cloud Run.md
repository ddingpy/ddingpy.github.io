---
layout: default
parent: Contents
date: 2025-07-07
nav_exclude: true
---

# Securing Your Microservices: A Guide to Internal-Only Traffic in Cloud Run
- TOC
{:toc}

### Securing Your Microservices: A Guide to Internal-Only Traffic in Cloud Run

You can configure your Google Cloud Run services to communicate with each other internally, shielding them from the public internet and creating a more secure microservices architecture. This is achieved by setting the ingress of the receiving service to "internal" and ensuring the calling service is properly configured to send internal requests.

This guide will walk you through the essential steps to establish this internal communication, covering both network configuration and authentication.

---

### Restricting Ingress Traffic

The first step is to lock down your "receiving" Cloud Run service to only accept traffic from internal Google Cloud sources.

**To configure ingress settings:**

1.  Navigate to the Cloud Run section of the Google Cloud Console.
2.  Select the service you want to make internal.
3.  Click on **Edit & Deploy New Revision**.
4.  Under the "Networking" tab, locate the "Ingress" settings.
5.  Choose **Allow internal traffic only**.
6.  Click **Deploy**.

This setting ensures that your service is no longer publicly accessible and can only be reached from within your Google Cloud project.

---

### Enabling Service-to-Service Communication

For one Cloud Run service to call another internal service, the calling service needs to be able to route its outbound traffic through a Virtual Private Cloud (VPC) network.

#### **Configure VPC Egress**

You need to configure the "calling" service to send traffic to the VPC network. This is done using a Serverless VPC Access connector.

1.  **Create a Serverless VPC Access connector:**
    * In the Google Cloud Console, go to "Serverless VPC Access".
    * Click **Create connector**.
    * Provide a name, select the region (it must be the same as your Cloud Run services), and choose a VPC network and subnet.
    * Define the IP range for the connector.
    * Click **Create**.

2.  **Connect your Cloud Run service to the VPC connector:**
    * Go to your "calling" Cloud Run service in the console.
    * Click **Edit & Deploy New Revision**.
    * Under the "Networking" tab, in the "Egress" section for "VPC", select the VPC Access connector you just created.
    * For "VPC Egress," you can select either:
        * **Route only requests to private IPs to the VPC:** This is a good option if your service also needs to access public APIs.
        * **Route all traffic to the VPC:** This forces all outbound traffic from the service through your VPC.
    * Click **Deploy**.

---

### Authenticating Between Services

Even with internal networking, it's crucial to ensure that only authorized services can invoke each other. This is accomplished using Identity and Access Management (IAM) and service accounts.

#### **1. Create a Dedicated Service Account**

It's a best practice to create a specific service account for your "calling" service to give it a distinct identity.

1.  In the Google Cloud Console, go to "IAM & Admin" > "Service Accounts".
2.  Click **Create Service Account**.
3.  Give the service account a name and description.
4.  Click **Create and Continue**. You can skip granting this service account access to the project for now.
5.  Click **Done**.

#### **2. Assign the "Cloud Run Invoker" Role**

Now, you'll grant the service account of the "calling" service permission to invoke the "receiving" service.

1.  Go to the "receiving" Cloud Run service in the console.
2.  Click on the "Permissions" tab.
3.  Click **Add Principal**.
4.  In the "New principals" field, enter the email address of the service account you created for the "calling" service.
5.  In the "Select a role" dropdown, choose **Cloud Run Invoker**.
6.  Click **Save**.

#### **3. Associate the Service Account with the Calling Service**

Your "calling" service needs to use the identity of the service account you created.

1.  Go to your "calling" Cloud Run service in the console.
2.  Click **Edit & Deploy New Revision**.
3.  Under the "Security" tab, in the "Service account" dropdown, select the service account you created.
4.  Click **Deploy**.

#### **4. Obtaining and Using an ID Token**

When the "calling" service makes a request to the "receiving" service, it must include an ID token in the `Authorization` header. Google Cloud's client libraries can handle this for you automatically.

Here's a Python example of how to make an authenticated request:

```python
import os
import google.auth.transport.requests
import google.oauth2.id_token
import requests

def make_internal_request():
    # The URL of the receiving Cloud Run service
    receiving_service_url = os.environ.get("RECEIVING_SERVICE_URL")

    # Acquire an ID token
    auth_req = google.auth.transport.requests.Request()
    id_token = google.oauth2.id_token.fetch_id_token(auth_req, receiving_service_url)

    # Include the ID token in the Authorization header
    headers = {
        "Authorization": f"Bearer {id_token}"
    }

    try:
        response = requests.get(receiving_service_url, headers=headers)
        response.raise_for_status()  # Raise an exception for bad status codes
        return response.text
    except requests.exceptions.RequestException as e:
        print(f"Error making internal request: {e}")
        return None

if __name__ == "__main__":
    # Ensure the environment variable is set with the URL of your internal service
    # e.g., https://your-internal-service-name-xxxx.a.run.app
    response_data = make_internal_request()
    if response_data:
        print(f"Response from internal service: {response_data}")

```

By following these steps, you'll have a secure and internal communication channel between your Cloud Run services, leveraging the robust security features of Google Cloud.