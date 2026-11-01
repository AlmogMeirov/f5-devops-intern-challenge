# Generative AI Disclosure

Using AI tools for this challenge is allowed. Be honest here — an accurate
"I used AI heavily" is worth more to us than an inaccurate "I did not".

**Did you use AI tools?** (yes / no): Yes, heavily.

**Which ones?** 
Claude (Anthropic), used as a supporting development and debugging tool. I led the problem-solving process, identified what needed to be investigated, directed the debugging steps, and made the decisions on how to approach and resolve each issue. I used Claude to assist with running commands, analyzing errors, and validating potential solutions in a sandbox environment. I reviewed and understood every change before applying it, then independently verified the final solution through local builds, tests, and GitHub Actions.

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

While hardening the proxy image, Claude suggested adding RUN nginx -t to the Dockerfile to catch configuration errors during the build. When I tested this, the build failed with host not found in upstream "backend:8080". I investigated the failure and realized that Nginx was trying to resolve the backend hostname during the image build, even though that hostname is only available within the Docker Compose network at runtime. I decided to remove the build-time check and rely on runtime health checks and integration tests to verify that the proxy starts correctly and can communicate with the backend.

---

## 3. Anything in your submission you could not fully explain?

We would rather know. Write "nothing" if that is the case.

> Nothing.
