# RelayDesk

RelayDesk is a self-hosted n8n workflow set for handling Facebook Messenger support. It uses an FAQ-backed Gemini responder for messages it can answer and hands unanswered questions to a human support team in Telegram.

## What it does

- Checks incoming Messenger messages against FAQ data in Google Sheets and drafts supported answers with Google Gemini.
- Applies a business-scope guardrail and routes unanswered questions into a support queue.
- Notifies a Telegram support group, where staff can claim tickets and reply to customers.
- Forwards eligible staff replies back to Messenger and saves replies as FAQ candidates for review.
- Includes a Docker Compose stack for running n8n with task runners, the n8n sandbox services, and SearXNG.

The repository contains workflow exports, not a standalone application. All three exports are inactive; import and configure them in your own n8n instance before enabling them.

## Workflows

| Workflow | Purpose |
| --- | --- |
| [Messenger Support Bot](workflows/Messenger%20Support%20Bot.json) | Receives Messenger webhook events, attempts an FAQ-based response, and creates a support ticket when a human is needed. |
| [Support Queue Poller](workflows/Support%20Queue%20Poller.json) | Checks for pending tickets every minute and posts a claimable notification to Telegram. |
| [Telegram Reply Handler](workflows/Telegram%20Reply%20Handler.json) | Handles ticket claim/unclaim actions and forwards valid replies to Messenger. It also records replies as FAQ candidates. |

## Get started

### Requirements

- Docker Engine with the Docker Compose plugin, or an existing n8n instance. The included stack pins n8n to version `2.44.6`.
- A publicly reachable HTTPS URL for Messenger webhooks.
- Configured accounts for Facebook Messenger (Meta), Telegram, Google Sheets, and Google Gemini.

### Run the included stack

1. Copy the environment template and generate values for its secrets:

   ```sh
   cp .env.example .env
   openssl rand -hex 32
   ```

   Run `openssl rand -hex 32` for each secret placeholder and replace the literal `$(openssl rand -hex 32)` text in `.env` with the generated value. Compose does not execute shell commands from the env file. Set `N8N_SANDBOX_SERVICE_API_KEY` to the same value as an entry in `SANDBOX_API_KEYS`. Replace the example webhook/editor URLs with your public HTTPS base URL for a proxied deployment.

2. Add a SearXNG settings file. `docker-compose.yml` mounts `./searxng-settings.yml`, but that file is not included in this repository. Create it using the [SearXNG settings documentation](https://docs.searxng.org/admin/settings/index.html), or remove the SearXNG service and its related n8n setting if you do not need it.

3. Start the services and check their status:

   ```sh
   docker compose up -d
   docker compose ps
   ```

   The n8n editor is bound to `127.0.0.1:5678`; use a local browser or configure a reverse proxy for remote access. The sandbox runner uses privileged Docker-in-Docker. Review the Compose configuration and keep internal service ports private, especially on internet-facing hosts.

To stop the stack without deleting its persistent data, run `docker compose down`.

### Import and configure the workflows

1. In n8n, import each JSON file from the `workflows/` directory using **Import from File**. You can also use an existing n8n instance instead of the included Compose stack.
2. Set up credentials for Telegram, Google Sheets, Google Gemini, and Meta Graph API (the HTTP Header Auth credential). Imported credential references belong to the source instance and must be replaced with your own.
3. Re-select your Google Sheets documents/tabs and n8n Data Tables in the relevant nodes. The exports refer to resources from another instance. Provide FAQ, support-ticket, and FAQ-candidate sheets, plus the Data Table values used for the Facebook Page ID, Telegram group chat ID, and ticket sequence.
4. Configure the Meta webhook to point to the public n8n webhook URL, and set up the Telegram bot and support group. Review the FAQ content, business-scope prompt, and ticket fields for your organization.
5. Test each workflow with test accounts and data. Enable the workflows only after their credentials, resource references, and webhook configuration are correct.

For a customer, the intended flow is:

```text
Messenger message -> FAQ/Gemini answer, or a ticket for human support
                  -> Telegram agent claims and replies to the ticket
                  -> reply is sent to Messenger and saved as an FAQ candidate
```

For more detail on operating the stack and importing workflows, see the [n8n Docker guide](https://docs.n8n.io/hosting/installation/docker/) and [workflow import/export guide](https://docs.n8n.io/workflows/export-import/).

## Get help

- Browse the [workflow exports](workflows/) and [Compose configuration](docker-compose.yml).
- Report bugs or request features in the [GitHub issue tracker](https://github.com/mhrafin/n8n-fb-page-message/issues).
- Consult the [Meta Messenger Platform documentation](https://developers.facebook.com/docs/messenger-platform/), [Telegram Bot API](https://core.telegram.org/bots/api), and [Gemini API documentation](https://ai.google.dev/gemini-api/docs) for integration-specific setup.

## Maintainers and contributing

RelayDesk is maintained by [@mhrafin](https://github.com/mhrafin). Contributions are welcome; open an issue to discuss substantial changes, then submit a pull request with the updated workflow export(s) and a short description of how they were tested.

Before sharing workflow exports, remove credentials, tokens, private webhook details, and customer or test data. There is no automated test suite in this repository; at minimum, check that the JSON parses and import/test changes in a staging n8n instance. For example:

```sh
jq empty workflows/*.json
```

No `LICENSE` file is present. Check with the maintainer about reuse and redistribution terms.
