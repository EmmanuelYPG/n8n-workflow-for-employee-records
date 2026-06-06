# n8n Registration & Weekly Digest Workflow

An automation workflow built in n8n that manages the complete lifecycle of user registration from a web form: validation, storage, multi-channel notifications, geographic enrichment, and weekly digest. The workflow is active in production and consists of two completely independent flows that share the same Google Sheets document as a data source.

---

## General Architecture

The file contains 23 nodes distributed across two flows. The first responds in real-time to each registration form submission; the second runs in the background once per week and consolidates daily activity into a report for the team. Additionally, there is a third Telegram trigger configured but without outgoing connections, which is documented at the end as pending work.

The active integrations are Google Sheets (read and write), HubSpot CRM (contact creation), Google Tasks (follow-up task creation), Telegram (team alerts), SMTP (transactional emails to users and digests to the team), and ipapi.co (IP geolocation).

---

## Flow 1 — Registration via Webhook

### Trigger

The flow is activated by an HTTP POST webhook exposed at the `/registration-form` route. The response to the client is not returned immediately but is handled by a `Respond to Webhook` node at the end of the process, which allows processed data to be included in the response instead of a simple acknowledgment.

### Data Normalization

The first step transforms raw request body fields into a uniform internal schema. The `body.full_name` field is converted to `Name`, the `body.question1` field is converted to `phone`, and a `timestamp` with the date and time of receipt is automatically added. This node acts as a data contract for all subsequent nodes: from here on, no node accesses the original webhook body directly.

### Email Validation and Main Branching

The `email validation` node applies a regular expression to the `email` field to verify it has the minimum required format (`^[\w.-]+@[\w.-]+\.[a-zA-Z]{2,}$`). From this node, two mutually exclusive branches emerge.

If the email **fails validation**, the workflow records the data in the `errors` tab of the Google Sheets spreadsheet and returns a failed JSON response to the client:

```json
{ "Status": "Failed", "message": "the email was invalid" }
```

The process ends there for that registration.

If the email **passes validation**, n8n simultaneously activates two sub-branches in parallel from the same IF node output.

### Sub-branch A — Registration, Email, and CRM

This chain handles the main path for a valid user. First, it appends to the `users` tab of the spreadsheet, persisting the five normalized fields. It then waits 10 seconds—likely to ensure the Sheets write has been consolidated before proceeding—and then sends a confirmation HTML email directly to the registered user, with the subject *"Registro a movistar completado!"*, listing the data they just submitted.

After the email, the flow checks the phone number: if it starts with `55` it's a Mexico City number and the contact is created in HubSpot with email, name, and phone but without a country field. If it doesn't start with `55`, the contact is still created in HubSpot but with the `country` field set to `"NON mexican"`. Both branches converge at the `Create a task` node, which generates a task in Google Tasks with the title `"search employees: [Name]"`, a due date 5 days out, and complete registration notes. Finally, the `Respond to Webhook` node returns a success response to the client:

```json
{
  "status": "success",
  "message": "Form recived correctly",
  "nombre": "...",
  "timestamp": "..."
}
```

### Sub-branch B — Geographic Enrichment and Telegram Alert

This chain runs in parallel to the previous one from the moment the email is validated. It takes the IP address that the form sends in `body.ip` and makes a POST request to `https://ipapi.co/{ip}/json` to obtain the user's geolocation. From the response, it extracts Country, IP, City, Postal Code, Latitude, and Longitude. With that data, it sends a text message to the internal team's Telegram channel reporting the name, email, country, and postal code of the new registration. This node also points to the same success response node as sub-branch A.

---

## Flow 2 — Weekly Digest

### Trigger

This flow does not respond to external events. A `Schedule Trigger` activates it automatically every Monday at 7:00 AM. It has no dependency on Flow 1 other than reading from the same Google Sheets document.

### Filtering Logic and Conditional Sending

The flow reads all rows from the `users` tab and passes them to a JavaScript code node that filters them, keeping only those whose `timestamp` corresponds to today's date:

```javascript
const today = new Date().toISOString().split("T")[0];
const filteredItems = items.filter((item) => {
  const timestamp = new Date(item?.json?.timestamp).toISOString().split("T")[0];
  return timestamp === today;
});
return filteredItems;
```

An IF node checks if the resulting array has at least one element. If there are no records for the day, the flow ends without sending anything. If there are, a second code node builds an HTML `<ul>` list with the name and email of each user for the day, and that list is injected into the body of an HTML email sent to the team address, with the subject `"Resumen de contactos - [date]"`.

---

## Credentials Configuration

For the workflow to work on your own n8n server, you must configure the following credentials in the Credentials section of the panel:

The **Google Sheets** and **Google Tasks** credentials must grant access to the spreadsheet with ID `1JqoaBMuZdcymHSGH12LWRAhBOTfdlnqIgQgjXBvPP1s`. The **HubSpot** account authenticates via OAuth2. The **Telegram** node requires the bot token and has the chat ID `6697851144` configured as the fixed destination for alerts. The email nodes (`emailSend`) are configured with the account `emmanuel7pg7@gmail.com` as the sender; the weekly digest goes to `elviflores2017@gmail.com`.

The IP enrichment node consumes `https://ipapi.co/{ip}/json` without authentication, although the API has limits on its free plan that should be reviewed under high load.

---

## Google Sheets Spreadsheet Structure

The workflow writes to two tabs in the same file *"Usuarios registrados"*:

The **`users`** tab receives records with valid emails and is the data source for the weekly digest. The **`errors`** tab receives attempts with invalid emails, useful for auditing. Both expect columns with the headers `Name`, `Email`, `phone`, `message`, and `timestamp`.

---

## Known Issues

**Critical Bug — incorrect email field in HubSpot for non-Mexican numbers.** In the `non mexican number` node, the `email` field of the HubSpot contact is incorrectly mapped to the phone number instead of the email:

```
"email": "={{ $('Wait 10 sec').item.json.phone }}"
```

This causes the contact created in HubSpot to have a phone number where it should have an email, which breaks any deduplication or subsequent flow based on that field. The correction is:

```
"email": "={{ $('email validation').item.json.email }}"
```

**Potential Bug — `$items.length` expression in digest IF node.** The `If1` node in the weekly flow evaluates the condition with `={{ $items.length }}`, which is not a standard expression for n8n v1. In recent versions of the expression engine, `$items` without arguments can resolve to `undefined`, causing the `> 0` condition to always be `false` and the digest to never be sent. The correct form in n8n v1 is:

```
={{ $input.all().length }}
```

It is recommended to verify this node with a manual execution before considering the weekly flow stable.

---

## Pending Work

There is a `Telegram Trigger` node configured to listen for incoming messages from the bot, but it has no outgoing connections. It is incomplete and performs no action in its current state. The intended flow is not documented; any extension of this workflow should resume it or remove it to avoid confusion.

---

## Version and Environment

Workflow created with n8n. The internal ID of the workflow is `oclUYrfobpHx35aq` and it is marked as active (`"active": true`). No credentials are included in the export file.
