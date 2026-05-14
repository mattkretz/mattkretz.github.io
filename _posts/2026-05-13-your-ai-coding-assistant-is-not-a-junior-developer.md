---
layout: post
title: "Your AI Coding Assistant Is Not a “Junior Developer”"
tags: [LLM, AI, RSE, Software Engineering]
author: Dr. Matthias Kretz
coauthor: and Dr. Ralph J. Steinhagen
license: "[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/)"
---

Listen into a conference talk or onboarding session on AI coding assistants, and you will hear the same metaphor. The AI 
is the “junior developer” or “co-worker” on the team. You mentor it. You review its work. You trust it with routine 
tasks, not the hard problems. The framing feels natural, modest, even generous. It is also, we will argue, wrong in a 
specific and consequential way. Calling an AI a junior imports trust and patience that the system cannot earn. It trains 
teams in the wrong review habits. It risks skipping over the exact experiences that turn juniors into seniors in the 
first place.

The framing has started to travel — from this year’s International Conference on Software Engineering into team 
playbooks, onboarding documents, and procurement decisions across software, hardware, verification, control-system, and 
scientific-computing teams. Once a framing becomes invisible shorthand, so do the assumptions it carries.

This is not a philosophical complaint about language. It is a concrete issue of AI safety, AI interpretability, and AI 
dependability — three terms that, unlike the regulatory buzzword “trustworthy AI”, point at engineering problems that 
can actually be worked on.

## What the Metaphor Quietly Imports

When we call an AI system a junior, we import a whole set of well-practised habits. We grant the benefit of the doubt. 
We invest patience in “mentoring” rather than scrutiny. We extend a trust advance that we expect the system will earn 
back through visible growth. None of these are bad habits — they are how experienced scientists and engineers are 
supposed to treat newer colleagues. They are also the wrong defaults for a large language model (LLM).

The table below sketches where the analogy breaks down. The point is not to disparage either side, but to show that the 
two cases need different habits of engagement:

| Human junior colleague | Contemporary LLM coding assistant |
| ---------------------- | --------------------------------- |
| Learns from each task; over months, develops intuition and judgement. | No persistent learning between sessions; improvement happens through retraining, not experience. |
| Builds a causal model of the domain through practice and feedback. | Produces outputs from statistical associations in training data; surface coherence is not a model. |
| Errors tend to correlate with inexperience and are traceable. | Errors include confident, fluent fabrications whose distribution is hard to predict from context. |
| Accountable to peers, institutions, and professional norms. | Accountability rests with the human operator; the system has none. |
| Signals uncertainty and escalates when out of depth. | Uncertainty calibration is inconsistent and does not reliably track actual competence. |

None of this makes these tools less useful. It makes them a different kind of tool, one that needs a different set of 
operating habits — which is the whole reason words matter here.

The metaphor is not pure error. It captures something real — namely, that LLMs, like junior colleagues, produce work 
whose quality varies, benefits from review, and improves when surrounded by good engineering practice. Practitioners 
reach for the framing because some of its predictions hold up. The mistake is to treat the metaphor as a complete model 
of the relationship rather than as a partial analogy. Where it breaks down, it breaks down in directions that matter 
most for the things experienced engineers actually do — calibrating trust, sequencing learning, and verifying that the 
system understands what it is doing.

The automation-safety field has a name for what happens when we get the habits wrong. Lee and See’s 2004 analysis of 
calibrated trust in automation describes a failure mode familiar from aviation, process control, and medical decision 
support for decades: when operators over-trust a partially reliable automated system, they stop monitoring. The rare 
failures that do slip through are much more costly than they would have been without automation. The junior-developer 
metaphor is a textbook example. It is a trust-calibration mistake dressed up as professional courtesy.

## The First Problem: Why the Metaphor Feels Natural

The reason this particular frame is sticky is that the human brain does this automatically. Heider and Simmel showed in 
1944 that people spontaneously ascribe intentions, emotions, and personalities to two triangles and a circle moving on a 
screen. This is part of a well-documented bias: the human brain over-detects patterns, favouring false alarms over 
missed threats — which is why illusions like the Face on Mars vanish under better data. Experimental psychology has 
documented the same pattern over and over: when something shows even minimal social cues, humans respond socially — 
automatically, largely below awareness, and even when the observer rationally knows better.

The computing field has known this from the start. In 1966, Joseph Weizenbaum’s ELIZA — a trivial pattern-matching 
chatbot meant as a parody of Rogerian therapy — produced such strong emotional engagement from its users (including 
Weizenbaum’s own secretary) that he spent the rest of his career warning the field against confusing simulated 
understanding with the real thing.

So: personifying these systems is not naïve, and we are not going to stop doing it. Even the authors of this article 
type "please" and "thank you" to their AI assistants — not out of belief in the system's agency, but simply because that 
is our natural language for human-to-human interaction, and we use it as the interface. The question is not whether we 
personify, but which person we have in mind.

“Junior developer” is an unusually bad choice because it imports exactly the trust and patience these LLM systems cannot 
earn.

## The Tools Are Genuinely Useful

A thoughtful scientific and engineering culture sometimes does walk away from tools on principle, and is right to — 
there are technologies that should be declined, and history records several cases where principled refusal was the 
correct answer. Our views differ on whether AI belongs in that category. But we both agree that AI coding assistants can 
be used to improve development work when applied responsibly.

Used well, they are an excellent “spell-, grammar-, and thesaurus-checker on steroids”: they speed up refactoring, draft 
boilerplate, write test scaffolding, translate code from one idiom to another, and flag obvious problems. In the hands 
of an experienced practitioner — whether a scientist writing analysis code, a software engineer maintaining a framework, 
or a hardware engineer using AI assistance for design or verification — these tools offer tangible accelerations in 
routine work. While the extent of the gains varies with how they are applied, the potential for meaningful productivity 
and quality improvements is clear.

A useful analogy: most professional painting work is not the Sistine Chapel. It is walls, trim, and signage, done to 
standard by people who know their trade. A craftsman painter and Leonardo da Vinci are not better or worse w.r.t. each 
other; they are different activities, both necessary and skilled.
Today’s AI coding tools are excellent craftsman-painters, and they interpolate well within an existing body of evidence 
and practice.
They are not Leonardos — not because they are stupid, but because creating something genuinely new requires causal 
understanding, goal formation, and judgment that statistical pattern synthesis cannot supply.

Both activity classes are mixed in every real project. The mistake is not to use the craftsman; the mistake is to 
confuse the two.

This is the point we most want to make clear. AI coding tools are tools, and responsibly designed tools, used by people 
who understand their limits, make software — and the science that depends on it — better. The rest of this article is 
about what “understanding the limits” concretely requires.

## The Second Problem: The Learning-Ladder Problem

Here is where the craftsman-versus-Leonardo distinction becomes urgent rather than decorative. The skills that 
eventually separate the two — identifying invariants and boundary conditions, spotting when a problem is subtly wrong, 
knowing which part of a system is load-bearing — are not taught by lecture. They are practised painfully during routine 
craft work. A junior scientist or engineer who spends three days debugging a memory corruption or chasing why a 
simulation silently diverges learns something that cannot be transferred by reading someone else’s write-up.
The struggle is the curriculum.

If we hand the craft work to an AI before that curriculum has run, we do not accelerate a junior’s development — we 
prevent it. A growing empirical literature is documenting exactly this pattern: cognitive offloading to generative AI 
correlates with reduced engagement, shallower learning, and weaker critical-thinking measures, with de-skilling effects 
most pronounced among less-experienced users.

The parallel with homework is not just rhetorical. It is structurally the same problem. If children produce essays with 
an LLM before they have learned to write and think on the page, they do not become faster thinkers and writers — they 
simply invert the master–tool relationship, and the foundational skills never form. For junior scientists and software 
engineers, the stakes are no smaller. The cognitive habits that make a senior practitioner valuable — suspicion of 
plausible-looking wrong answers, the reflex to check invariants, the willingness to find out why something works rather 
than accepting that it does — are precisely the habits that an LLM-first workflow bypasses.

This is not an argument for keeping juniors away from these tools. It is an argument for *sequencing*: protected space 
to develop judgement first, powerful tools second. Reverse the order, and the tool becomes a substitute for the judgment 
rather than a multiplier of it.

## The Third Problem: We Don’t Know What the Model Is Optimising For

There is one more reason the junior metaphor misleads, and it comes from inside AI safety research itself. When we call 
a system a junior, we implicitly assume its goals are aligned with ours, as a new colleague’s generally are — through 
shared socialisation, professional accountability, and the simple fact that we can ask what they were trying to do and 
usually get an honest answer. None of this applies to a large neural network.

The goal structures encoded in model weights are neither directly inspectable nor guaranteed to match the objective that 
the training procedure appeared to optimise. Hubinger and colleagues named this the learned-optimisation problem in 
2019: a trained model can develop an internal objective that differs from the one it was trained on, and the mismatch 
may remain invisible until the model encounters inputs we did not test. The EU AI Act now treats this as its own 
regulatory category, separate from the application-level risks covered elsewhere — Articles 51 and 55 cover opacity, 
misaligned capabilities, and model-level cybersecurity.
The concern is not speculative. It is active foundational research with regulatory consequences, in a class of systems 
deployed daily by software, hardware, and scientific computing teams.

The junior metaphor papers over all of this. Juniors have goals we can ask about. Models have objectives we can only 
infer from behaviour, and only on the distribution of inputs we happened to test.
Treating the second as if it were the first is not a minor category error.

## What This Means in Practice

Three practical implications follow, offered as editorial markers rather than prescriptions — each community will need 
to work out the details for its domain.

Training pathways need to protect juniors' cognitive development.
Concretely, this means protected exercises where AI assistance is paused or bounded, feedback that targets reasoning 
rather than output, and assessment that distinguishes “produced a working solution” from “understands why it works”. One 
concrete instance: junior-only assignments where the AI assistant is muted for the first sprint, so the practitioner 
forms their own model of the problem before tooling shapes it. Variants for the full range of software and research 
contexts — from graduate-student analysis code to long-lived infrastructure frameworks to industrial codebases — are 
overdue.

Review and documentation practices need to account for model opacity.
Code review for AI-assisted contributions has to check for the failure modes LLMs actually produce — plausible-looking 
wrong answers, silent assumption changes, correct-looking code written against the wrong invariant — and not only for 
the failure modes a human would produce. One concrete instance: a review checklist line that asks whether the 
contributor can explain the change without consulting the AI. Provenance documentation for AI-assisted work should 
become as routine as citing a library or a dataset.

Safety- and accuracy-critical domains need explicit boundaries.
Where software drives control systems, medical devices, scientific instruments, published results, or infrastructure, 
the point at which AI involvement ends and human verification begins should be documented as any other safety- or 
correctness-relevant interface. One concrete instance: a labelled handoff in the codebase where AI-generated changes 
require human sign-off before reaching production. The EU AI Act’s distinction between systemic and application-level 
risks provides a useful framework for thinking, even for organisations not directly within its scope.

## Honest Language Makes Responsible Use Possible

AI coding tools are powerful and useful, and we are going to keep using them — in our own work, in our teams, and in the 
software infrastructure we build. That is the premise, not the question.

The question is whether we can talk about them honestly enough to use them well. The junior-developer metaphor fails in 
both directions: it underestimates human juniors by reducing their growth to output generation, and it overestimates 
machines by assuming understanding where there is only statistical competence.
Between those two errors, the practical, safe and dependable integration work becomes harder, not easier.

We do not need a perfect replacement term. We need the discipline to say what we actually mean: capable 
code-transformation and pattern-synthesis tools, extraordinarily useful inside their limits, that require operators 
trained to know where those limits are.
That is a less comfortable description than “junior colleague”. It is also the one that makes responsible engineering 
possible.
The tools we build and adopt now will shape how software — and the engineering and science that run on it — gets 
developed for a long time to come. The most useful thing we can do for the next generation of people who will rely on 
that software is to give them tools they can use well, and the training to know why that is different from using them 
quickly.

*[Dr Matthias Kretz](/about) and [Dr Ralph J. Steinhagen](https://github.com/RalphSteinhagen) are senior scientists at 
GSI Helmholtzzentrum für Schwerionenforschung / FAIR in Darmstadt, Germany. Matthias develops real-time software for 
experimental detector systems and has played a leading role in bringing SIMD abstractions into the ISO C++ standard, 
with open-source contributions ranging from the KDE community to efforts that make high-performance C++ accessible to 
working scientists and engineers. Ralph has developed real-time software for particle accelerators since his time at 
CERN and continues to do so at GSI/FAIR. An IEEE member for more than two decades, he is the primary architect of GNU 
Radio 4 and contributes to real-time signal-processing systems used in demanding scientific and critical infrastructure.*

*This post was initially published on 
[Medium](https://medium.com/@dr.steinhagen/your-ai-coding-assistant-is-not-a-junior-developer-56226251546d)*
