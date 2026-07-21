# HallHop

> A Chrome extension that turns the school hall pass into a one-click, schedule-aware check-in / check-out system for students.

![Chrome Extension](https://img.shields.io/badge/Chrome-Extension-4285F4?logo=googlechrome&logoColor=white)
![Manifest V3](https://img.shields.io/badge/Manifest-V3-brightgreen)
![JavaScript](https://img.shields.io/badge/JavaScript-ES%20Modules-F7DF1E?logo=javascript&logoColor=black)
![Status](https://img.shields.io/badge/status-testing-orange)

HallHop is a Manifest V3 Chrome extension that lets a student log a hall-pass "check out" and "check in" straight from the browser toolbar. It signs in with the student's **Home Access Center (HAC)** credentials, figures out which class they're currently in from their schedule, and records each trip — with a live timer — to a backend service. It supports families with multiple students and automatically works out the right period using A/B-day logic and bell-schedule times.

## Table of contents

- [How it works](#how-it-works)
- [Features](#features)
- [Architecture](#architecture)
- [Project structure](#project-structure)
- [Installation](#installation)
- [Usage](#usage)
- [Backend API](#backend-api)
- [Configuration](#configuration)
- [Privacy & credentials](#privacy--credentials)
- [Development notes](#development-notes)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

## How it works

1. The student opens the extension popup and enters their HAC username and password.
2. HallHop calls its backend API to fetch, in parallel, the student's info, schedule report, active student, and student list.
3. Using the current date/time, an A/B-day calculation, and the bell schedule, HallHop determines the current period, class, teacher, and room — and even the lunch designation based on the building number.
4. The student taps **Check Out** to start a timer and log the trip; tapping again checks them back in and records the duration.
5. Sessions are cached in `chrome.storage.local` for up to 30 minutes so the popup can restore state (including an in-progress checkout) without re-login.

## Features

- **HAC login** with session restore (30-minute window) via `chrome.storage.local`.
- **Automatic schedule detection** — current period/class from an A/B-day calculation plus bell-schedule time slots.
- **Lunch designation** inferred from the room's building number for 3rd/7th periods.
- **Check-out / check-in timer** with a live "Checked Out (Xm Ys)" label and duration logging.
- **Multi-student support** — an avatar dropdown to switch between students, with a confirmation modal.
- **School-hours guardrails** — checkout is blocked on weekends, before ~8:45 AM, and after ~4:20 PM.
- **Polished popup UI** — cursor-follow header gradient, card hover gradients, button ripple effects, loading overlay, and modals.
- **Background service worker** that pings the API on a recurring alarm to keep the backend warm.

## Architecture

The popup uses native ES modules with a single entry point (`js/popup/init.js`) that wires the pieces together:

| Module | Responsibility |
| --- | --- |
| `js/popup/init.js` | Entry point — boots auth, UI, dropdown, schedule, and checkout in order. |
| `js/popup/auth.js` | Login form handling and session restore from storage. |
| `js/popup/api.js` | All backend calls (login, schedule report, student list/switch, checkout/checkin logs). |
| `js/popup/schedule.js` | A/B-day + bell-schedule logic to compute the current class and lunch. |
| `js/popup/checkout.js` | Check-out/check-in flow, timer control, and school-hours restrictions. |
| `js/popup/dropdown.js` | Student-switching dropdown, highlighting, and confirm modal. |
| `js/popup/ui.js` | Rendering helpers, animations, show/hide, and debug logging. |
| `modules/timer.js` | Start/stop timer and duration formatting. |
| `js/background.js` | Service worker: install hooks, message handling, and keep-alive ping alarm. |

## Project structure

```
HallHopProctor/
├── manifest.json          # MV3 manifest (name, permissions, background worker)
├── popup.html             # Popup markup (login form, schedule card, modals)
├── icon.png               # Toolbar icon
├── css/
│   └── styles.css         # Popup styling
├── js/
│   ├── background.js       # Service worker (alarms, ping)
│   └── popup/
│       ├── init.js         # Popup entry point
│       ├── auth.js         # Login + session restore
│       ├── api.js          # Backend API calls
│       ├── schedule.js     # Class/period + A/B-day logic
│       ├── checkout.js     # Check-out/in + timer + guardrails
│       ├── dropdown.js     # Student switcher
│       └── ui.js           # UI helpers + animations
└── modules/
    └── timer.js            # Timer utilities
```

## Installation

HallHop is not packaged for the Chrome Web Store, so load it as an unpacked extension:

1. Clone or download this repository.
   ```bash
   git clone https://github.com/RithulBhat/HallHopProctor.git
   ```
2. Open `chrome://extensions` in Chrome.
3. Enable **Developer mode** (top-right toggle).
4. Click **Load unpacked** and select the `HallHopProctor` folder.
5. Pin **HallHop** to the toolbar for easy access.

## Usage

1. Click the HallHop toolbar icon to open the popup.
2. Enter your HAC username and password and press **Check My Class**.
3. HallHop shows your current A/B day, period, class, teacher, and room.
4. Press **Check Out** when you leave — a timer starts. Press it again (**Check In**) when you return to log the duration.
5. Use the avatar in the top-right to switch between students, or **Log Out** to clear the session.

> Checkouts are intentionally disabled outside school hours and on weekends.

## Backend API

The extension talks to a companion backend (default `https://hacapi-hh.onrender.com`) that proxies Home Access Center and stores checkout logs. Endpoints used include:

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/api/getInfo` | POST | Fetch the student's name/info. |
| `/api/getReport` | GET | Fetch the schedule report. |
| `/lookup/current` | POST | Get the active student. |
| `/lookup/students` | POST | List students on the account. |
| `/lookup/switch` | POST | Switch the active student. |
| `/logs/checkout` | POST | Record a check-out. |
| `/logs/checkin` | POST | Record a check-in with duration. |

The HAC base URL is currently set to Round Rock ISD (`https://accesscenter.roundrockisd.org`).

## Configuration

- **API base URL** — change `API_BASE` in `js/popup/api.js` to point at your own backend.
- **HAC district** — update the `base_url` values in `js/popup/api.js` for a different district's Home Access Center.
- **Bell schedule & A/B reference** — the time slots and the A/B reference date live in `js/popup/schedule.js`; adjust them to match your campus.
- **Host permissions** — `manifest.json` allow-lists the backend host; update `host_permissions` if you change the API URL.

## Privacy & credentials

HallHop sends HAC credentials to the configured backend and caches them in `chrome.storage.local` to support the 30-minute session restore. This is fine for a personal/testing setup, but if you deploy this more widely you should consider tokenized sessions instead of storing raw passwords, serving the backend over a domain you control, and clearly disclosing what is stored and sent. Only use the extension with a backend you trust.

## Development notes

- Built on **Manifest V3** with a module-type service worker.
- The popup is plain ES modules — no bundler or build step required.
- Verbose debug logging is on by default (`DEBUG = true` in `js/popup/ui.js` and other modules); view it in the popup's DevTools console.
- Uses the `storage` and `alarms` permissions.

## Roadmap

- [ ] Replace stored raw passwords with short-lived tokens.
- [ ] Make the district / HAC base URL configurable from an options page.
- [ ] Turn off debug logging for release builds.
- [ ] Package for the Chrome Web Store.

## Contributing

Contributions and issue reports are welcome:

1. Fork the repository and create a feature branch.
2. Keep each popup concern in its own module under `js/popup/`.
3. Test by loading the unpacked extension and checking the console logs.
4. Open a Pull Request describing the change and how you tested it.

## License

No license file is currently included, so all rights are reserved by default. If you intend for others to reuse this code, add a `LICENSE` file (for example, the MIT License).
