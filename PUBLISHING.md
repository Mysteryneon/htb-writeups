# Publishing

Two ways to get these online. The markdown works standalone, so you can start with
Option A today and upgrade to Option B whenever you want it to look like a real blog.

---

## Option A — Public GitHub repo (do this first)

Zero extra tooling. The write-ups render natively on GitHub, and the repo itself
is a portfolio piece.

```bash
cd htb-writeups
git init
git add .
git commit -m "Initial commit: repo scaffold + Writeup"

# create an empty repo named 'htb-writeups' on GitHub, then:
git remote add origin git@github.com:<your-username>/htb-writeups.git
git branch -M main
git push -u origin main
```

Each box's `README.md` renders automatically when someone opens the folder.
Link the repo from your LinkedIn / CV.

---

## Option B — Rendered blog via GitHub Pages + Hugo

Turns the same markdown into a styled site (like the box author's own write-up blog).

1. **Install Hugo** (extended):
   ```bash
   # macOS:  brew install hugo
   # Debian/Kali:  sudo apt install hugo
   ```

2. **Create the site and add a theme** (PaperMod is clean and security-blog friendly):
   ```bash
   hugo new site site && cd site
   git init
   git submodule add https://github.com/adityatelange/hugo-PaperMod themes/PaperMod
   echo 'theme = "PaperMod"' >> hugo.toml
   ```

3. **Point content at your write-ups.** Put each box under `site/content/posts/`,
   e.g. `site/content/posts/writeup.md`, with front matter at the top:
   ```markdown
   ---
   title: "HackTheBox — Writeup"
   date: 2026-08-14
   tags: ["htb", "linux", "easy", "cmsms", "path-hijack"]
   ---
   ```
   (Everything below the front matter is the existing write-up body.)

4. **Preview locally:**
   ```bash
   hugo server -D
   # open http://localhost:1313
   ```

5. **Deploy on GitHub Pages** — add `.github/workflows/hugo.yml` (Hugo's official
   Pages workflow: https://gohugo.io/hosting-and-deployment/hosting-on-github/),
   push, and enable Pages → *GitHub Actions* in the repo settings.

Your site lands at `https://<your-username>.github.io/<repo>/`.

---

## Keep private work private

Anything real — client engagements, lab configs, unredacted flags, credentials —
goes in a **separate private repo**, never here. The `.gitignore` in this repo
already excludes common sensitive paths (`private/`, keys, `.ovpn`, loot), but the
hard rule is: public repo = retired boxes only.
