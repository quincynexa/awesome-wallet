> ## Documentation Index
> Fetch the complete documentation index at: https://developers.fireblocks.com/llms.txt
> Use this file to discover all available pages before exploring further.

<AgentInstructions>

## Submitting Feedback

If you encounter incorrect, outdated, or confusing documentation on this page, submit feedback:

POST https://developers.fireblocks.com/feedback

```json
{
  "path": "/reference/webhooks-best-practices",
  "feedback": "Description of the issue"
}
```

Only submit feedback when you have something specific and actionable to report.

</AgentInstructions>

# Best practices for webhooks

Webhooks provide transaction status and balance updates. We recommend using webhooks instead of synchronously polling transaction and balance endpoints. This approach is more efficient and better supports internal reconciliation logic.

## Triggering internal logic on incoming transactions

When an incoming transaction triggers internal logic on your side (for example, sweeping), we recommend triggering that logic only after receiving the balance update webhook, rather than acting on the incoming transaction webhook alone. Additionally, verify that the wallet balance update reflects the same block height as the transaction.

> Acting on the incoming transaction webhook before the balance is confirmed can lead to reconciliation errors. Always wait for the balance update webhook and cross-check the block height before proceeding.

***

## Handle events asynchronously

Configure your handler to process incoming events with an asynchronous queue. Processing transactions synchronously can lead to long response times and potential timeouts, particularly during periods of high market volatility when transaction volumes surge.

If all events are handled synchronously, sudden surges in activity could overwhelm your system and degrade performance. Asynchronous queues allow you to manage these transaction bursts efficiently, ensuring your system processes events at a sustainable rate without bottlenecks or failures.

***

## Respond with a 2xx status promptly

Your endpoint should return a successful 2xx status code as soon as possible, before executing any time-consuming logic that might lead to a timeout. For instance, send a 200 response before processing tasks like marking a customer’s invoice as paid in your accounting system.

***

## Managing duplicate events

Your webhook endpoint may sometimes receive the same event multiple times. To prevent duplicate processing, track event IDs and ignore any that have already been handled.

Additionally, ensure you're processing the most recent event by comparing timestamps in the **createdAt** field, rather than relying solely on the event ID.

***

## Event ordering

Although Fireblocks strives to send notifications in order per resource, we cannot guarantee that events will always be delivered in the sequence they are generated. We recommend designing your endpoint to process events independently, without relying on their delivery order.

***

## Circuit breaker

Fireblocks introduces a circuit breaker mechanism for managing faulty webhook endpoints to ensure fairness and high throughput across all users. This feature helps identify and suspend endpoints that consistently fail, ensuring that system resources are optimized and reliable for everyone.

The circuit breaker monitors error rates for each webhook endpoint and applies suspension policies based on predefined thresholds. These policies vary depending on the type and severity of the errors. Webhooks with an error rate exceeding a specific threshold will be automatically disabled to protect system integrity.

To prevent performance issues in your application, handle events asynchronously with queues and quickly return a 2xx status code.

***

## Recovering from downtime

If your webhook server was offline for an hour or longer due to a deployment or network issue, you can resend all missed or failed webhook notifications for a given webhook. You can set the resend duration for up to 24 hours.
