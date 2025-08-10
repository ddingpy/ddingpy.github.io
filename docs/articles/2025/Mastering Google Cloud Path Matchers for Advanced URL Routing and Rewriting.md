---
layout: default
parent: Contents
date: 2025-07-09
nav_exclude: true
---

## Mastering Google Cloud Path Matchers for Advanced URL Routing and Rewriting
- TOC
{:toc}

In the landscape of Google Cloud Load Balancing, **Path Matcher** stands as a pivotal component for sophisticated traffic management. It allows you to direct incoming requests to different backend services or buckets based on the path of the request URL. This guide provides a comprehensive walkthrough of using Path Matchers, with a special focus on its application for routing based on URL query parameters and rewriting URL paths.

### 1. Introduction to Path Matchers in GCP

A **Path Matcher** is a configuration element within a URL map that defines the routing logic based on the URL's path. When a request reaches your Google Cloud Load Balancer, the URL map first uses host rules to determine which path matcher to use. The selected path matcher then examines the request's path to route it to the appropriate backend.

**Primary Use Cases:**

* **Service-Based Routing:** Directing traffic to different microservices based on the URL path (e.g., `/users/*` to the user-service, `/products/*` to the product-service).
* **Content-Based Routing:** Serving different types of content from different backends (e.g., `/static/*` to a Cloud Storage bucket, `/dynamic/*` to a set of virtual machines).
* **A/B Testing & Canary Deployments:** Routing a percentage of traffic or specific users (identified by headers or query parameters) to a new version of a service.
* **URL Rewriting:** Modifying the URL path before it's sent to the backend, enabling cleaner user-facing URLs while maintaining a different internal URL structure.

### 2. Path Matcher Syntax for URL Rewriting

You can define Path Matcher rules using either YAML files or `gcloud` command-line tool. The key to advanced routing and rewriting lies within the `routeRules` of a path matcher, which offer more granular control than simple `pathRules`.

A `routeRule` consists of:
* `priority`: Rules are evaluated in order of priority, from lowest to highest number.
* `matchRules`: Conditions that a request must meet for the rule to be applied. This can include matching on the path, headers, and **query parameters**.
* `routeAction`: The action to take if the request matches, such as forwarding to a backend service or performing a URL rewrite.

#### URL Rewrite Syntax

The `urlRewrite` section within a `routeAction` is where you define how the URL should be modified. Its primary fields are:

* `pathPrefixRewrite`: Replaces the matched path prefix with a new value.
* `hostRewrite`: Replaces the host of the request.

Here's a snippet of a YAML configuration for a URL map with a path matcher and a route rule:

```yaml
name: my-url-map
defaultService: projects/your-project/global/backendServices/default-service
hostRules:
- hosts:
  - '*'
  pathMatcher: my-path-matcher
pathMatchers:
- name: my-path-matcher
  defaultService: projects/your-project/global/backendServices/default-service
  routeRules:
  - priority: 1
    matchRules:
    - prefixMatch: /old-path/
      queryParameterMatches:
      - name: version
        exactMatch: 'v2'
    routeAction:
      urlRewrite:
        pathPrefixRewrite: /new-path/
      service: projects/your-project/global/backendServices/new-service

```

#### Manipulating URL Query Parameters

A crucial point to understand is that Google Cloud Load Balancer's `urlRewrite` feature **natively supports rewriting the path and host, but not directly adding, removing, or modifying query parameters** that are sent to the backend.

However, you can use `queryParameterMatches` within `matchRules` to route traffic based on the presence or value of query parameters. The backend service will then receive the original, unmodified query parameters from the client's request.

### 3. Practical Examples of URL Query-Based Routing and Path Rewriting

Here are three real-world scenarios demonstrating the use of Path Matchers.

#### Example 1: A/B Testing with Query Parameters

* **Scenario:** You want to route users with the query parameter `experiment=new-feature` to a new version of your service (`new-feature-service`), while all other users go to the stable version (`stable-service`).
* **'Before' URL (from client):** `https://example.com/search?q=gcp&experiment=new-feature`
* **'After' URL (to backend):** The path and query parameters are forwarded as is. The routing decision is based on the query parameter.
* **gcloud Configuration:**

```bash
# Create a new route rule for the A/B test
gcloud compute url-maps add-path-matcher my-url-map \
    --path-matcher-name=my-path-matcher \
    --default-service=projects/your-project/global/backendServices/stable-service \
    --new-hosts=example.com

# Add a route rule to match the query parameter
gcloud compute url-maps add-route-rule my-url-map \
    --path-matcher-name=my-path-matcher \
    --service=projects/your-project/global/backendServices/new-feature-service \
    --route-rule-priority=10 \
    --match-rule-query-param-matches="experiment=new-feature"
```

#### Example 2: Redirecting Legacy API Paths with Query-Based Routing

* **Scenario:** You've refactored your API. The old endpoint `/api/v1/data` with a `type=legacy` query parameter should now be served by a new backend, and the path should be rewritten to `/api/v2/data`.
* **'Before' URL (from client):** `https://api.example.com/api/v1/data?type=legacy`
* **'After' URL (to backend):** `https://api.example.com/api/v2/data?type=legacy`
* **YAML Configuration:**

```yaml
- name: my-path-matcher
  defaultService: projects/your-project/global/backendServices/default-api-service
  routeRules:
  - priority: 100
    matchRules:
    - prefixMatch: /api/v1/data
      queryParameterMatches:
      - name: type
        exactMatch: 'legacy'
    routeAction:
      urlRewrite:
        pathPrefixRewrite: /api/v2/data
      service: projects/your-project/global/backendServices/legacy-api-service
```

#### Example 3: Routing to a Specific Backend for a Marketing Campaign

* **Scenario:** A marketing campaign uses URLs with the tracking parameter `utm_campaign=summer_sale`. All traffic with this parameter should be directed to a specialized backend service that can handle campaign tracking.
* **'Before' URL (from client):** `https://www.example.com/products/on-sale?utm_campaign=summer_sale`
* **'After' URL (to backend):** The path and query string are passed through to the campaign backend.
* **gcloud Configuration:**

```bash
gcloud compute url-maps add-route-rule my-url-map \
    --path-matcher-name=my-path-matcher \
    --service=projects/your-project/global/backendServices/campaign-service \
    --route-rule-priority=50 \
    --match-rule-query-param-matches="utm_campaign=summer_sale"
```

### 4. Best Practices and Considerations

* **Organize with Priorities:** Use the `priority` field in `routeRules` to create a clear and predictable order of execution. Lower numbers have higher precedence.
* **Descriptive Naming:** Use clear and descriptive names for your URL maps, path matchers, and backend services to make your configuration easier to understand and manage.
* **Default Service as a Fallback:** Always configure a default service for your path matcher. This ensures that any traffic that doesn't match a specific rule is still handled gracefully.
* **Limitations on Query Rewriting:** Be aware that you can't directly manipulate the query string of the request that is sent to the backend. If you need to add, remove, or change query parameters, this logic must be handled within your backend application or by using a solution like a proxy in front of your backend.
* **Testing and Validation:**
    * **`gcloud compute url-maps validate`:** Before applying a new URL map configuration, use this command to check for syntax errors and basic logical issues.
    ```bash
    gcloud compute url-maps validate --source /path/to/your/url-map.yaml
    ```
    * **Staging Environments:** Test complex routing and rewrite rules in a non-production environment that mirrors your production setup as closely as possible.
    * **Logging and Monitoring:** Enable logging for your load balancer. The logs will show the original request and how it was handled, which is invaluable for debugging your path matcher rules. Check the `jsonPayload.statusDetails` field in the logs for information about which rule was matched.