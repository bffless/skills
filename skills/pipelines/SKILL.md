---
name: pipelines
description: Backend automation without code using handler chains
---

# Pipelines

**Docs**: https://docs.bffless.app/features/pipelines/

Pipelines provide backend functionality for static sites without writing server code. Chain handlers together to process forms, store data, send emails, and more.

## Handler Types

| Handler | Purpose |
|---------|---------|
| **Form** | Parse form submissions (multipart, JSON, URL-encoded) |
| **Data CRUD** | Create, read, update, delete DB Records |
| **Email** | Send emails via configured provider |
| **Response** | Return custom JSON or redirect |
| **Function** | Custom JavaScript for complex logic |
| **Aggregate** | Combine data from multiple sources |

## DB Records

Schema-based data storage built into BFFless:

1. Define schema in project settings (fields, types, validation)
2. Use Data CRUD handler to interact with records
3. Query with filters, sorting, pagination

## Expression Syntax

Access data throughout the pipeline using expressions:

- `input.*` - Parsed request body
- `query.*` - URL query parameters
- `params.*` - URL path parameters
- `headers.*` - Request headers
- `steps.<name>.*` - Output from previous handler
- `user.*` - Authenticated user info (if applicable)

Example: `${input.email}` or `${steps.createUser.id}`

## Common Workflows

**Contact form:**
1. Form handler → parse submission
2. Data CRUD → store in "submissions" schema
3. Email → notify admin
4. Response → thank you message

**Waitlist signup:**
1. Form handler → parse email
2. Data CRUD → check if exists, then create
3. Email → send confirmation to user
4. Response → success or "already registered"

**Webhook receiver:**
1. Form handler → parse JSON payload
2. Function → validate signature, transform data
3. Data CRUD → store event
4. Response → 200 OK

## Configuration Tips

1. Name handlers descriptively for readable expressions
2. Use Response handler last to control what client sees
3. Test with simple inputs before adding validation
4. Check pipeline logs for debugging failed executions

## Troubleshooting

**Pipeline not triggering?**
- Verify endpoint path matches request URL
- Check HTTP method (GET, POST, etc.) is correct
- Ensure pipeline is enabled and deployed

**Expression returning undefined?**
- Check handler name matches exactly (case-sensitive)
- Verify previous handler completed successfully
- Use pipeline logs to see actual values at each step
