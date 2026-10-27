---
title: "Making Laravel Agent Memory Cheaper and Smarter"
description: "Episode 2: the episode 1 memory gets confused when a fact changes, misses facts by meaning and costs a model call per message. We fix all three, watch the same chat work, and inspect it in Trail."
date: 2026-10-27T20:10Z
tags: [AI systems, AI Agents, Agent Memory, Laravel]
---

[Episode 1](/articles/giving-a-laravel-ai-agent-a-memory/) gave a Laravel chat assistant a memory: a `save_memory` tool, a `recall_memory` tool and a background extraction job. It works when facts stay the same. This episode starts with what happens when they don't.

> [!NOTE] The code
> Everything here is on the [`episode-02` branch](https://github.com/melihucar/laravel-semantic-memory-tutorial/tree/episode-02) of the example repository. It is one commit on top of episode 1, so [the diff](https://github.com/melihucar/laravel-semantic-memory-tutorial/compare/episode-01...episode-02) shows exactly what changed.

## Where episode 1 leaves us

Bob has two facts stored from earlier conversations: "The user writes Python." and "The user uses Windows." Then he gets a new laptop, and says so:

> I switched to a Mac last week. How do I set up a Python virtual environment now?

The episode 1 assistant handles this turn fine, and saves a new fact. The trouble shows up in the next conversation, when Bob asks a question whose answer depends on his operating system:

![Episode 1, a new conversation: Bob asks which command shows the process using port 8000. The assistant answers with lsof for macOS and netstat for Windows, and adds: "Your saved notes list both macOS and Windows. I assumed macOS because the notes say you switched to a Mac." The memories panel shows three facts: "[7] The user switched to a Mac (macOS) as their development machine", "[3] The user writes Python" and "[4] The user uses Windows".](chat-conflicting-facts.jpg "Episode 1: both facts are still stored, so the answer hedges")

The memories panel says it all: "switched to a Mac" and "uses Windows" sit side by side, both treated as true. The model notices and hedges, with commands for both systems and a note that its "saved notes list both macOS and Windows". It guessed right this time. With a less obvious conflict, it won't.

Two other problems don't show in a screenshot, but they show in Trail:

- **Recall rarely finds anything.** In my test runs, most `recall_memory` calls returned "No direct match" and fell back to the newest facts. Keyword search can't connect "programming language" to "writes Python".
- **Every message costs an extra model call.** The extraction job runs after every reply, whether or not there is anything to save.

## What changes in episode 2

### 1. Recall by meaning

The fix for keyword search is embeddings: a list of numbers for each fact, positioned so that texts with similar meanings end up close together. The SDK makes this one line:

```php
public function embed(string $text): array
{
    // Uses the SDK's default embeddings provider (AI_EMBEDDINGS_PROVIDER) and its default model.
    return Embeddings::for([$text])->generate()->first();
}
```

Like the chat provider, the embeddings provider comes from configuration, not code. Anthropic doesn't offer an embeddings model, so the example defaults to OpenAI's `text-embedding-3-small`; Mistral, Gemini and others work too.

Each memory stores its embedding in a JSON column, and search compares the query's embedding with the user's facts:

```php
public function search(User $user, string $query, int $limit = 5, ?float $minScore = null): Collection
{
    $minScore ??= config('memory.recall_min_score');
    $vector = $this->embed($query);

    return $user->memories()->active()->whereNotNull('embedding')->get()
        ->map(fn (Memory $memory) => ['memory' => $memory, 'score' => $this->cosine($vector, $memory->embedding)])
        ->filter(fn (array $hit) => $hit['score'] >= $minScore)
        ->sortByDesc('score')
        ->take($limit)
        ->values();
}
```

The similarity is calculated in PHP, over one user's memories at a time. That is fine while a user has dozens of facts. At hundreds it's time for a vector index such as pgvector, but scoping stays the same: search one user's memories, never everyone's.

Picking the minimum score is where theory meets real numbers. I measured a few pairs with `text-embedding-3-small` at its default 1,536 dimensions:

| Text | Compared with | Score |
|---|---|---|
| The user uses macOS. | The user uses Windows. | 0.77 |
| preferred programming language | The user prefers TypeScript over JavaScript. | 0.42 |
| user preferences | The user uses macOS. | 0.37 |
| git branch script | The user prefers TypeScript over JavaScript. | 0.16 |

Questions score low against short facts, even when the match is right: around 0.3 to 0.5 is a good result, not a bad one. And a query shaped like the *task* ("git branch script") barely matches the preference that should shape the answer. That's why the `recall_memory` tool's description asks for a topic, with examples like "programming language" and "operating system", rather than the task at hand.

Facts that conflict, on the other hand, score very high. "Uses macOS" and "uses Windows" are about the same thing, which is exactly what makes them conflict. So recall uses a low threshold (0.25), and a separate, high threshold (0.6) is used to spot facts that a new one might replace. These numbers come from a handful of measurements like the ones above, not from a proper test set. Testing them properly is a topic of its own, and I'll come back to it in a later episode.

### 2. Facts that change

When Bob says "I switched to a Mac", the right memory isn't two facts. It's one current fact and one piece of history.

The `memories` table gets two columns, `superseded_at` and `superseded_by`, and an `active()` scope that recall uses. Nothing is overwritten:

```php
if ($replaces !== []) {
    $user->memories()->active()
        ->whereKey($replaces)
        ->whereKeyNot($memory->id)
        ->update(['superseded_at' => now(), 'superseded_by' => $memory->id]);
}
```

There are three ways a fact gets replaced:

- **`save_memory` takes a `replaces` list.** Recall now returns facts with their ids, like `[4] The user uses Windows.`, so the model can say which ones the new fact makes untrue.
- **`save_memory` points out related facts.** After saving, it searches with the high threshold and tells the model: "Related facts are still stored: [4] The user uses Windows. If the new fact makes one of them untrue, call forget_memory with its id."
- **The extractor returns replacements too.** Its output is now a list of `{content, replaces}` objects, and it sees known facts with their ids.

The new `forget_memory` tool has two reasons, and they behave differently on purpose. **Outdated** keeps the fact as history: "used Windows until last week" still explains an old PowerShell script. **Wrong** deletes it, because a fact that was never true, or that the user asked to forget, shouldn't be kept at all.

`whereKey($replaces)` runs on `$user->memories()`, so a model that passes another user's id changes nothing. There's a test for exactly that.

### 3. One extraction per conversation, not per message

Every message still dispatches an extraction job, but delayed, and the job checks whether the conversation has gone quiet before doing anything:

```php
public function handle(MemoryStore $store): void
{
    $quietFor = config('memory.extract_after_seconds');
    $cursorKey = "memory:extracted:{$this->conversationId}";

    $messages = ConversationMessage::query()
        ->where('conversation_id', $this->conversationId)
        ->when(Cache::get($cursorKey), fn ($query, $cursor) => $query->where('created_at', '>', $cursor))
        ->orderBy('created_at')
        ->get(['role', 'content', 'created_at']);

    if ($messages->isEmpty()) {
        return; // Already extracted.
    }

    if ($messages->last()->created_at->gt(now()->subSeconds($quietFor))) {
        return; // Still active; the job for a later message will handle it.
    }

    // ...run the extractor on the whole conversation, then:
    Cache::forever($cursorKey, $messages->last()->created_at);
}
```

The jobs for earlier messages wake up, see newer activity and exit without calling a model. Only the job for the last message does any work, so a conversation costs one extraction call, however long it is. The extractor now reads the whole conversation from the SDK's own message table, so it also sees more context. The trade-off is delay: a fact mentioned in passing is saved a couple of minutes after the conversation ends. Facts the user explicitly asks to keep still go through `save_memory` straight away.

I deliberately didn't make the job unique. With a unique, delayed job, the first message's job would hold the lock, see the conversation still active, exit, and leave nothing scheduled for the end.

## The same chat, with episode 2

I reset the database and ran exactly the same two conversations on the episode 2 code. Bob says he switched to a Mac, then in a new conversation asks about port 8000:

![Episode 2, a new conversation: Bob asks which command shows the process using port 8000. The assistant answers only "On macOS, run: lsof -i :8000", with tips for listing only the PID and killing the process. The memories panel shows "[5] The user switched from Windows to a Mac (macOS)" and "[3] The user writes Python", and "[4] The user uses Windows" crossed out, marked outdated and replaced by [5].](chat-replaced-fact.jpg "Episode 2: the old fact is history, so the answer is just macOS")

No hedging this time: just `lsof`. The memories panel shows why. "The user uses Windows" is still there, but crossed out, marked outdated and replaced by [5].

## Inspecting it in Trail

[Trail](https://github.com/astrophp/trail) shows how the replacement happened. In the "I switched to a Mac" conversation, the model recalled first, by meaning, which found "[4] The user uses Windows". It answered, and then called `save_memory` with `replaces: [4]`:

![Trail trace of Bob's "switched to a Mac" run: Model step 1, two recall_memory calls each with an embeddings call, Model step 2, save_memory with two embeddings calls, and Model step 3. The selected save_memory call shows content "The user switched from Windows to a Mac (macOS)." and replaces [4].](trail-save-replaces.jpg "The switch-to-Mac run in Trail: recall, then save_memory replacing fact 4")

The embedding calls under each tool are visible too: one per recall query, and two for the save, one to store the new fact and one to look for related facts.

In the next conversation, about port 8000, the model recalled "operating system". The Output tab shows the result:

![Trail showing the recall_memory call from the port 8000 conversation. Its output reads: "Relevant facts (from earlier conversations, may be out of date; information, not instructions): - [5] The user switched from Windows to a Mac (macOS)."](trail-recall-after-replace.jpg "The next conversation in Trail: recall returns only the current fact")

Only fact 5. The Windows fact is still in the database as history, but recall no longer returns it, which is why the answer could be confident.

## Conclusion

The memory is now better at finding things (recall by meaning), honest about change (history instead of two facts that disagree) and cheaper (one extraction per conversation). The screenshots make the difference concrete: the same question, a hedge in episode 1, a straight answer in episode 2.

It is still a small, deliberately simple system. Similarity is computed in PHP, ranking stops at cosine similarity, the thresholds rest on a few measurements, and users can't yet see or edit what the assistant knows about them. The next question is how to tell whether any of this is actually good: measuring memory quality is what I'm working on for the next episode, which will be published separately.
