# Deploying from Android / Termux

Replacing the contents of the existing `turneradam.github.io` repository with
the new site, working entirely from the phone. Git history is preserved; only
the files change.

Expect 20 to 30 minutes, most of it waiting for `pkg` to download.

---

## 1. Termux setup

Skip any step where the tool is already installed.

```
pkg update && pkg upgrade -y
pkg install -y git gh openssh unzip nano
```

Grant Termux access to the phone's shared storage. Android will show a
permission dialogue; accept it.

```
termux-setup-storage
```

This creates `~/storage/`, with `~/storage/downloads` pointing at the folder
where your browser saves files. That is how the site archive gets from the
Downloads folder into the repository.

---

## 2. Authenticate with GitHub

```
gh auth login
```

Answer as follows:

| Prompt | Answer |
|---|---|
| What account do you want to log into? | GitHub.com |
| Preferred protocol for Git operations | HTTPS |
| Authenticate Git with your GitHub credentials? | Yes |
| How would you like to authenticate? | Login with a web browser |

`gh` prints an eight-character device code and waits. Copy it, tap the
github.com/login/device link it shows, paste the code, approve. Return to
Termux; it will have continued on its own.

HTTPS rather than SSH here on purpose. `gh` configures the git credential
helper for you, so pushes need no further authentication and you avoid
generating and registering a key from a phone keyboard.

Confirm it worked:

```
gh auth status
```

---

## 3. Clone the existing repository

```
cd ~
gh repo clone turneradam/turneradam.github.io
cd turneradam.github.io
```

---

## 4. Look at what is already in there

Do this before deleting anything.

```
ls -la
git log --oneline | head -20
```

Things worth keeping if present:

| File or directory | Keep? |
|---|---|
| `.git/` | Always. Never delete this |
| `LICENSE` | Yes, if you added one deliberately |
| `.github/workflows/` | Only if you know what the workflow does |
| `CNAME` | The new archive contains its own; either is fine |
| Old HTML, CSS, JS, images | No. These are the March attempt |
| `node_modules/`, `package.json`, Tailwind config | No |

If the repo is empty apart from a README, nothing needs removing and you can
skip straight to step 6.

---

## 5. Clear out the old files

This removes tracked files from the working directory but leaves `.git`
untouched, so history survives intact.

```
git rm -r --quiet .
ls -la
```

`ls` should now show little beyond `.git`. If a stray untracked file survived:

```
git clean -fd
```

**If you decided to keep something in step 4**, restore it before committing:

```
git checkout HEAD -- LICENSE
```

---

## 6. Get the new site onto the phone

Download `adamhturner-site.zip` from this conversation using the phone's
browser. It lands in the Downloads folder.

```
cd ~/turneradam.github.io
unzip ~/storage/downloads/adamhturner-site.zip
ls -la
```

You should see:

```
404.html  CNAME  DEPLOY.md  DEPLOY_TERMUX.md
group.html  index.html  publications.html  research.html  style.css
```

If `unzip` reports the file is missing, check the exact filename:

```
ls ~/storage/downloads | tail -5
```

Some browsers append `(1)` or save to a different folder.

---

## 7. Add the .nojekyll file

GitHub Pages runs everything through Jekyll by default, which silently ignores
files and folders beginning with an underscore. Nothing here starts with one
today, but this prevents a future surprise and makes builds quicker.

```
touch .nojekyll
```

---

## 8. Commit and push

```
git config user.name "Adam Turner"
git config user.email "turneradam@users.noreply.github.com"

git add -A
git status
```

Read the `git status` output before committing. Deletions of old files and
additions of new ones are both expected. Anything unexpected, stop and look.

```
git commit -m "Replace site with hand-coded static version"
git push
```

---

## 9. Check the Pages settings

Open github.com in the phone browser → repository → **Settings** → **Pages**.

| Field | Value |
|---|---|
| Source | Deploy from a branch |
| Branch | `main` (or `master`, whichever the repo uses) |
| Folder | `/ (root)` |
| Custom domain | `adamhturner.com` |
| Enforce HTTPS | Tick only once the certificate has issued |

If the branch is `master` rather than `main`, leave it alone. Renaming a branch
from a phone is not worth the risk.

Give it a minute, then load `https://turneradam.github.io`. Confirm the new
site appears there before worrying about the custom domain.

---

## 10. DNS

Covered in `DEPLOY.md`, sections 6 to 8. Summary: delete Porkbun's parking
records, then add

| Type | Host | Answer | TTL |
|---|---|---|---|
| ALIAS | *(blank)* | `turneradam.github.io` | 600 |
| CNAME | `www` | `turneradam.github.io` | 600 |

Wait for propagation, then tick Enforce HTTPS.

From Termux you can check propagation directly:

```
pkg install -y dnsutils
dig +short adamhturner.com
```

---

## 11. Later edits from the phone

```
cd ~/turneradam.github.io
nano group.html
git add -A && git commit -m "Add alumni table" && git push
```

`nano` on a phone keyboard is tolerable for a line or two and unpleasant for
anything larger. The group page ships with the student and alumni tables
commented out as a ready-made template, so adding them is a matter of
uncommenting and filling in rather than writing markup from scratch. Even so,
that job is better done on the MacBook.

To pull changes made elsewhere before editing:

```
git pull
```

---

## 12. Termux troubleshooting

| Symptom | Fix |
|---|---|
| `gh: command not found` | `pkg install gh` |
| `~/storage` does not exist | Run `termux-setup-storage`, accept the Android dialogue |
| Permission denied reading Downloads | Same, the permission was declined the first time |
| `git push` asks for a password | `gh auth login` again, choosing HTTPS and "authenticate Git" |
| Termux killed while downloading | Android battery optimisation. Exempt Termux in system settings |
| `unzip: cannot find` | Check the real filename with `ls ~/storage/downloads` |
| Push rejected, non-fast-forward | `git pull --rebase` then push again |
