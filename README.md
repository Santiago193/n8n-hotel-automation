# Hotel WhatsApp Automation System

An AI-powered customer support automation system for hotels, built with [n8n](https://n8n.io). It handles incoming WhatsApp messages across text, image, and audio formats — classifying conversations, routing them intelligently, processing bank transfer receipts, and giving administrators a full control panel through Telegram.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Workflows](#workflows)
  - [hotel.json — Main Orchestrator](#hoteljson--main-orchestrator)
  - [sub-text-hotel.json — Text Processing](#sub-text-hoteljson--text-processing)
  - [sub-images-hotel.json — Media Analysis](#sub-images-hoteljson--media-analysis)
  - [telegram-hotel.json — Admin Panel](#telegram-hoteljson--admin-panel)
  - [limpieza-hotel.json — Maintenance](#limpieza-hoteljson--maintenance)
- [Database Schema](#database-schema)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Setup](#setup)
- [Telegram Bot Commands](#telegram-bot-commands)
- [Notes](#notes)
- [License](#license)

---

## Overview

This system automates hotel customer interactions over WhatsApp using a modular n8n workflow architecture. When a guest sends a message, the system:

1. Validates the sender against a spam blocklist
2. Checks if the user already has an open support case
3. Enforces business hours (6:00 AM – 9:00 PM), auto-replying outside that window
4. Routes the message to the appropriate sub-workflow based on content type
5. Uses AI to classify, respond to, or escalate the conversation

All workflows share a PostgreSQL database and communicate through n8n's `Execute Workflow` node, keeping each concern isolated and independently maintainable.

---

## Architecture

```
hotel.json  (entry point & orchestrator)
├── sub-text-hotel.json     → AI text classification & response
├── sub-images-hotel.json   → Image, audio & video analysis
├── telegram-hotel.json     → Admin control panel via Telegram bot
└── limpieza-hotel.json     → Scheduled cleanup & daily reporting
```

---

## Workflows

### `hotel.json` — Main Orchestrator

The entry point for all incoming WhatsApp events via webhook (Evolution API).

**Pipeline:**
1. **Webhook** — receives `messages.upsert` events from the WhatsApp provider
2. **user-data** — normalizes fields: `identifier`, `message`, `tipo`, `instancia`, `apikey`, `numero-limpio`
3. **Table setup** — ensures `conversaciones_dia` exists and registers the contact
4. **Spam check** — queries `spam_numeros`; blocked contacts are silently dropped
5. **Support check** — queries `en_espera`; users with open cases bypass the AI and go directly to support
6. **Business hours check** — a JavaScript node evaluates the current hour; messages outside 6 AM–9 PM receive an automated out-of-hours reply
7. **Message routing** — a Switch node splits by `messageType`:
   - `conversation` → `sub-text-hotel`
   - `imageMessage` / `audioMessage` / `videoMessage` → `sub-images-hotel`

---

### `sub-text-hotel.json` — Text Processing

Handles all incoming text messages with AI-driven classification and response.

**Pipeline:**
1. **Redis message buffer** — pushes the incoming message to a Redis list keyed by `identifier`, allowing multiple rapid messages to be grouped before processing
2. **Grouping check** — compares the first buffered message against the current one to detect when the user has finished typing
3. **IA-text-router** — a Groq-powered agent (Kimi K2) with access to the PostgreSQL chat history tool (`consultarhistorial`) classifies the message into one of seven categories:

   | Category | Description |
   |---|---|
   | `relevante` | Requires a response (questions, requests, greetings) |
   | `soporte` | User needs human assistance |
   | `irrelevante` | No actionable content (e.g., "ok", "👍") |
   | `spam` | Unsolicited promotions or bot-like behavior |
   | `insistencia` | Repeated unanswered messages |
   | `agendar` | Appointment scheduling intent |
   | `error` | Broken or incomprehensible message |

4. **Switch2** — routes to the appropriate branch per category
5. **AI Agent** — for `relevante` messages, a second Groq agent generates a conversational WhatsApp-style reply using PostgreSQL-backed memory (`memoryPostgresChat`)
6. **Statistics update** — updates `conversaciones_dia` flags (`relevante`, `soporte`, `spam`) for daily reporting
7. **Support escalation** — `soporte` messages insert the user into `en_espera` and notify the admin via Telegram

---

### `sub-images-hotel.json` — Media Analysis

Processes images, audio, and video messages sent via WhatsApp.

**Pipeline:**
1. **Switch** — detects media type: `imageMessage`, `audioMessage`, or `videoMessage`
2. **Convert to binary** — decodes the base64 payload into a binary file
3. **Gemini 2.5 Flash** — analyzes the media and produces a detailed description or transcription
4. **image-agent** — a Groq agent (Kimi K2) generates a natural, WhatsApp-appropriate reply based on the media description
5. **Bank transfer detection** — a parallel branch uses a structured output parser to determine if the image is a Banco Pichincha transfer receipt:
   - Extracts: `numero_transferencia`, `monto`, `fecha`, `remitente`, `destinatario`, `banco`
   - Checks for duplicates in the `transferencias` table before inserting
   - Notifies the admin via Telegram with transfer details
   - Sends the user a confirmation or duplicate-warning message

---

### `telegram-hotel.json` — Admin Panel

A Telegram bot that gives hotel staff real-time control over the system.

**Trigger:** Telegram Trigger (listens for incoming messages from authorized users)

**Available commands:**

| Command | Description |
|---|---|
| `/help` | Shows the full command reference |
| `/verificarpago <number>` | Marks a bank transfer as verified and notifies the guest on WhatsApp |
| `/listarpendientes` | Lists all unverified bank transfers |
| `/liberar <number>` | Removes a phone number from the active support queue |
| `/agregarsoporte <number>` | Manually adds a phone number to the support queue |
| `/estadisticas` | Displays today's conversation breakdown (total, support, relevant, pending, spam) |

Bot messages are filtered to ignore other bots (`is_bot` check). All operations interact directly with PostgreSQL.

---

### `limpieza-hotel.json` — Maintenance & Reporting

A scheduled workflow that runs daily to keep the system clean and generate reports.

**Pipeline:**
1. **Statistics snapshot** — queries `conversaciones_dia` for daily totals and sends a formatted summary to the admin via Telegram
2. **Daily cleanup** — truncates `conversaciones_dia` and `en_espera`
3. **Weekly check** — if today is Sunday (`getDay() === 0`):
   - Queries the last 20 human messages from `n8n_chat_histories`
   - Sends them to a Groq AI agent for a weekly conversation summary
   - Sends the summary to the admin via Telegram
   - Truncates `n8n_chat_histories`
   - Sends a weekly cleanup confirmation

---

## Database Schema

All tables are created automatically by the workflows on first run.

```sql
-- Tracks all unique contacts per day with classification flags
CREATE TABLE IF NOT EXISTS conversaciones_dia (
  id            SERIAL PRIMARY KEY,
  identificador VARCHAR(30) NOT NULL UNIQUE,
  relevante     BOOLEAN DEFAULT false,
  soporte       BOOLEAN DEFAULT false,
  spam          BOOLEAN DEFAULT false,
  pendiente     BOOLEAN DEFAULT false
);

-- Contacts currently in the human support queue
CREATE TABLE IF NOT EXISTS en_espera (
  identificador TEXT PRIMARY KEY
);

-- Blocked phone numbers
CREATE TABLE IF NOT EXISTS spam_numeros (
  identificador TEXT PRIMARY KEY
);

-- Bank transfer receipts submitted via WhatsApp
CREATE TABLE IF NOT EXISTS transferencias (
  id                    SERIAL PRIMARY KEY,
  numero_transferencia  VARCHAR(20) UNIQUE NOT NULL,
  monto                 NUMERIC(12, 2) NOT NULL,
  fecha                 VARCHAR(20),
  remitente_nombre      TEXT,
  remitente_cuenta      VARCHAR(20),
  destinatario_nombre   TEXT,
  destinatario_cuenta   VARCHAR(20),
  banco                 VARCHAR(100),
  numero_telefono       VARCHAR(20),
  verificado            BOOLEAN DEFAULT false,
  identifier            VARCHAR(30),
  instacia              VARCHAR(30),
  api_number_instancia  VARCHAR(100)
);

-- AI conversation memory (managed by n8n's memoryPostgresChat node)
-- n8n_chat_histories
```

---

## Tech Stack

| Component | Role |
|---|---|
| [n8n](https://n8n.io) | Workflow automation engine |
| [PostgreSQL](https://www.postgresql.org) | Persistent storage for all data |
| [Redis](https://redis.io) | Message buffering for grouped WhatsApp messages |
| [Evolution API](https://github.com/EvolutionAPI/evolution-api) | WhatsApp messaging gateway |
| [Telegram Bot API](https://core.telegram.org/bots/api) | Admin control panel |
| [Groq](https://groq.com) (Kimi K2 / GPT-OSS) | Message classification and AI responses |
| [Google Gemini 2.5 Flash](https://deepmind.google/technologies/gemini/) | Image, audio, and video analysis |

---

## Prerequisites

- A running **n8n** instance (self-hosted or cloud)
- **PostgreSQL** database accessible from n8n
- **Redis** instance accessible from n8n
- **Evolution API** instance connected to a WhatsApp number
- A **Telegram Bot** token (via [@BotFather](https://t.me/BotFather))
- API credentials for **Groq** and **Google Gemini (PaLM)**

---

## Setup

1. **Import workflows** — import all five `.json` files into your n8n instance via *Settings → Import Workflow*

2. **Configure credentials** — in n8n, create and attach the following credentials:
   - `Postgres account` — your PostgreSQL connection
   - `Redis account` — your Redis connection
   - `Telegram account` — your Telegram bot token
   - `Groq account` — your Groq API key
   - `Google Gemini (PaLM) Api account` — your Gemini API key

3. **Set the webhook URL** — copy the webhook URL from `hotel.json` and register it as the webhook endpoint in your Evolution API instance

4. **Activate workflows** — enable all five workflows. The `limpieza-hotel` schedule and `telegram-hotel` trigger will start listening immediately

5. **Verify tables** — send a test message through WhatsApp; the first execution will auto-create all required database tables

---

## Telegram Bot Commands

Once the bot is running, send `/help` to get the full command list. All commands require the sender's Telegram ID to match the configured admin ID (`idjefetelegrm`).

```
/help                     Show this help menu
/verificarpago <number>   Verify a bank transfer by transfer number
/listarpendientes         List all unverified transfers
/liberar <number>         Release a number from the support queue
/agregarsoporte <number>  Add a number to the support queue
/estadisticas             Show today's conversation statistics
```

---

## Notes

- **Credentials are not included** — all API keys and connection strings must be configured in your n8n instance
- **Business hours** are hardcoded as 6:00 AM – 9:00 PM in the `Code` node inside `hotel.json`; adjust the JavaScript logic there if needed
- **Admin Telegram ID** is set as a static value (`idjefetelegrm`) in the `user-data` node of `hotel.json`; update it to match your Telegram user ID
- The **Evolution API base URL** (`172.17.42.196:8080`) is embedded in HTTP Request nodes across multiple workflows; update it to your actual server address before deploying

---

## License

This project is licensed under the [MIT License](LICENSE).

---

*Built by Santiago David — Automation & AI Engineering*
