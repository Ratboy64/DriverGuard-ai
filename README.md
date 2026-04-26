# DriveGuard AI 🛡️

**Real-time crash risk monitor powered by Claude AI**

A mobile-first Progressive Web App (PWA) built for iPhone 15. Monitors speed, G-force, and driving behavior in real time — with an AI coach powered by Anthropic's Claude that gives live feedback, analyzes incidents, and can send parent alert emails.

---

## Features

- 📡 **Live telemetry** — GPS speed + accelerometer G-force via your phone's sensors
- 🎯 **Composite risk score** — Weighted algorithm using speed, vehicle safety rating, crash probability, G-force, and maneuver severity
- 🤖 **Claude AI Coach** — Real-time driving feedback, chat interface, incident analysis
- 📧 **Parent alerts** — EmailJS integration sends alerts on threshold violations
- 🔒 **PIN-protected tuning** — Lock down settings with a 4–8 digit PIN
- 🚗 **NHTSA vehicle data** — Safety star ratings pulled from the National Highway Traffic Safety Administration
- 📋 **Incident log** — Full history with AI-generated summaries, CSV export
- 📲 **iPhone PWA** — Install to home screen for a native app feel

---

## Quick Deploy (GitHub Pages)

1. **Fork or clone** this repository
2. Go to your repo → **Settings → Pages**
3. Under **Source**, select `Deploy from a branch`
4. Choose `main` branch, `/ (root)` folder → **Save**
5. Your app will be live at: `https://YOUR-USERNAME.github.io/REPO-NAME/`

> GitHub Pages may take 1–2 minutes to go live on first deploy.

---

## iPhone 15 — Add to Home Screen

Once your GitHub Pages URL is live:

1. Open the URL in **Safari** on your iPhone
2. Tap the **Share** button (box with arrow up)
3. Scroll down and tap **"Add to Home Screen"**
4. Tap **Add** — DriveGuard AI now launches like a native app

> ⚠️ Motion sensor access on iOS 13+ requires a user gesture inside Safari. Use the "Enable Motion Sensors" button in the setup wizard.

---

## Setup Wizard

On first launch you'll be guided through 5 steps:

| Step | What to set up |
|------|----------------|
| 1 | **PIN** — protects your tuning settings (default: `2049`) |
| 2 | **Vehicle** — year, make, model for NHTSA safety star lookup |
| 3 | **Parent Alerts** — EmailJS account for email notifications |
| 4 | **AI Coach** — your Anthropic API key |
| 5 | **Permissions** — GPS + motion sensor access |

All data is stored locally on your device (`localStorage`). Your API key never leaves your phone.

---

## Getting Your Anthropic API Key

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. Sign up or log in
3. Navigate to **API Keys** → **Create Key**
4. Copy the key (starts with `sk-ant-...`)
5. Paste it in the setup wizard or in **⚙ TUNE → AI Coach Settings**

The app uses `claude-haiku-4-5` for efficient real-time coaching. Haiku is the fastest and most cost-effective model for high-frequency driving analysis.

---

## EmailJS Setup (Parent Alerts)

1. Create a free account at [emailjs.com](https://www.emailjs.com)
2. Add an **Email Service** (Gmail, Outlook, etc.) → note your **Service ID**
3. Create an **Email Template** with these variables:

```
To: {{to_email}}
Subject: DriveGuard AI Alert — {{trigger_type}}

Vehicle: {{vehicle}}
Risk Score: {{risk_score}}
Speed: {{speed}} MPH
Lateral G: {{lat_g}}
AI Analysis: {{ai_analysis}}
Incidents This Session: {{session_incidents}}
Time: {{timestamp}}
```

4. Note your **Template ID** and **Public Key**
5. Enter all three in the app under **⚙ TUNE → Parent Email**

---

## Tuning Panel

Unlock with your PIN, then adjust:

- **Risk Factor Weights** — how much each sensor contributes to the composite score
- **Alert Thresholds** — score, speed (MPH), and G-force trigger levels
- **Notification Mode** — immediate, cooldown, count-based, or session summary
- **Change Vehicle** — update year/make/model without going through wizard
- **VIN Decode** — precise NHTSA safety data from your 17-character VIN
- **Demo Mode** — simulate speed and G-force for testing without driving

---

## Demo Mode

Enable in **⚙ TUNE → Demo/Simulation** to test the app without driving. Set a simulated speed and lateral G value to see how the risk score, AI coach, and alerts all respond.

---

## Privacy

- All data (API key, vehicle info, incidents) is stored only in `localStorage` on your device
- No backend server — all API calls go directly from your browser to Anthropic and NHTSA
- The app can be used without an AI key in basic monitoring mode

---

## File Structure

```
driveguard-ai/
├── index.html       ← The entire app (single file)
├── manifest.json    ← PWA manifest for "Add to Home Screen"
├── icon-192.png     ← App icon (192×192)
├── icon-512.png     ← App icon (512×512)
├── .nojekyll        ← Tells GitHub Pages to skip Jekyll
└── README.md        ← This file
```

---

## License

MIT — free to use, fork, and modify.
