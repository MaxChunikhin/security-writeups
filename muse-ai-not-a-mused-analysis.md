# Analysis: Meta Muse's "not-a-mused" Vulnerability — Why a Secure VM Wasn't Enough

*My analysis of a real, publicly disclosed vulnerability. Original discovery and credit: **Patrick Wardle** (founder, Objective-See Foundation; author of *The Art of Mac Malware*), disclosed September 21, 2026. This is my own breakdown of the vulnerability and the architectural question it raises, not a claim of original discovery.*

---

## 1. Background

Meta launched **Muse**, a personal AI agent for macOS, in September 2026. Unlike a chatbot, Muse is agentic, it's designed to act on the user's behalf: scheduling appointments, making purchases, drafting documents, and reading/writing across email, calendar, messaging, and even a linked iPhone. To do any of that, users grant it access to the microphone, camera, files, and multiple linked accounts.

The app was a hit, reportedly reaching #1 in the US App Store within days and around 2.5 million downloads in its first two weeks. Meta marketed its security posture heavily: a **"Secure VM"** (Muse's actual reasoning/compute runs in an isolated, dedicated virtual machine on Meta's own cloud infrastructure, not on the user's Mac) paired with a local component called **"Sentinel"**, meant to mediate what data flows from the user's machine up to that VM. Meta called this "first-of-its-kind" privacy and security protection, and backed it with bug bounty rewards of up to $300,000.

Thirteen days after launch, Patrick Wardle published a working zero-day.

## 2. The vulnerability

Wardle found that Muse relies on an **undocumented setting**, `endo_voyager_dictation_endpoint`, which controls where the app sends the audio of a user's dictated voice prompts. Normally, this points to Meta's own servers.

The problem: **any unprivileged local process can rewrite this setting, no special macOS permissions or entitlements required.** Once changed, the next time the user clicks the microphone and dictates a prompt, that audio (and everything Muse does with it) is sent to a server the attacker controls instead of Meta's.

**Prerequisite:** this is a *local* attack. The attacker needs to already be running code on the victim's Mac, through existing malware, or through social engineering like a [ClickFix attack](https://www.microsoft.com/en-us/security/blog/2025/08/21/think-before-you-clickfix-analyzing-the-clickfix-social-engineering-technique/) (a fake CAPTCHA that tricks a user into pasting and running a malicious command themselves). It is *not* a remote, zero-click hole on its own — that's an important nuance and worth stating clearly in any writeup, rather than overstating the threat.

## 3. What an attacker gains

Once dictation traffic is redirected, Wardle demonstrated three escalating impacts:

1. **Audio interception** — capturing the raw content of what the user says to Muse.
2. **Prompt injection** — the attacker can insert their own instructions into the stream, and Muse has no reliable way to tell attacker-supplied input apart from the real user's input. This is the general, unsolved problem with AI agents: they can't inherently distinguish trusted instructions from untrusted data.
3. **Authentication token theft** — because Muse authenticates itself to act on the user's behalf, stealing that token hands the attacker the same standing access Muse has: email, messaging, calendar, files, camera, and (via device linking) a paired iPhone's location.

Wardle's public proof-of-concept (`not-a-mused`, on GitHub) demonstrated writing malicious files to disk, taking photos through the Mac's camera, and pulling the real-time location of a linked iPhone, showing the reach extends past the compromised Mac itself.

## 4. The architectural question

What makes this vulnerability interesting to me is that the Secure VM itself did not really fail. Meta put a lot of effort into isolating Muse's reasoning, credentials, and tools inside its cloud environment. Sentinel is supposed to act as the permission authority for connector actions and network egress inside that environment. The problem is that this attack happened before those protections really mattered. The vulnerable dictation setting lived on the user's Mac, and an unprivileged local process could change where Muse sent the user's voice input. If the input is redirected before it ever reaches the trusted Meta infrastructure, a heavily secured VM on the other end does not protect that part of the data flow.

That makes this more of a **trust-boundary problem** than a failure of virtualization itself. Meta secured the cloud environment where the agent operates, but a security-sensitive piece of local configuration was still outside that protection. The design assumed the local client would send input to the intended transcription service, but the destination itself could be changed by software that did not need elevated privileges. Once that assumption failed, the protections deeper in the architecture could be bypassed.

A comparison that makes sense to me would be putting a web application behind a strong firewall while allowing any local program to silently change the system's proxy settings. The server can be secure, but that does not help if an attacker can redirect the traffic before it ever reaches the server. DNS hijacking has a similar shape: the real destination may be trustworthy, but the security model breaks if the client can be tricked into sending sensitive traffic somewhere else.

The first fix I would want is to prevent ordinary unprivileged processes from changing a setting that controls where sensitive dictation traffic is sent. Muse should also verify that transcription traffic is going to an expected Meta-controlled endpoint instead of trusting a locally writable value. Authentication tokens should be scoped as narrowly as possible so compromising one service does not automatically hand over control of the entire agent. Another improvement would be keeping transcription on-device when possible. Wardle pointed out that macOS already supports local transcription, which would have removed this specific redirection path entirely.

Meta later said it released a hotfix that patched the zero-day. That addresses the immediate vulnerability, but the larger lesson is architectural: security controls around the agent's cloud environment only work if the local inputs, configuration, and paths into that environment are treated as part of the same trust boundary.

## 5. Takeaway

My biggest takeaway is that isolating the AI model is not the same thing as securing the entire AI agent. An agent like Muse has access to so many parts of a user's digital life that every component feeding data into it or acting on its behalf becomes part of the security boundary. If one relatively small local setting can redirect trusted input and expose that access, the rest of the architecture can be well designed and the system can still have a serious weakness.

---

**Sources / further reading:**
- Patrick Wardle's original disclosure thread on X, and his [`not-a-mused` proof-of-concept on GitHub](https://github.com/pwardle/not-a-mused)
- [Privacy Guides: "Meta's Muse AI Assistant Vulnerable to Hijacking Via Undocumented Setting"](https://www.privacyguides.org/news/2026/09/22/metas-muse-ai-assistant-vulnerable-to-hijacking-via-undocumented-setting/)
- [Ars Technica: "Muse, Meta's extraordinarily privileged AI assistant, has a serious 0-day"](https://arstechnica.com/security/2026/09/muse-metas-extraordinarily-privileged-ai-assistant-has-a-serious-0-day/)
- [Meta AI Research: "How We Built Safety Into Muse"](https://research.meta.ai/blog/security-and-safety-for-ai-agents-our-approach-with-muse)
- Additional coverage: 9to5Mac, TechRadar, ITNews.com.au

**Status note:** Ars Technica updated its September 21 report to state that Meta released a hotfix patching the zero-day. Because this is a fast-moving issue, the disclosure and patch status should still be rechecked before publishing or substantially revising this analysis.
