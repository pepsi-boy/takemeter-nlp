# TakeMeter — Planning

## Community

I'm using r/soccer. It's one of the most active sports communities on Reddit, and the discourse is all over the place — you get people posting detailed tactical breakdowns right next to someone losing their mind over a last-minute goal. That range makes it a good fit for a classification task because the differences between post types are real and meaningful, not manufactured.

## Labels

**analysis** — the post builds an actual argument using stats, tactics, historical context, or structured reasoning. The claim is supported, not just stated.
- *"United's pressing has completely broken down this season — their PPDA is 12.3, worst in the top half of the table, compared to 8.1 last year under the same system."*
- *"Rodri's value isn't in what you see, it's in what doesn't happen. City concede 0.4 fewer xG per 90 when he plays vs. when he doesn't — that's elite."*

**hot_take** — a bold or controversial opinion stated without real supporting evidence. The post asserts something confidently but doesn't back it up.
- *"Mbappe is already better than peak Messi, it's not even close."*
- *"The Premier League is the worst-coached top league in Europe. The tactics are embarrassing."*

**reaction** — an immediate emotional response to something that just happened. No argument, just a feeling.
- *"THAT SAVE. THAT SAVE. I CANNOT."*
- *"Bro how did that not go in. I'm done. Season over."*

## Hard Edge Cases

The trickiest boundary is **hot_take vs. analysis**. A post that cites one stat to support a big claim can feel like analysis, but if the stat is cherry-picked and the post is mostly just asserting an opinion, it's a hot take.

**Decision rule:** if the evidence in the post would still support the claim even if you stripped out the opinion framing, label it analysis. If removing the stat wouldn't really hurt the argument — or the stat feels decorative — label it hot_take.

Example: *"Ronaldo is finished — he's only scored 3 goals in his last 15 games."* The stat is real but selectively chosen with no broader context. → hot_take.

I'll keep a notes column in my CSV for borderline cases and document at least 3 in detail.

**Documented hard cases from annotation:**

1. *"Pep's won the lot and he still thinks it's a trophy, I'd believe him over any redditor tbh"*
Could be **hot_take** or **analysis** — there's a logical argument being made (Pep's credentials lend weight to his view), but the reasoning is assertive rather than structured. → **hot_take**. The post isn't building an argument so much as using Pep's reputation as a shortcut.

2. *"R9 is overrated overall is my hot take, people put him in the GOAT conversation when he's had neither the peak or longevity"*
Could be **analysis** (references specific criteria: peak and longevity) or **hot_take** (no actual stats, just assertion). → **hot_take**. The criteria are vague and unverified — "neither the peak or longevity" isn't backed up by anything specific.

3. *"This is A BIT like Fergie dragging those makeshift United squads to titles. Our season has been so cursed with injuries that if we had a lesser quality manager, we would for sure not make CL semis"*
Could be **reaction** (emotional response to Arsenal's injury crisis) or **analysis** (draws a historical comparison to build an argument). → **analysis**. Despite the emotional tone, it makes a structured claim using a historical parallel as evidence.

## Data Collection Plan

- Source: r/soccer, pulling from top posts and comment threads across match threads, discussion posts, and transfer threads
- Target: ~70 examples per label (210 total) to avoid imbalance
- Collection method: manual copy-paste into a CSV with columns: `text`, `label`, `notes`
- If a label is under 20% after 200 examples, I'll go back and specifically search for posts that fit that label rather than collecting randomly

## Evaluation Metrics

I'll use **accuracy** as a baseline number, but it's not enough on its own — if one label dominates the dataset, a model can hit high accuracy by just predicting that label constantly.

I'll also report **per-class F1** for each label. F1 combines precision (of everything the model labeled "hot_take", how many actually were?) and recall (of all the real "hot_take" posts, how many did it catch?) into one score. A separate F1 per label tells me if the model is struggling with a specific class, which overall accuracy would hide. The confusion matrix will show which label pairs are getting mixed up most.

## Definition of Success

A useful classifier for this task needs to hit at least **75% overall accuracy** on the test set, with no single label's F1 below **0.65**. If one class is consistently ignored or mislabeled, the model isn't doing the job even if overall accuracy looks fine.

For deployment in a real community tool, I'd want 80%+ accuracy and F1 above 0.70 across all labels before trusting it on unseen posts.

## AI Tool Plan

**Label stress-testing:** I'll paste my three label definitions into Claude and ask it to generate 8-10 posts that sit at the boundary between hot_take and analysis (the hardest pair). If I can't cleanly label them, I'll tighten the definitions before annotating.

**Annotation assistance:** I'll use Claude to pre-label batches of ~50 posts at a time using my definitions as the prompt. I'll review and correct every single label — no skimming. I'll mark pre-labeled rows in a `pre_labeled` column in the CSV for disclosure.

**Failure analysis:** After fine-tuning, I'll paste all wrong predictions into Claude and ask it to identify patterns — things like post length, sarcasm, or specific label pairs that keep getting confused. I'll verify any patterns it spots by re-reading the examples myself before writing them up.
