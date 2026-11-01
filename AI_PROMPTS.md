# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): Yes, heavily.

**Which ones?** (e.g. ChatGPT, Claude, Copilot, Cursor): Claude (Anthropic), as
an agent that could run commands. I worked through the challenge with it step by
step: it ran builds and tests in a sandbox, I reviewed every change and its
explanation, then ran everything myself locally and on GitHub Actions.

> If you answered **no**, write "N/A" in sections 1 and 2 and skip to
> section 3. That is a perfectly good answer.

---

## 1. Key prompts

List the main prompts you used. Paraphrasing is fine.

1. `Here is the challenge repo. Let's do it together at the highest level: go through every step and explain exactly what to do and why.`
2. `Reproduce each defect before fixing it and keep the real error output as evidence for the RCA. Fix one defect per commit.`
3. `The tests must pass on every run - don't touch the tests or app.py. What are the options for waiting on the backend, and why is a healthcheck better than sleep/retries?`
4. `Why does docker stop take 10 seconds and the backend exit with 137?`
5. `Bonus: make the proxy image minimal and non-root, harden it, and verify every security claim against the running container (headers, uid, capabilities, read-only fs).`

---

## 2. Where the AI got it wrong

Describe **one concrete case** where the AI's answer was wrong, outdated, or
sub-optimal. Cover: what it suggested, how you noticed it was wrong, and what
you did instead. 3–5 sentences.

> For the hardened proxy image, Claude added `RUN nginx -t` to the Dockerfile so
> that an invalid config would fail at build time instead of at deploy time. It
> sounds like best practice, but the build immediately failed with
> `[emerg] host not found in upstream "backend:8080"`: `nginx -t` resolves upstream
> hostnames, and `backend` only exists in DNS on the Compose network at runtime,
> not during `docker build`. We removed the step and rely on the Compose
> healthcheck instead (nginx with a bad config never becomes healthy, so the
> pipeline still fails fast). A second, subtler one: after fixing the
> executable bit with `git update-index --chmod=+x`, a later `git add -A` silently
> re-staged mode `644` from the working copy. We caught it only because a test run
> returned exit 126 again; `git show --summary` showed `mode change 100755 => 100644`.

---

## 3. Anything in your submission you could not fully explain?

We would rather know. Write "nothing" if that is the case.

> Nothing.
