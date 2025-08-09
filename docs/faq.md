---
layout: default
title: FAQ
nav_order: 5
---

# Frequently Asked Questions
{: .no_toc }

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## General Questions

### What is this platform?

Our platform is a comprehensive solution for [your use case]. It provides tools and APIs to help you build, deploy, and manage applications efficiently.

### How much does it cost?

We offer several pricing tiers:

| Plan | Price/Month | Features |
|:-----|:------------|:---------|
| **Free** | $0 | Basic features, 1,000 API calls |
| **Pro** | $29 | Advanced features, 10,000 API calls |
| **Business** | $99 | All features, 100,000 API calls |
| **Enterprise** | Custom | Unlimited usage, dedicated support |

### Is there a free trial?

Yes! All new accounts get a 14-day free trial of the Pro plan. No credit card required.

### Can I change plans later?

Absolutely. You can upgrade or downgrade your plan at any time from your account settings. Changes take effect immediately, and billing is prorated.

## Technical Questions

### What programming languages are supported?

Our platform supports:
- JavaScript/TypeScript
- Python
- Java
- Ruby
- Go
- PHP
- C#/.NET
- And more via REST API

### Can I self-host?

Yes, we offer self-hosted options for Business and Enterprise plans. Contact our sales team for more information.

### What are the system requirements?

**Minimum Requirements:**
- 2GB RAM
- 1GB disk space
- Node.js 14+ or Python 3.8+

**Recommended:**
- 4GB RAM
- 5GB disk space
- Latest LTS version of Node.js or Python

### Is there an API?

Yes! We provide a comprehensive REST API. See our [API Reference](/api/) for complete documentation.

## Account & Billing

### How do I reset my password?

1. Go to the [login page](https://app.example.com/login)
2. Click "Forgot password?"
3. Enter your email address
4. Check your email for reset instructions
5. Follow the link to create a new password

### Can I have multiple accounts?

Yes, you can create multiple accounts with different email addresses. For team collaboration, we recommend using our Organizations feature instead.

### What payment methods do you accept?

We accept:
- Credit cards (Visa, MasterCard, American Express)
- Debit cards
- PayPal
- Wire transfers (Enterprise only)

### How do I cancel my subscription?

To cancel your subscription:
1. Log in to your account
2. Go to Settings → Billing
3. Click "Cancel Subscription"
4. Follow the confirmation steps

Your account will remain active until the end of the current billing period.

## Security & Privacy

### Is my data secure?

Yes, we take security seriously:
- All data is encrypted at rest and in transit
- We use industry-standard AES-256 encryption
- Regular security audits
- SOC 2 Type II certified
- GDPR compliant

### Where is my data stored?

Data is stored in secure data centers in:
- United States (US-East, US-West)
- European Union (Ireland)
- Asia-Pacific (Singapore)

You can choose your preferred region during setup.

### Can I export my data?

Yes, you can export all your data at any time:
1. Go to Settings → Data Management
2. Click "Export Data"
3. Choose format (JSON, CSV, or SQL)
4. Download the export file

### Do you have a bug bounty program?

Yes! We run a bug bounty program through HackerOne. Report security vulnerabilities and earn rewards. Visit [security.example.com](https://security.example.com) for details.

## Troubleshooting

### Why is my application slow?

Common causes and solutions:

1. **Check your plan limits** - You may be hitting rate limits
2. **Optimize your queries** - Use pagination and filtering
3. **Enable caching** - Reduce redundant API calls
4. **Check network latency** - Use a region closer to you
5. **Review error logs** - Look for performance bottlenecks

### I'm getting authentication errors

Try these steps:
1. Verify your API key is correct
2. Check if the key has expired
3. Ensure you're using the correct environment (test vs. production)
4. Verify your account is active
5. Check rate limits haven't been exceeded

### My webhook isn't working

Common webhook issues:

1. **Verify the URL** - Ensure it's publicly accessible
2. **Check the signature** - Validate webhook signatures
3. **Review firewall rules** - Allow our IP addresses
4. **Check SSL certificate** - Must be valid and not self-signed
5. **Test with ngrok** - For local development

### How do I debug issues?

Debugging tools available:

```bash
# Enable debug mode
export DEBUG=true

# Check logs
our-tool logs --tail 100

# Run diagnostics
our-tool doctor

# Test connectivity
our-tool ping
```

## Features

### Can I schedule tasks?

Yes, you can schedule tasks using:
- Cron expressions
- Recurring schedules
- One-time scheduled runs
- Timezone-aware scheduling

Example:
```yaml
schedule:
  cron: "0 0 * * *"  # Daily at midnight
  timezone: "America/New_York"
```

### Do you support webhooks?

Yes, we support webhooks for real-time notifications. Events include:
- Resource created/updated/deleted
- Task completed/failed
- User actions
- System events

### Is there a CLI tool?

Yes! Install our CLI:
```bash
npm install -g @example/cli
```

See the [CLI documentation](/guide/cli/) for usage.

### Can I integrate with other services?

We support integrations with:
- GitHub/GitLab/Bitbucket
- Slack/Discord/Teams
- AWS/Google Cloud/Azure
- Jira/Trello/Asana
- And more via Zapier

## Support

### How do I contact support?

Support channels:

| Channel | Response Time | Availability |
|:--------|:--------------|:-------------|
| Email | 24 hours | 24/7 |
| Chat | 1 hour | Business hours |
| Phone | Immediate | Enterprise only |

Email: [support@example.com](mailto:support@example.com)

### Do you offer training?

Yes, we provide:
- Free video tutorials
- Live webinars (monthly)
- Documentation and guides
- Custom training (Enterprise)

### Is there a community forum?

Yes! Join our community:
- [Forum](https://forum.example.com)
- [Discord](https://discord.gg/example)
- [Stack Overflow](https://stackoverflow.com/questions/tagged/example)
- [Reddit](https://reddit.com/r/example)

### How do I report a bug?

Report bugs via:
1. [GitHub Issues](https://github.com/yourusername/yourrepository/issues)
2. Email: [bugs@example.com](mailto:bugs@example.com)
3. In-app feedback button

Include:
- Steps to reproduce
- Expected behavior
- Actual behavior
- Error messages
- Screenshots if applicable

## Migration

### Can I migrate from another platform?

Yes, we provide migration tools for:
- Heroku
- AWS Lambda
- Google Cloud Functions
- Azure Functions
- Custom platforms (contact support)

### How long does migration take?

Migration timeline:
- Small projects: < 1 hour
- Medium projects: 2-4 hours
- Large projects: 1-2 days
- Enterprise: Custom timeline

### Will there be downtime?

No, our migration process is designed for zero downtime. We migrate in phases and switch over when ready.

---

## Still have questions?

Can't find what you're looking for?

[Contact Support](mailto:support@example.com){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 }

---

<small>FAQ last updated: {{ site.time | date: "%B %d, %Y" }}</small>