# 🔍 JobFinder

A clean, fast, frontend-only job search aggregator. No backend, no API keys, no auth — just instant search links across 8 major job platforms.

**[Live Demo →](https://olamidedurodola.github.io/job-finder)**

---

## Features

- 🎯 **Multi-board search** — LinkedIn, Indeed, Glassdoor, ZipRecruiter, Monster, SimplyHired, CareerBuilder, Google Jobs
- 🏢 **Company filter** — optionally target a specific company across all boards
- 📤 **WhatsApp share** — pre-filled message with role, location, company & link
- 📧 **Email share** — mailto with subject + body pre-filled
- 📋 **Copy link** — one-click URL copy per platform
- 🚀 **Open All** — launch all 8 boards in separate tabs
- 🕒 **Recent searches** — saved in localStorage, click to reload
- 🌙 **Dark mode** — respects system preference + manual toggle
- ⌨️ **Keyboard friendly** — press Enter to search from any field

## Usage

No installation needed. Open `index.html` directly in your browser, or serve with any static file host.

```bash
# Serve locally with Python
python -m http.server 3000

# Or with Node
npx serve .
```

## Tech Stack

- Pure HTML + Vanilla JavaScript
- [Tailwind CSS](https://tailwindcss.com) via CDN
- Google Fonts (Inter)
- No build step required

## Deployment

Works on any static hosting:
- GitHub Pages
- Netlify (drag & drop the folder)
- Vercel

## License

MIT
