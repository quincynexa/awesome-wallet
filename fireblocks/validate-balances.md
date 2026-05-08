> ## Documentation Index
> Fetch the complete documentation index at: https://developers.fireblocks.com/llms.txt
> Use this file to discover all available pages before exploring further.

<AgentInstructions>

## Submitting Feedback

If you encounter incorrect, outdated, or confusing documentation on this page, submit feedback:

POST https://developers.fireblocks.com/feedback

```json
{
  "path": "/docs/validate-balances",
  "feedback": "Description of the issue"
}
```

Only submit feedback when you have something specific and actionable to report.

</AgentInstructions>

# Validate Balances

> Fireblocks customers can create automated mechanisms to notify their organization about incoming transactions using the following methods:

* Webhook notifications
* Pulling the transaction history log using the Fireblocks API

When monitoring notifications through these methods, only consider incoming transactions for reconciliation after receiving a notification that the transaction’s status has been updated to `COMPLETED` and the `VAULT_BALANCE_UPDATE` webhook has been received.

Fireblocks provides additional information to help clients validate and reconcile their balances according to the blockchain state. This information includes:

* The [`TRANSACTION_STATUS_UPDATED`](/reference/transaction-webhooks#transaction-status-updated) webhook event contains the `blockInfo` object, which includes:

  * `blockHeight`: The block height where the transaction was included
  * `blockHash`: The block hash where the transaction was included

* The `VAULT_BALANCE_UPDATE` webhook event and the `GET /vault/accounts/{vaultAccountId}/{assetId}` endpoint both include `blockHeight` and `blockHash` properties.
  * For UTXO-based assets (e.g., BTC, LTC, BCH, DOGE), `blockHash` is not currently populated in `VAULT_BALANCE_UPDATE` responses. Only `blockHeight` is available. To retrieve `blockHash` for UTXO-based transactions, use `blockInfo.blockHash` from the `TRANSACTION_STATUS_UPDATED` webhook or the `GET /transactions/{txId}` endpoint.

* The [`GET /transactions/{txId}`](/reference/gettransaction) endpoint returns the transaction by its Fireblocks ID and includes the `blockInfo` object with the `blockHeight` and `blockHash` properties

Before crediting the end-user, customers need to run a reconciliation process to ensure that the incoming balance is accurately reflected and up to date according to the same `blockHeight` provided in the transaction notification object. If the balance is not up to date with the network state and the same height value, customers should not credit their clients until the issue is inspected and resolved.

Fireblocks strongly recommends checking the wallet’s balance using the Fireblocks API to verify that the deposit is included and the balance is up to date. This is crucial to prevent potential loss of funds from crediting end-clients before the balance is truly updated. While Fireblocks takes extensive measures to ensure the accuracy and reliability of its services, it is important for customers to implement all necessary validations and follow best practices to avoid any potential issues.

Below is a sequence diagram for additional reference:

<img src="https://mintcdn.com/fireblocks-43c4b3ee/8uV7V_rBjqF0ZDAg/images/docs/24e39d7-validate_balance.png?fit=max&auto=format&n=8uV7V_rBjqF0ZDAg&q=85&s=9ba69787bca6ac10b1f8c36127926576" alt="" width="1091" height="564" data-path="images/docs/24e39d7-validate_balance.png" />
