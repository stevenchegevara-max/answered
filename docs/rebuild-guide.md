# Rebuild guide

From zero to a live AI receptionist. Assumes a Google account, a Vapi account, and a Telnyx number.

## 1. Server (Oracle Cloud Always Free)

1. Create an Oracle Cloud account, home region Germany Central (Frankfurt).
2. Create a VM.Standard.E2.1.Micro instance (under "Specialty and previous generation"), Ubuntu 24.04, with a public IP. ARM A1.Flex is better but often out of capacity.
3. SSH in as `ubuntu`, then:

```bash
sudo apt update && sudo apt install -y docker.io
sudo fallocate -l 2G /swapfile && sudo chmod 600 /swapfile
sudo mkswap /swapfile && sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
sudo docker run -d --name n8n --restart unless-stopped -p 5678:5678 \
  -v n8n_data:/home/node/.n8n \
  -e GENERIC_TIMEZONE="Europe/Berlin" \
  -e NODE_FUNCTION_ALLOW_EXTERNAL=luxon \
  docker.n8n.io/n8nio/n8n
```

## 2. Permanent HTTPS URL (Tailscale Funnel)

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
sudo tailscale funnel --bg 5678
```

Enable HTTPS certificates in the Tailscale admin console (DNS page) first. The funnel URL becomes `https://<machine>.<tailnet>.ts.net`.

For editing n8n, use an SSH tunnel so Google OAuth still sees localhost:

```bash
ssh -i <key> -L 5678:localhost:5678 ubuntu@<server-ip>
```

## 3. Google

1. Google Cloud project, OAuth client (Web application), callback `http://localhost:5678/rest/oauth2-credential/callback`. Enable Calendar, Sheets, Gmail APIs.
2. Create a calendar named `TourwithChe Planning` and a spreadsheet named `TourwithChe Leads` with tabs `Leads` and `Follow-ups`.
3. In n8n, create Google credentials for Calendar, Sheets, Gmail with that OAuth client.

## 4. n8n workflows

Import the three files from `n8n/`, re-select the Google credentials on every Google node, select the sheet and tabs in workflow C, publish all three.

Webhook paths (keep these exact paths, the Vapi tools point at them):

- `/webhook/brightmove-check-availability`
- `/webhook/brightmove-book-appointment`
- `/webhook/brightmove-request-callback`
- `/webhook/brightmove-end-of-call`

## 5. Vapi

1. Create the three tools from `vapi/emma-tourwithche-assistant.json` (descriptions and parameter schemas are in there), Server URL pointing at the funnel URL + paths above.
2. Create the assistant: paste the system prompt from `vapi/emma-tourwithche-system-prompt.txt`, attach the three tools, set the end-of-call Server URL to the funnel URL + `brightmove-end-of-call`, transcriber Deepgram nova-3 language `multi`, recording off.
3. Publish, then attach the Telnyx number.

## 6. Test

Call the number. Ask about a trip, book a planning call, hang up. Verify: calendar event, confirmation email, owner email, new row in the Leads sheet, and the end-of-call summary email.

## New client checklist

1. New system prompt with their facts, services, prices, hours.
2. New Google calendar and sheet names, updated in the workflows.
3. Email copy in workflow A and B updated to their voice.
4. Their Vapi phone number. Done.
