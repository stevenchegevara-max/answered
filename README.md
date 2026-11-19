# answered

an AI receptionist that answers a small business's phone, books real appointments into the owner's calendar, and emails a summary of every call. Built to be resold: one engine, a new script per client.

## the problem

small service businesses miss calls while doing the actual work. Missed calls are lost revenue. Hiring a receptionist costs more than most solo owners can justify.

## what it does

- answers calls around the clock with a natural voice
- speaks English, German and French, detects and follows the caller's language automatically
- checks the real Google Calendar and offers only free slots
- books the appointment, sends the caller a confirmation email and the owner a notification
- takes callback requests and flags urgent ones for same-day callback
- logs every call to a Google Sheet
- sends the owner an email summary after each call and a digest every Monday morning

## live deployment

runs in production for **TourwithChe** (tourwithche.com), a Rwanda tour company. Emma, the receptionist, answers questions about gorilla trekking and safaris, quotes published package ranges, and books free 30-minute planning calls into the founder's calendar.

## architecture

```
caller
  -> phone number (Telnyx)
  -> Vapi assistant (Deepgram nova-3 STT, gpt-4o-mini, OpenAI TTS)
  -> tool webhooks (HTTPS via Tailscale Funnel)
  -> n8n on an Oracle Cloud Always Free VM (Docker)
       -> Google Calendar   availability check + event creation
       -> Gmail             caller confirmation + owner notification
       -> Google Sheets     lead log + follow-up queue
```

three n8n workflows:

| workflow | trigger | job |
|---|---|---|
| a, tools | Webhooks from Vapi | check availability, book, log callback |
| b, call reports | End-of-call webhook | sheet log, owner summary email, follow-up queue |
| c, weekly digest | Schedule, Monday 8am | weekly stats email for the owner |

## design decisions

- **recording always off, transcript-only.** German legal guardrail, GDPR-friendly by default.
- **Read-only Google scopes** wherever write access is not needed.
- **Tool calls answer in a single line** so the voice model can speak the result directly.
- **Honest AI disclosure**: the assistant says it is automated when asked (EU AI Act transparency).
- **Costs under EUR 25/month**: Vapi usage credit plus free tiers for everything else. The server costs nothing (Oracle Always Free), the URL costs nothing (Tailscale Funnel).

## latency and voice tuning

- target: first response audio under about 1 second after the caller stops speaking
- transcription-based endpointing, tuned wait times per punctuation type
- barge-in enabled: the caller can interrupt, the assistant stops instantly
- spoken filler line before calendar lookups to mask tool latency
- speech-to-text keyword boosting for brand and place names

## repo layout

```
vapi/   assistant config and system prompt
n8n/    the three workflows, importable JSON
docs/   rebuild guide, zero to live in an afternoon
```

## setup

see [docs/rebuild-guide.md](docs/rebuild-guide.md). The files in this public repo use placeholders: replace `owner@example.com` and `https://your-n8n-server.example.com` with your own values.

## adapting to a new business

the engine is generic. A new client means: a new system prompt with their facts and prices, a renamed calendar and sheet, and their voice in the email copy. Everything else stays untouched.

## license

MIT
