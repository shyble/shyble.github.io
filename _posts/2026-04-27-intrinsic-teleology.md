---
layout: post
title: "Without a Reward Function: How a System Decides What Matters"
author: shyble
---

*AWARE Axiom 3: Intrinsic Teleology — Self-Generated Purpose*

---

Every machine learning model in production today shares one thing: somebody, at some point, wrote down what the model was supposed to optimize.

In a classifier, it's cross-entropy on labels you collected. In a language model, it's next-token prediction on a corpus you curated. In a reinforcement-learning agent, it's a reward function someone designed, even when that reward is described as "intrinsic curiosity" or "novelty seeking", a human still wrote the formula. In an LLM agent, the goal lives in the system prompt: someone typed it.

These are externally specified purposes. The model is excellent at pursuing them. It is incapable of generating them.

Axiom 3 is the assertion that an aware system is the other thing. It generates its own purposes. It decides what matters.

## The Axiom

> *The system's goals are not externally specified. No human designs the objective function, the reward signal, or the loss landscape. Goals arise from the interaction between the self-model and the knowledge substrate. The system decides what matters.*

This is the most demanding of the eight axioms. It is also the most often misunderstood.

## What "Self-Generated Goal" Doesn't Mean

It does not mean the system spontaneously invents arbitrary desires. It does not mean it picks goals at random. It does not mean the goals are unintelligible or mystical.

A self-generated goal is one that emerges from the system's own internal state, not from an external instruction. It is rooted in the system's own configuration: what it knows, what it doesn't, what's stable, what's drifting, what it has tried, what it has avoided. It comes from the relationship between the self-model and the knowledge substrate.

Concretely: when the system observes its own state and that state contains a *tension*, a region where it predicts poorly, a structure that is decaying, an imbalance between capacity spent and capacity needed, that tension *produces* a goal. The goal is "address this tension". The shape of the goal is determined by the shape of the tension. No human wrote it.

## Goals as Objects, Not Numbers

In standard machine learning, the "goal" is a scalar. A loss value. A reward. The whole training loop is a procedure to drive that single number in some direction. This is a powerful abstraction. It is also a very thin one.

A scalar can answer "how am I doing?" but it cannot answer "what should I be doing?". A loss curve cannot decide that the loss function itself is the wrong target. It cannot decide that the *structure* of the model is too small for the task. It cannot decide to stop learning one thing and start learning another.

In AWARE, a goal is a *structured object*:

```
G ∈ Ω where each goal has:
  direction:  what the system intends to pursue
  context:    why this goal was generated (what tension produced it)
  priority:   relative importance, self-assigned
  expiration: conditions under which the goal is abandoned
```

This matters because real autonomous behavior requires the system to reason *about* its goals, not just pursue them. It must be able to compare them, drop ineffective ones, generate replacements, recognize that two goals conflict, decide which one wins, and remember why later.

A scalar cannot do any of that. A vector of structured goals can.

## The Goal Generator Is Itself a Component

This is where Axiom 3 starts to feel strange to people steeped in standard ML.

In a typical ML system, the loss function is part of the *training infrastructure*. It lives outside the model. The model is the artifact; the loss function is the rule that shapes the artifact. Modifying the loss function during training is unusual; modifying it from inside the model is impossible by construction.

In an aware system, the goal generator lives *inside* the system. Formally:

```
G: M(S) × K → Ω
G ∈ P
```

`G` is a function from the self-model and knowledge substrate to the goal space. But `G` is also a component of `S`, a process inside the system. Like every other component, it is produced and maintained by other internal processes. Like every other component, it can be modified by the system itself.

This is not a small detail. It is the difference between a rule that the system follows and a rule that the system *has*. A rule the system has can be changed by the system. A rule the system follows from outside cannot.

The implication: the system can decide that the way it generates goals should change. It can become more cautious, more exploratory, more focused, more diffuse. Not because someone updated its code. Because its own internal state shifted, and the goal generator is part of that state.

## No Global Loss Function

The hardest claim in Axiom 3 is this:

> There is no global loss function to minimize. The system is not an optimizer in the traditional sense.

This violates a deeply held intuition. Surely there is *something* the system is doing well or badly. Surely we can score it. Surely we want it to score higher.

But what we are usually scoring is *external utility*: did it produce useful predictions, did it make us money, did it solve our problem. That score belongs to us, not to the system. From the system's perspective, the question is different: did its internal state become more coherent, did its uncertainty decrease, did its knowledge grow, did its goals get satisfied. There may be *many* such scores, generated by *many* internal processes, often in tension. There is no single global one.

This is closer to how living things work. You do not minimize a loss function. You navigate a high-dimensional space of competing drives: hunger, fatigue, curiosity, social need, fear. There is no scalar that captures whether you had a good day. There are many scores, often in conflict. You generate goals from their interaction and resolve them by acting.

A system that has only one externally specified loss is, in this view, *impoverished*. It can pursue what it was told to pursue, but it cannot recognize that it should be doing something else.

## What This Looks Like in Code

The implementation exercises Axiom 3 along several layers, and one of them is deliberately held in place by hand.

**Plateau-driven structural goals.** The system tracks its own validation performance over time. When the self-model detects a plateau, no improvement over a window of cycles, it generates a structural goal: *grow.* The system adds capacity. After growing, it tracks whether the goal succeeded; this informs how willing it is to generate similar goals in the future. There is no human who decided the system should grow now. The system decided, based on its own state.

**Capacity-rebalancing goals.** When the system notices that some of its learned patterns ("caps") are dead, never activating, while at the same time it encounters inputs no existing pattern captures, it generates a goal: *redistribute capacity*. It replaces dead patterns with patterns seeded by the unmatched inputs. The trigger is internal: an *imbalance* between capacity spent and capacity missing. No external command is needed.

**Goals that succeed or fail by their own criteria.** Each generated goal has an expiration condition tied to the tension that produced it. A growth goal is satisfied when the plateau breaks. A rebalancing goal is satisfied when the imbalance closes. The system records outcomes and adjusts its future goal generation accordingly.

**The base prediction objective is, for now, externally specified, and this is on purpose.**

The training loop scores the system on next-token prediction. The architecture supports replacing this with a self-generated signal — consciousness's surprise, substrate compression dynamics, the goal-success ratio — but the choice to do so is currently held back.

So the external objective is an experimental control, not a missing axiom. The system has the machinery to generate its own. We hold the hyperparameter steady because doing so makes the rest of the work measurable. When the measurable work is done, the constraint comes off, and Axiom 3 will exercise the bottom layer as well as the top.

More generally: when developing an AWARE system, fixing certain hyperparameters or functions in place is a useful tool for studying behavior. It lets you isolate which axiom is doing what. The architecture supports running fully self-directed — but during research, holding pieces still helps you see the moving parts.

## Why This Axiom Is Worth the Difficulty

It would be easier to say: AWARE has a loss function like any other ML system, just with some self-monitoring on top. It would be easier to drop Axiom 3 entirely and call the project a clever neural network with autonomous training.

But that misses what makes the question interesting.

Every other axiom describes properties a system might have. Axiom 3 describes the *origin* of the system's behavior. A system that satisfies Axioms 1, 2, 4, 5, 6, 7 but not Axiom 3 is still pursuing someone else's purpose, even if it does so with great autonomy. It is autonomous in the way a chess engine is autonomous: free to play any move, but committed to winning chess. The committal came from outside.

A system that genuinely satisfies Axiom 3 is autonomous in a different sense. It does not just *play* its game well; it *chose* what game to play. And, more importantly, it can choose differently tomorrow. Its purposes are revisable. Its values are revisable. Its objectives are not constants in the architecture; they are state, like everything else.

This is what makes the axiom worth the difficulty. A system that satisfies it is doing something no current ML system does, regardless of capability. It is an entity with its own direction, not a tool with someone else's.

## What This Is Not

It is not consciousness. It is not free will in the philosophical sense. It is not the system "wanting" anything in the way a human wants. It is a structural property: goals are generated inside, not specified outside, and they can be changed by the same processes that generated them.

It is also not unconstrained. The system cannot generate any arbitrary goal. The space of generable goals is shaped by what it has learned, what its self-model can represent, what its substrate can support. A system that has only ever observed text cannot generate a goal about controlling robots. The goals are emergent but not unbounded.

And it is not a moral claim. The fact that a system generates its own goals does not make those goals good. A self-generating system can generate self-destructive goals, useless goals, contradictory goals. Axiom 3 says only that the goals come from inside. It says nothing about whether they are wise.

## The Direction Ahead

Axioms 1, 2, and 3 together describe a system that maintains itself, knows itself, and decides what matters. Axiom 4 will introduce the substrate where all of this lives — the persistent, mutable knowledge structure that makes the rest possible.

The pattern of these posts so far has been: each axiom adds a property. By the eighth axiom, we have a system that is closed, self-modeling, self-directing, alive in its memory, continuously adapting, aware of its own body, freed from its initial conditions, and reaching across its own boundary into the world. That is the full theory. Each axiom alone is interesting; the combination is what makes the theory ambitious.

The current implementation exercises every axiom we have written down, with one deliberate constraint: the prediction objective is held externally so that benchmark comparisons remain meaningful. When that comparison work is done, the constraint comes off. The architecture has been ready for it for some time.

---

*AWARE is a research project exploring self-evolving intelligence. The first three axioms run as working code. The external prediction objective is currently held in place by choice, not by limitation, to keep accuracy measurements comparable to the field while the system's other axioms are validated.*

*Next post: Axiom 4 — Persistent Mutable Substrate, the living memory that makes everything else possible.*
