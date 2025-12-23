🏨 n8n Hotel Automation System

Automated customer support system for hotels built with n8n, integrating WhatsApp, Telegram, PostgreSQL, and AI models to handle conversations, classify messages, analyze media, manage support cases, and perform automated maintenance tasks.

📌 Overview

This project implements a modular automation architecture using n8n to manage hotel customer interactions across multiple channels.

The system:

Receives incoming WhatsApp messages (text, images, audio)

Classifies conversations using AI

Routes messages to automated responses or human support

Provides an administrative panel via Telegram

Generates daily statistics and performs scheduled database cleanup

🧱 System Architecture

The solution is divided into independent but connected workflows, each responsible for a specific task:

hotel.json
 ├── sub-text-hotel.json     # AI text processing & classification
 ├── sub-images-hotel.json   # Image & audio analysis
 ├── telegram-hotel.json     # Admin panel via Telegram
 └── limpieza-hotel.json    # Scheduled cleanup & reports

🔁 Main Workflow
hotel.json

Role: Entry point and workflow orchestrator.

Receives WhatsApp events via webhook

Normalizes user data (instance, API key, phone number)

Detects message type (text, image, audio)

Routes execution to the appropriate sub-workflow

🤖 AI Text Processing
sub-text-hotel.json

Handles incoming text messages using AI:

Classifies messages into:

soporte (support)

spam

agendar

insistencia

irrelevante

Uses AI language models (Groq) with conversation memory

Sends automated responses via WhatsApp API

Stores conversational context using Redis / PostgreSQL memory

🖼️ Media Analysis
sub-images-hotel.json

Processes images and audio messages:

Converts media to base64

Analyzes content using AI

Extracts relevant data (e.g., payment receipts)

Stores extracted information in the database

Integrates results back into the main workflow

🧑‍💼 Admin Panel (Telegram)
telegram-hotel.json

Provides a Telegram-based control panel for administrators:

Manage support cases (add / remove numbers)

Verify payments

List pending conversations

View daily statistics

Receive system notifications

All actions interact directly with the PostgreSQL database.

🧹 Maintenance & Automation
limpieza-hotel.json

Scheduled workflow for system maintenance:

Generates daily conversation statistics

Sends reports to Telegram

Cleans:

Chat history tables

Daily conversation records

Support queues

Performs weekly cleanup based on day-of-week logic

🛠️ Tech Stack

n8n – Workflow automation

PostgreSQL – Persistent data storage

Redis – Conversation memory

WhatsApp API – Customer communication

Telegram Bot API – Admin interface

Groq AI Models – Message classification & analysis

Node.js – Runtime environment

🚀 Features

Modular workflow architecture

AI-powered message classification

Multi-channel support (WhatsApp + Telegram)

Automated media analysis

Admin backoffice via Telegram

Scheduled cleanup & reporting

Scalable and maintainable design

⚠️ Notes

Credentials and API keys are not included for security reasons

Workflows must be imported into an existing n8n instance

Database tables are created automatically if they do not exist

📂 Usage

Import the .json workflows into n8n

Configure required credentials (PostgreSQL, WhatsApp API, Telegram)

Activate the workflows

Set up the webhook endpoint in your WhatsApp provider

📄 License

This project is licensed under the MIT License.


✨ Author

Santiago David
Software Engineering / Automation & AI
