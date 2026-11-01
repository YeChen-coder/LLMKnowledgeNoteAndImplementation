> English version of [ResumePostingWorkflow.md](../ResumePostingWorkflow.md). Original notes are written in Chinese/English mix.


Alright, let's talk about it.

Since this isn't really something with code — it's a workflow — it doesn't fit well in a repository, so I'll just rant about it here. There's a whole pile of nonsense.

First, the current overall framework: Grokbot and Claude Code handle the upfront data collection.
• Why Grokbot specifically? Because it has to watch for new postings on Indeed and LinkedIn.
• Claude Code scrapes postings from various smaller sites.

All collected raw data first goes into a scout queue on the Claude Code side for screening. Only postings that pass the screen and are correctly formatted get sent to Codex, which runs a scheduled task every 45 minutes.

Right now, every application sends a confirmation email to my inbox, so that email serves as the evidence. Of course, the inbox also has to be connected to Codex, so that Codex, as the applier, can confirm the task was really completed.

This raises the question of who monitors the inbox. (Anyone who has job-hunted will get it: once your application volume goes up, the junk mail starts carpet-bombing you.

Then there's all the shady behavior of résumé systems — some won't let you submit unless you subscribe to something. And Indeed loves sending marketing texts like "We found a few positions that are a perfect fit for you," which are just annoying — all useless stuff.

Add the rejection letters, which really affect your mood. Having a human mind absorb that kind of wear and tear isn't quite right.)

I didn't have Codex or Claude Code take over or monitor the inbox, for several reasons:

1. Claude Code:
I'd actually love for Claude Code to monitor my inbox, but first, I haven't gotten off the Duet waitlist; second, maybe I just haven't found the right place, but I can't find a way to sign in to another account's email — it defaults to my Claude Code account email, which isn't the one I apply with, and clicking connect won't let me sign in.
2. Codex:
It can connect to my other inboxes, but its computer use burns way too many tokens, and my token quota is really tight.
3. Why not have Grok do it?
Because my family also uses Grok, and I'd rather they not see these messages — better to keep things separated.

So in the end the job went to the free Muse. But Muse has its own pile of issues:
• Scheduled tasks: if I set them to run more often, for some reason it doesn't execute every time, and it doesn't tell you when it skips.
• Reading content: it doesn't read everything; it matches on keywords. At first, if an email subject said "congratulations" or "next step" or the like, it would read it and notify me. Later I told it to read everything, and it still doesn't read the body — only the subject line. Right now it relies entirely on email subjects.

This could probably be optimized further, but I'm just too tired to keep fiddling. If something comes up, I'll ask it myself.

To keep context manageable, and so I don't have to archive a pile of sessions auto-created by the scheduled task every day, I capped applications at 3 per session.

This was forced on me too: Codex's tokens run out way too fast, and early on they even quietly changed the token limits. At first it wasn't this complicated — very simple, Codex did everything itself, finding postings and applying. But later, because of usage limits, I had to hand the upfront data collection to other AI bots with spare token capacity.

The filter has also gone through many revisions: at first the scope was too narrow and it barely found any jobs; after I changed it, the scope was inaccurate and way too broad; then after repeated tweaking, I finally got it to a reasonable scale.

As for why everything has to go through the SQS queue on the Claude Code side: it's mainly to save Codex tokens, so Codex doesn't have to search again because a URL was wrong, unreachable, or missing.

Anything that enters the queue via Claude Code is properly formatted and can be safely handed off to Codex.

Codex actually has one more layer, too.

I added it because I noticed that when Grok finds postings, it sometimes can't capture the full job description — only the part shown on the card. Only when Claude Code actually opens the page can you see the full job description. Previously, it was applying me to purely senior roles that required French or 8 years of experience, which I thought was pointless.

So I added another filter on the Codex side: look at the full job description, and if it mentions something I simply can't do (like French or 8 years of experience), don't apply; but roles asking for 3–5 years are fine to apply to. AI is sometimes too literal, and precisely because of that, the screening scope needs to be loosened a bit — widened moderately.

Another thing I really want to rant about: these days, basically 95% of postings make you fill in a huge pile of boxes. Gender, gender identity, select the race, plus things like "Are you identified as having a disability?", "Do you identify as a member of Two-Spirit / Lesbian / LGBT?", and so on.

Uploading your résumé isn't enough — its parsing is probably wrong, so you have to go through it again to check. Then comes a whole other pile of questions: willing to relocate, what's your expected salary… all sorts of messy, repetitive questions that just pointlessly drain the applicant's energy.

In the future, can we make it so that at least every job seeker has an AI twin, and every recruiter has one too, and the two AIs connect and sort things out themselves, instead of wasting a person's energy?

Let's stop wearing each other down — if you need anything, please talk to my AI first. I sincerely think this might be one form of future society.

Back to the point: why haven't I written about this before, and why start today? Because Codex just made me really mad — it somehow got on my nerves.

While it was running a task just now, its built-in browser had problems for a few minutes. That's fine — I've got Chrome, and Codex has computer use, so with my browser open it could always log in itself. But it told me it went to find Edge, Edge returned "tool unavailable," and it just stopped. — A rant: I think something's off with Codex this week. This setup has been running on my end for at least a full month, and before this it basically never called whole-computer computer use, never even called Edge or Chrome — the built-in browser handled everything. But this week it's started constantly reaching for whole-computer computer use.

The problem is, my prompt says it clearly: take 3 postings from the queue and apply, and there are at least 10 to 12 in the queue. After hitting an error on the first one, it stopped. So I looked into why it stopped: even if the first one errors — it always happens — but for the other two, why did you suddenly stop?

So I asked Codex, and Codex returned me the error response from Edge. And that is where the problem is lying: that error message is sent by Edge, but my prompt is sent as a user. Of course I have more priority, but Codex is taking the message from Edge as a cut signal.

Afterward I wrote this into Run.md. Run.md is a file maintained jointly by Claude Code and Codex; it spells out where my latest résumé is, my personal details, and content that needs to be added dynamically.

And there are other problems — truly a whole pile.

Some sites ask you to self-assess your skills, and I just watched Codex, on one application site, fill in my Large Language Model, MLOps, and Time Series skills all as Practitioner. They're hiring an AI engineer! Which HR person would see a résumé or form like that and think it's worth moving forward? Honestly, I have a clear conscience — I think I'm pretty good at taming AI.

I'm really angry, and I have no idea why Codex did this. I also don't know how many times it pulled this kind of thing back when I let it run unattended on a schedule in the background. It's genuinely sad.

That said, I actually think my design is pretty clever.

My résumé is still evolving rapidly, but I've specified directly in README.md where the résumé lives. These days I usually have Claude Code write my résumé — not because it's convenient, but because during our conversations it can record things for me directly. I don't have to dump a huge pile of stuff on it and then go back through to see what I just learned.

That's actually pretty nice: I don't have to re-upload my résumé on the Codex page every time it's updated. If this weren't decoupled, it would be really annoying.

That's the Codex-side trouble I ran into tonight.

Thinking back, Grok-a-bot has had plenty of problems too. The one I mean here is CAPTCHAs: it's fine that Grok-a-bot hits a CAPTCHA, but it could at least call me or send me a message. It doesn't.

It actually runs as a subversion. When it hits a problem, it just grinds through the current scheduled batch, then happily trots back to tell me: "Today's batch 1 is done; got 0 jobs on Indeed because we were blocked by Cloudflare."

Every time I see that I want to swear; it gives me a headache. Being blocked by Cloudflare is fine — just tell me so I can go log in. Grok bot has both a web and a mobile app; I could hop over and log in from anywhere. I've told it countless times, "Call me if something comes up," but it uses a subagent, which can't call me. A subagent and a sub-bot aren't quite the same: a sub-bot can call me, a subagent can't.
