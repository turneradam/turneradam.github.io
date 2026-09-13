# Deploying adamhturner.com

Static HTML on GitHub Pages, domain registered at Porkbun. No build step, no
dependencies, no CI. Roughly 20 minutes of work plus up to an hour of waiting
for DNS and the TLS certificate.

---

## 0. Before you start

Two decisions to settle first.

**The group page still has placeholder tables.** An empty alumni table under a
heading claiming 56 completed theses is worse than no table. Either populate
eight to twelve entries with permission, or delete both tables and keep the
prose counts plus the collaborator list. Deleting takes 30 seconds and the page
stands fine without them.

**Check whether the repo already exists.** There was an earlier attempt at this
in March 2026. If `turneradam.github.io` is already on your account with old
content in it, decide now whether to wipe it or start a fresh repo under a
different name.

```
# lists your repos, needs the gh CLI; otherwise just look on github.com
gh repo list turneradam --limit 100 | grep -i "github.io\|website\|site"
```

---

## 1. Preview locally

Before anything touches the network, look at it in a browser.

```
cd ~/proj/adamhturner.com          # wherever you unpacked the files
python3 -m http.server 8000
```

Open `http://localhost:8000`. Click every nav link. Resize the window narrow to
check the mobile layout. Tab through the page to confirm focus outlines appear.
`Ctrl-C` to stop.

Fix anything wrong now. Pushing broken markup and then fixing it leaves a
messy first commit.

---

## 2. Create the repository

On github.com, create a new repository:

| Field | Value |
|---|---|
| Name | `turneradam.github.io` |
| Visibility | **Public** |
| Initialise with README | No |
| Add .gitignore | No |
| Add licence | No |

The name matters. A repository named `<username>.github.io` is treated as a
*user site* and serves from the repository root of the default branch, which is
the simplest possible arrangement. Any other name becomes a *project site* and
serves from a subpath, which adds complications you do not need.

Public visibility is required for Pages on a free account.

---

## 3. First push

```
cd ~/proj/adamhturner.com

# keep your personal address out of the commit history
git config --global user.name "Adam Turner"
git config --global user.email "turneradam@users.noreply.github.com"

git init
git branch -M main
git add -A
git commit -m "Initial site"
git remote add origin git@github.com:turneradam/turneradam.github.io.git
git push -u origin main
```

If you have not set up SSH keys on this machine, either do that or swap the
remote for HTTPS and authenticate with a personal access token:

```
git remote set-url origin https://github.com/turneradam/turneradam.github.io.git
```

### Files that should be in the commit

```
404.html
CNAME
group.html
index.html
publications.html
research.html
style.css
DEPLOY.md          (this file, optional)
```

Nothing else. No `.DS_Store`, no editor backups, no `node_modules`.

---

## 4. Add a .nojekyll file

GitHub Pages pipes everything through Jekyll by default, which silently ignores
any file or directory whose name starts with an underscore. Nothing here starts
with an underscore today, but a future `_drafts` or `_images` directory would
vanish without explanation. One empty file prevents that permanently and makes
builds quicker.

```
touch .nojekyll
git add .nojekyll
git commit -m "Skip Jekyll processing"
git push
```

---

## 5. Enable Pages

Repository → **Settings** → **Pages** in the left sidebar.

| Field | Value |
|---|---|
| Source | Deploy from a branch |
| Branch | `main` |
| Folder | `/ (root)` |
| Custom domain | `adamhturner.com` |
| Enforce HTTPS | **Leave unticked for now** |

Click Save after setting the branch, then again after entering the domain.

Within a minute or two the site should be live at
`https://turneradam.github.io`. Confirm that before touching DNS. If it is
broken here, DNS will not fix it.

### A note on the CNAME file

The `CNAME` file in the repository and the "Custom domain" field in Settings
are two views of the same thing. Entering the domain in Settings writes the
file; editing the file updates the setting. Never delete the `CNAME` file, and
never let a deployment overwrite it, or the custom domain silently detaches and
the site falls back to the `github.io` address.

---

## 6. Porkbun DNS

Porkbun dashboard → **Domain Management** → `adamhturner.com` → **DNS**.

### Delete first

Porkbun creates parking records on registration. Remove every existing `A`,
`ALIAS` and `CNAME` record at the apex and at `www` before adding your own.
Leftover parking records will fight yours and produce intermittent failures
that are miserable to diagnose.

Leave `MX`, `TXT` and `NS` records alone.

### Then add

| Type | Host | Answer | TTL |
|---|---|---|---|
| ALIAS | *(leave blank)* | `turneradam.github.io` | 600 |
| CNAME | `www` | `turneradam.github.io` | 600 |

Porkbun supports `ALIAS` at the apex, which is the cleanest option: it tracks
GitHub's addresses automatically if they ever change.

### Fallback if ALIAS misbehaves

Four `A` records instead, all with a blank host:

| Type | Host | Answer | TTL |
|---|---|---|---|
| A | *(blank)* | `185.199.108.153` | 600 |
| A | *(blank)* | `185.199.109.153` | 600 |
| A | *(blank)* | `185.199.110.153` | 600 |
| A | *(blank)* | `185.199.111.153` | 600 |
| CNAME | `www` | `turneradam.github.io` | 600 |

These four addresses are GitHub's published Pages endpoints. They change very
rarely, but if you use `A` records you own the problem when they do.

---

## 7. Wait, then verify

Propagation on Porkbun is usually under 30 minutes. Check from the command
line rather than guessing:

```
dig +short adamhturner.com
dig +short www.adamhturner.com
```

The apex should resolve to the four GitHub addresses (an `ALIAS` resolves to
the same `A` records behind the scenes). The `www` lookup should show
`turneradam.github.io` followed by the same addresses.

Then check the site actually serves:

```
curl -sI http://adamhturner.com | head -3
```

A `200` or a `301` redirect to HTTPS both mean it is working.

---

## 8. Turn on HTTPS

Go back to **Settings → Pages**. Once DNS resolves, GitHub requests a Let's
Encrypt certificate automatically. This usually takes 10 to 30 minutes but can
occasionally take hours.

While it is pending you will see a note that the certificate is being
provisioned. **Wait for that to clear, then tick "Enforce HTTPS".**

Ticking it early does no permanent harm but the site will throw certificate
warnings to any visitor until provisioning completes, which is the opposite of
what you want during an application window.

---

## 9. Final checks

Work through these on the live domain, not on localhost:

| Check | How |
|---|---|
| Apex loads over HTTPS | `https://adamhturner.com` |
| `www` redirects to apex | `https://www.adamhturner.com` |
| Every nav link works | Click all four on each page |
| 404 page appears | `https://adamhturner.com/nonsense` |
| External links resolve | ORCID, GitHub, LinkedIn, YouTube, both TV5 |
| Mobile layout holds | Load it on your phone, not a resized browser |
| Link preview looks right | Paste the URL into Slack or a draft email |

The last one matters more than it sounds. The `og:` tags in `index.html`
control what a recruiter sees when someone forwards your link.

---

## 10. Updating later

Three commands from anywhere, including Termux on the phone:

```
cd ~/proj/adamhturner.com
# edit files
git add -A && git commit -m "Add alumni table" && git push
```

The live site updates within a minute. There is no build to run and nothing to
watch. If a change does not appear, hard-refresh with `Ctrl-Shift-R`; the
browser cache is the usual culprit rather than the deployment.

Set a calendar reminder to revisit the "Last updated" line in the footer every
six months. A site dated two years ago reads as abandoned even when the content
is current.

---

## 11. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| "Domain does not resolve to the GitHub Pages server" | DNS not propagated, or parking records still present | Wait 30 min; re-check Porkbun for leftover apex records |
| Site loads unstyled | `style.css` missing from the commit, or a path typo | `git ls-files` to confirm it was pushed |
| Certificate warning in browser | HTTPS enforced before provisioning finished | Untick Enforce HTTPS, wait, re-tick |
| Custom domain field keeps emptying | `CNAME` file absent from the repository | Restore it, commit, push |
| Apex works, `www` does not | Missing or wrong `www` CNAME record | Add `www` → `turneradam.github.io` |
| Certificate stuck on "new" for over 24h | Known intermittent GitHub fault | Remove the custom domain, save, wait 10 min, re-add it |
| Changes not appearing | Browser cache | `Ctrl-Shift-R`, or check the Actions tab for a failed deploy |

---

## 12. Housekeeping

| Item | Why it matters |
|---|---|
| Porkbun auto-renew ON | A lapsed domain gets re-registered by squatters within hours |
| TOTP two-factor on Porkbun and GitHub | SMS is weak, and worse once your number changes on relocation |
| Recovery email is personal, never institutional | Your Ateneo address stops working when you leave |
| Calendar reminder to check the card on file | Auto-renew fails silently on an expired card |

The domain outlives every institution you will work at. Treat the registrar
account as long-term infrastructure rather than a one-off purchase.
