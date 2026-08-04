# Bluemantle CRM

> A comprehensive, self-hosted WhatsApp® CRM tailored for Bluemantle LLP.
> Built by **ZAV INFO TECH**.
> 
> 🌐 **Visit our website:** [bluemantletechnology.com](https://bluemantletechnology.com/)

---

## 🌟 Overview

Bluemantle CRM is a powerful system designed to handle WhatsApp Business communications. It offers a shared inbox, contact management, sales pipelines, broadcast messaging, and advanced no-code automations to streamline your business operations.

## ✨ Key Features

- **Shared Inbox:** Multiple agents working from a single WhatsApp number.
- **Contact Management:** Tags, custom fields, deduplication, and easy imports.
- **Sales Pipelines:** Kanban boards to track deals and link them to WhatsApp conversations.
- **Broadcasts:** Send Meta-approved template messages with dynamic variables and tracking.
- **No-Code Automations:** Visually build triggers for inbound messages, new contacts, keywords, and more.
- **AI Assistant:** Optional OpenAI/Anthropic integration for auto-replies and knowledge base retrieval.
- **Dashboard Analytics:** Track response times, messaging volume, and pipeline metrics in real-time.
- **Team Roles:** Owner, Admin, Agent, and Viewer access levels.

---

## 🚀 How to Run Locally

Follow these steps to get the CRM running on your local machine for development:

### 1. Clone the Repository
```bash
git clone https://github.com/bluemantleinstitute-bot/BMIT-CRM.git
cd BMIT-CRM
```

### 2. Install Dependencies
```bash
npm install
```

### 3. Setup Environment Variables
Duplicate the example environment file and fill in your Supabase and Meta credentials:
```bash
cp .env.local.example .env.local
```
*(Make sure to add your Supabase URL and Anon Key!)*

### 4. Start the Development Server
```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000) in your browser. You will be redirected to the login page.

---

## 📚 Stack

- **App:** Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS.
- **Database & Auth:** Supabase (Postgres, Auth, Storage, RLS).
- **Messaging:** Meta Cloud API (official WhatsApp Business API).

---

## 📄 License

This project is licensed under the [MIT License](./LICENSE). 
Copyright (c) 2026 BLUEMANTLE LLP. Built by ZAV INFO TECH.
