# beta-readers

A Claude Code skill that simulates three distinct beta readers giving independent feedback on a
fictional text — a scene, chapter, short story, or any prose excerpt.

## Why

Single-reader feedback has a single perspective's blind spots. A structural reader misses emotional
falsehood; an emotional reader lets plot holes slide; a prose reader ignores whether the story goes
anywhere. Three readers with different obsessions surface different problems — and where all three
agree, you have something that genuinely needs fixing.

## The Readers

| Reader | Persona | Reads for |
|--------|---------|-----------|
| **Marcus** | Former thriller writer, blunt | Plot, structure, pacing, stakes, scene function |
| **Sofia** | Psychologist-turned-reader, precise | Character motivation, emotional truth, reader connection |
| **Dev** | Poet-turned-editor, exacting | Prose rhythm, word choice, voice, dialogue, texture |

Each reader has a distinct voice. They don't always agree. That's intentional.

## Usage

```
/beta-readers
```

Then paste your text, or provide a file path:

```
/beta-readers [paste text here]
```

```
/beta-readers path/to/chapter.txt
```

Or just talk:

```
"Can you beta read this scene for me?"
"Give me feedback on this opening chapter."
"What's wrong with this passage?"
```

## What You Get

Three independent feedback blocks — one per reader — followed by a synthesis that identifies:

- Issues where ≥2 readers agree (fix these first)
- Issues where readers diverge (matters of craft judgment you decide)
- A priority order for the next revision pass

## Files

| File | Purpose |
|------|---------|
| `SKILL.md` | Reader personas, behavioral overrides, execution protocol |
| `README.md` | This file |
