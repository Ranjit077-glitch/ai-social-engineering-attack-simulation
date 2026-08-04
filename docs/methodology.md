# Detailed Methodology

This document walks through the full attack simulation and forensic
investigation process step by step, including the specific tools,
configurations, and outputs at each stage. It expands on the summary in the
main [README](../README.md) for the full write-up with all analysis and
citations, see the [project report](../report/Social_Engineering_Forensic_Analysis.pdf).

All steps were carried out in an isolated virtual lab environment. No real
users, systems, or data were involved at any stage (see Disclaimer in the
main README).

---

## 1. Reconnaissance

Before building the simulation, the attack was scoped around a
social-media-based delivery model rather than traditional mass-email
phishing. The reasoning: users tend to consume social media content quickly
and with limited scrutiny, and platform trust (e.g. familiarity with
Instagram stories) can be exploited alongside visual/authority trust from a
recognizable public figure. This shaped the choice of Instagram as the
delivery channel and a public figure as the deepfake subject.

## 2. Environment setup

- Created an isolated virtualized lab using **Oracle VirtualBox**
- Installed **Ubuntu** as the guest operating system to safely contain all
  attack tooling and prevent any interaction with real networks or systems

## 3. Phishing infrastructure setup using Zphisher

- Installed **Zphisher**, an open-source phishing simulation tool, on the
  Ubuntu VM via `git clone` from its GitHub repository
- Selected the **PayPal** template from Zphisher's built-in options (30+
  templates available) to clone a realistic login page
- Zphisher generated a live tunnel URL (via Cloudflare Tunnel) pointing to
  the cloned login page, along with a local capture mechanism for any
  submitted credentials

## 4. URL masking

- The raw Zphisher-generated URL was technical-looking and would likely
  raise suspicion on inspection
- Used a **URL Masker** tool (`url-masker.vercel.app`) to disguise the real
  destination:
  - **Original URL**: the raw Zphisher/Cloudflare Tunnel link
  - **Custom Domain** field: set to `paypal.com` to make the display text
    resemble the legitimate domain
  - **Phishing Keywords** field: set to `verification`, reinforcing an
    urgency/security framing
  - **Output**: a masked link displayed as
    `https://paypal.com-verification@clck.ru/3RL98F` visually resembling a
    PayPal URL while actually redirecting to the attacker-controlled
    Cloudflare Tunnel domain
- This technique doesn't change where a link goes it changes how the link
  *looks* to a user, which is the actual mechanism being demonstrated

## 5. AI-generated deepfake video creation

- Used **HeyGen**, a web-based AI video generation platform, to create a
  synthetic video
- Selected a pre-trained digital avatar representing **Elon Musk**, chosen
  specifically for his high public visibility and perceived authority in
  the technology and financial sectors; the same reason real-world
  attackers target recognizable public figures for credibility
- Scripted the video to deliver a fabricated announcement about a "PayPal
  security upgrade," following a common real-world phishing narrative
  pattern: urgent security change, limited time window, immediate action
  required
- The video was generated using HeyGen's avatar and voice model pipeline,
  producing a realistic synthetic video output

## 6. Delivery via simulated social media

- Posted the deepfake video as a simulated **Instagram story**, with the
  masked phishing link embedded in the story
- Instagram stories were selected as the delivery mechanism specifically
  because users tend to interact with story content quickly, with less
  scrutiny than they might apply to, say, an email

## 7. Redirection and credential capture

- Clicking the masked link redirected to the Zphisher-hosted cloned PayPal
  login page, visually consistent with the video's narrative
- Test credentials were submitted into the fake login interface within the
  controlled lab environment
- Zphisher's terminal output confirmed successful capture, logging the
  victim's simulated IP address and the submitted account/password pair to
  local files (`auth/ip.txt`, `auth/usernames.dat`)

## 8. Forensic investigation

With the attack simulation complete, the investigation flipped to the
defender's perspective, examining what evidence the attack left behind at
each layer:

### 8.1 AI-content detection
- Submitted the deepfake video to the **Zhuque AI Detection Assistant**,
  which flagged it as AI-generated with **99.68% probability**, based on
  facial consistency and temporal artifact analysis
- Independently verified using **Sightengine**, which classified the video
  as **"Likely AI-generated" at 99% confidence**, with a breakdown across
  diffusion models (Sora 48%, Veo 32%, Wan 13%) and a 6% face-manipulation
  score
- Running the video through two independent detection platforms was
  deliberate cross-validating a finding against multiple tools reduces
  the risk of a false positive/negative from any single detector

### 8.2 URL and infrastructure analysis
- Analyzed the masked URL using **urlscan.io**, which resolved the
  effective destination to `newer-summary-sim-anaheim.trycloudflare.com`
  (IP `104.16.230.132`) and returned a verdict of **"Potentially
  Malicious,"** explicitly flagging it as targeting the **PayPal** brand
- For comparison, ran the same analysis against the legitimate
  `www.paypal.com`, which resolved to Fastly-hosted infrastructure
  (`151.101.193.21`) with a certificate issued by **DigiCert EV RSA CA G2**
  and a domain registrar of **MarkMonitor, Inc.** the kind of established,
  dedicated infrastructure a large legitimate service maintains, in clear
  contrast to the cloned page's generic tunnel hosting

### 8.3 DNS comparison
- Used **nslookup.io** to pull DNS records for both the cloned site and the
  legitimate PayPal domain
- The cloned site's A records pointed to generic Cloudflare tunnelling IPs
  (`104.16.231.132`, `104.16.230.132`) with no dedicated hosting
  relationship
- The legitimate PayPal domain's A records resolved through a CNAME chain
  into `paypal-dynamic.map.fastly.net` tied to PayPal's own enterprise
  CDN setup
- This comparison is a reliable, repeatable pattern for distinguishing
  attacker-hosted clones (generic, disposable tunnel infrastructure) from
  the real services they impersonate (dedicated, long-standing CDN/hosting
  relationships)

### 8.4 Source-code inspection
- Manually inspected the HTML source of the cloned PayPal login page
- Found a `<link rel="canonical" href="https://www.paypal.com/signin">` tag
  embedded in the page
- A canonical tag is normally a legitimate SEO mechanism used to tell search
  engines which URL is the "real" version of a page when duplicate content
  exists; here, it had been co-opted by the phishing kit to falsely
  reference the real PayPal domain, likely to:
  - Increase the page's perceived authenticity if a user or automated tool
    inspected the source casually
  - Reduce the likelihood of the phishing page being immediately associated
    with itself in search engine indexing, since the canonical tag points
    search engines toward the *real* PayPal page instead

---

## Key Concepts

- **Social Engineering** — manipulating human trust and behavior (rather
  than exploiting technical vulnerabilities) to gain unauthorized access or
  information. This simulation specifically exploited authority (a
  recognizable public figure), urgency (a time-limited "security upgrade"),
  and platform trust (familiarity with Instagram stories).
- **Phishing** — using a fraudulent, cloned interface (here, a fake PayPal
  login page) to harvest credentials from a target who believes they're
  interacting with the legitimate service.
- **URL masking** — a technique that alters how a link is *displayed* to a
  user without changing its actual destination, used here to make a
  malicious URL appear to originate from `paypal.com`.
- **Deepfake technology** — AI-generated synthetic media (in this case, a
  video using a cloned likeness) designed to be difficult to distinguish
  from genuine content, and increasingly used to add false credibility to
  social engineering lures.
- **Digital forensics** — the systematic process of identifying,
  collecting, and analyzing evidence left behind by an attack. This project
  applied forensics across four layers: AI-content detection, URL/DNS
  infrastructure analysis, and HTML source-code inspection demonstrating
  that even a convincing front-end lure leaves a detectable trail once you
  look beneath the surface.
