# HVAC Technician Dispatch Orchestrator

A multi-agent dispatch workflow built in n8n for an HVAC business. A customer request comes in through a webhook. The workflow checks the technician sheet for available technicians in the same zone with the right skill, and an AI agent picks the best match. If a technician is assigned, the customer gets an email. Emergencies and jobs with no match go to the dispatcher on Telegram.

## Architecture

```
Webhook → Validate Input → Input Valid?
   ├── No  → Respond Invalid (400)
   └── Yes → Read Technicians → Build Match Prompt → Needs Human?
        ├── Yes (emergency or no candidates) → Ask Dispatcher → Mark Escalated → Log Dispatch
        └── No  → Matching Agent → Parse Match → Assigned?
             ├── Yes → Send Customer Email → Final Record Assigned → Log Dispatch
             └── No  → Ask Dispatcher → Mark Escalated → Log Dispatch
   → Log Dispatch → Respond Result
```

## Features

* Validates name, email, phone, address, zone, service type, and urgency. Invalid input gets a 400 response
* Filters technicians by availability, zone, and skill before the AI sees them, so the model can only pick from valid candidates
* Matching agent chooses the technician with the fewest current jobs and returns a one-line reason
* Output is checked against the candidate list, so an invented technician ID is never assigned
* Emergency jobs always go to a human dispatcher
* Customer receives an email with the technician's name and phone
* Every dispatch is logged to Google Sheets with its status (`ASSIGNED` or `NEEDS_DISPATCHER`)
* Customer text is wrapped in tags and treated as data, which guards against prompt injection

## Tech Stack

* n8n 2.x (workflow automation)
* OpenAI `gpt-5-mini` (matching agent)
* Google Sheets (technician roster and dispatch log)
* Gmail API (customer emails)
* Telegram Bot API (dispatcher alerts)

## Setup

1. Create a Google Sheet called `HVAC Dispatch` with two tabs.
   Tab `Technicians`, row 1:

```
TechID | Name | Phone | Skills | Zone | Available | CurrentJobs
```

   Tab `Dispatches`, row 1:

```
Timestamp | DispatchID | Name | Phone | ServiceType | Zone | Urgency | TechID | TechName | Status | Reason
```

2. Import `hvac_technician_dispatch_orchestrator.json` into n8n.
3. Add credentials: OpenAI, Telegram, Google Sheets OAuth2, and Gmail OAuth2. Assign them to the matching nodes.
4. Replace `YOUR_DISPATCH_SHEET_ID` with your sheet ID (in the Read Technicians and Log Dispatch nodes).
5. Replace `YOUR_TELEGRAM_CHAT_ID` with your chat ID (in the Ask Dispatcher node).
6. Activate the workflow and send a POST request to the webhook.

## Sample Request

```
{
  "name": "Test Client",
  "email": "you@example.com",
  "phone": "03111222333",
  "address": "House 12, Block C, North Nazimabad",
  "zone": "North",
  "service_type": "AC",
  "urgency": "normal"
}
```

Use `"urgency": "emergency"` to test the dispatcher path.

## Project Notes

* Built as part of my AI Automation portfolio.
* Technician matching is done in code first, then the AI chooses among the remaining candidates. This keeps the model's choice inside the list of valid, available technicians.
* Only `gpt-5-mini` is used for the agent. Temperature and response format options are left at their defaults.
