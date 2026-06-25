---
layout: post
title: "LongTermMemory Is Now on iOS: Spaced Repetition in Your Pocket"
---

*The best study session is the one you actually do. For most people that means five minutes on the train, ten minutes before bed, a quick review during lunch. The LongTermMemory iOS app is built around that reality.*

I've been working on [LongTermMemory](https://longtermemory.com) for a while now, and the web application has been available long enough that I've watched how people actually use it. The pattern is consistent: they upload materials and generate flashcards on a laptop, then they want to review on their phone. The [LongTermMemory iOS app](https://apps.apple.com/us/app/longtermemory-app/id6779253199) closes that loop.

## What the App Does

The core of LongTermMemory is AI-powered flashcard generation combined with spaced repetition scheduling. You upload a study document (PDF, PowerPoint, a photo of handwritten notes, or plain text), the AI reads it and produces question-answer pairs from the content, and the spaced repetition algorithm schedules each card at the optimal interval to move it into long-term memory before you'd naturally forget it.

The iOS app brings the review side of that workflow to your iPhone. Your account, your decks, and your progress sync from the web platform, so the workflow looks like this in practice: upload and review generated cards on a computer, then do daily review sessions on your phone whenever you have a spare few minutes.

The app runs on iOS 15.1 or later and also works on Apple Silicon Macs through the Mac App Store. It's a 34 MB install.

## Why Mobile Matters for Spaced Repetition

Spaced repetition only works if you show up consistently. The algorithm calculates exactly when each card should surface based on your personal forgetting curve, but if you miss the session the card is due, the whole system loses its precision.

Consistency is much easier when your review queue is on the device you already have with you. A five-minute session on the phone while waiting for coffee does more for long-term retention than an hour of passive re-reading at a desk. That's not intuition; it's what decades of memory research (starting with Ebbinghaus in the 1880s) established about the spacing effect.

The review interface in the app is optimized for short sessions. You see the question, produce your answer mentally, flip the card, rate your recall, and move to the next one. The interaction is low-friction by design, because the goal is to remove every excuse not to review.

## Who Benefits Most

**Students with dense course materials.** Uploading a chapter PDF and getting a study deck in minutes is a different experience from spending two hours in Anki building cards before you can start learning. The AI does the card creation; you do the learning.

**Professionals studying for certifications.** AWS, USMLE, CFA, bar exam, NCLEX — these all involve official study guides and reference documents that map well to flashcard generation. Upload the material, get the deck, let the algorithm manage your review calendar.

**Anyone who has tried Anki and stopped using it because making cards was too slow.** The research on spaced repetition is unambiguous: it works dramatically better than re-reading. The adoption gap has always been the upfront effort. Automating card creation removes that gap.

## A Few Honest Notes

The AI card generation performs best on text-dense content. For material that relies heavily on diagrams or flowcharts, the AI works from surrounding text, which means visual concepts may need a few manually added cards to cover properly. For most academic and professional study materials, though, the hit rate is high enough that editing a few cards is far less work than building a deck from scratch.

The app is currently at version 1.0.1. The core review experience is solid; feature depth will grow over time as the platform develops. The web application at [longtermemory.com](https://longtermemory.com) remains the more complete environment for uploading and managing content.

The app is free to download and use.

## How to Start

1. Download the [LongTermMemory app from the App Store](https://apps.apple.com/us/app/longtermemory-app/id6779253199)
2. Sign in or create a free account (the same account works on web and mobile)
3. Upload a piece of study material you're actively working with on the web platform
4. Open the app on your phone and start your first review session
5. Come back the next day — the algorithm will show you exactly what's due

The spaced repetition system takes care of the scheduling from there. Your job is to show up for the sessions it queues. With the app in your pocket, that's a much easier commitment to keep.
