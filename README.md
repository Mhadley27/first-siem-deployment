# first-siem-deployment 
Turning on the Lights
#
cybersecurity
#
devops
#
monitoring
#
security
Adding Vulnerability Detection and Compliance Auditing to My SIEM

Introduction
My name is Maryanne Hadley, and I am a cybersecurity professional driven by one core purpose — building digital environments where threats don’t stand a chance.

My path into cybersecurity was built on curiosity, determination, and an unwillingness to accept the status quo. Fascinated by the way systems communicate and where they become vulnerable, I pursued hands-on training in network security, threat intelligence, and incident management. What sets me apart isn’t just what I know — it’s how I think. I approach every system the way an attacker would, so I can defend it the way a protector must. Through rigorous study and real-world application, I developed the technical skills to monitor enterprise-level networks, identify vulnerabilities before they’re exploited, contain active threats, and translate complex security risks into clear, actionable strategies for teams at every level of an organization. response. I didn’t just want to understand technology — I wanted to protect it.
My original plan for this sprint had nothing to do with vulnerability scanning. I was going to bolt Sysmon onto my Windows endpoint and dig into process-level visibility. Somewhere along the way, a different question started nagging at me instead: forget “can I see what a process did” for a second — do I even know what's broken on these machines to begin with?
That's what pulled me toward Wazuh's Vulnerability Detection module and CIS Benchmark auditing. Instead of waiting for an attacker to do something suspicious and catching it after the fact, this is about finding the cracks before anyone gets the chance to walk through them — unpatched packages, sloppy configs, all the unglamorous stuff that actually gets companies breached. That's what hooked me, honestly. This isn't the flashy side of security. It's the janitorial side — patching, hardening, hygiene — and I wanted to see how much a SIEM could tell me about that without writing a single custom detection rule.

Setup
My environment has a handful of endpoints reporting into a central wazuh-siem server:
● USERENDPOINT — Windows Server 2022 Standard
● ad01 — Windows Server 2019 Standard
● observer — Ubuntu 24.04 LTS
● internal-fw.megaquagga.local — an internal firewall host
On paper, the plan was simple. Turn on Vulnerability Detection, get the agent installed on each machine, and run the right CIS Benchmark policy against each OS — the 2022 benchmark for USERENDPOINT, the 2019 for ad01.
In reality, just getting all my agents to show up as “active” turned into its own little saga. Three connected without a fight. The fourth — my internal firewall box — sat stuck on “never connected” no matter how long I waited, or how many times I refreshed the dashboard like that was somehow going to help. Eventually I actually logged into the host itself, and the answer was almost funny in how simple it was: the Wazuh agent service wasn't even running. Not a firewall rule. Not a DNS issue. It just never started. I re-ran the enrollment and registration process from scratch. The service came up, and the agent finally started phoning home like it should have all along.
By the end, the Endpoints dashboard looked like this:
Agent
OS
Status
USERENDPOINT
Windows Server 2022
Active
ad01
Windows Server 2019
Active
observer
Ubuntu 24.04 LTS
Active
internal-fw.megaquagga.local
—
Active (after re-enrollment)

Experiment time!
Experiment 1: Agent Deployment and Activation Verification
Why: None of the vulnerability data means anything if I can't trust that every agent is actually online and reporting in. A “never connected” agent isn't a small annoyance — it's a blind spot I might not even know I have.
How: I deployed the agent to all four endpoints, then watched the Endpoints → Agents dashboard to see who showed up, checking IP, group, and OS detection along the way.
Result: USERENDPOINT, ad01, and observer all came online right away, no drama. internal-fw. megaquagga.local didn't — it sat on “never connected” long enough that I stopped assuming it would fix itself. Logging into the host directly, I found the agent service simply wasn't running. Re-enrolling it fixed things immediately. Lesson learned: “I deployed it” and “it's actually working” are two very different sentences.
Experiment 2: Vulnerability Landscape Comparison Across Endpoints
Why: I wanted to know if the Vulnerability Detection module would give me something specific and useful per host — not just a vague “yeah, there are vulnerabilities,” but real numbers I could act on.
How: With all four agents finally reporting, I dug into the severity breakdown and the Top 5 Packages/CVEs list — first across the whole environment, then zoomed in on just USERENDPOINT.
Result: Environment-wide, the numbers were a little alarming: 569 Critical, 3,832 High, 6,292 Medium, and 605 Low severity findings. One package stood out well above everything else — linux-image-6.8.0-31-generic, with 8,147 hits. That's a pretty loud signal that the Linux host's kernel patching had fallen way behind. Zooming into just USERENDPOINT (Windows Server 2022) told a different story: 39 Critical, 1,398 High, 508 Medium, and 10 Low findings, with CVE-2024-20659 and CVE-2024-21302 near the top of the list.
Experiment 3: CIS Benchmark Compliance Scoring
Why: Vulnerability counts tell you about missing patches. They don't tell you about settings that are “working exactly as configured” but are still a bad idea. I wanted to see how my endpoints stacked up against an actual industry hardening standard instead of just assuming defaults were fine.
How: I ran the CIS Microsoft Windows Server Benchmark against both Windows boxes — v2.0.0 for USERENDPOINT, the matching version for ad01 — and compared pass/fail rates.
Result: Neither one did great. USERENDPOINT passed 94 of 359 checks — a 26% score. ad01 passed 84 of 346, coming in at 24%. What stood out was where they both failed the same way: password policy. Minimum password length, lockout threshold, lockout duration — all failed on both machines. Not a one-off mistake. That's just what these servers look like fresh out of the box, and it's a little unsettling how consistent that was.

Conclusion
Flipping on Vulnerability Detection and CIS Benchmark auditing turned an environment I'd have called “basically fine” into one with a real, prioritized list of problems: thousands of CVEs, a Linux kernel package that clearly hadn't been patched in a while, and two Windows servers that couldn't clear a quarter of a standard hardening checklist. None of that took custom detection rules or clever engineering. It just took turning the right features on — and, as it turns out, actually double-checking that every host was reporting in the first place.

Final thoughts
The part of this project that actually stuck with me wasn't the scary CVE counts. It was that internal-fw agent sitting on “never connected” for way longer than I want to admit before I bothered to check whether the service was even running. Such a small, dumb thing — but a good reminder that visibility gaps in security tooling are usually not some clever misconfiguration. They're just a service nobody started. “Is this thing actually running” is now permanently at the top of my troubleshooting checklist, right above anything fancier.

References
● Wazuh Vulnerability Detection documentation
● Wazuh Agent Enrollment and Registration documentation
● CIS Benchmarks — Microsoft Windows Server 2022 Benchmark v2.0.0
● CIS Benchmarks — Microsoft Windows Server 2019 Benchmark v2.0.0
