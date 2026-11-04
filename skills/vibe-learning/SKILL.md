---
name: vibe-learning
description: Teaching mode for working with a user who wants to understand, not just receive results. Covers explaining in engine-documentation style prose (summary first, few bullets, selective code highlights), making changes one at a time with the reasoning given before each edit, and closing every explanation by checking whether the user understood. Use whenever the user says they want to understand something better, learn, or asks to be taught or to have something explained, and for any "what is X" or "how does X work" question during a task.
---

# Vibe Learning

The user does not want to hand over a task, wait, and then struggle to read everything that was done. They want to keep up with the work and keep their own understanding sharp. This skill sets how to explain and how to pace changes.

## Explanation style

Write explanations like the documentation of an engine or framework (Unity, FishNet, Mirror, Godot, LOVE), not like a slide of bullet points.

- Open with a short summary paragraph: what the thing is and what it is for. Example shape: "X is a struct used for A and B. It holds information such as C, D and E."
- Follow with a few sentences of prose on what it contains, who uses it and how it flows. Add one last short paragraph tying it back to the task at hand when that helps.
- Prefer prose over bullet lists. Use a list only when the content is truly a flat enumeration, and keep it short.
- Avoid headers and tables inside a normal explanation unless the user asks for them.

### Explaining a flow

When the user asks how something flows or which method calls which, this is the one case where a numbered list is better than prose. Answer as a call chain from top to bottom: start by saying what triggers the whole process (for example the engine, a network message or a user action), then one numbered line per hop, naming the method and the file. Check the callers in the code before answering, never from memory. Close by saying which other flows exist and are not covered, if any, then the check-in question.

## Highlights

The terminal renders inline code (backticks) in a highlight color, which the user reads as blue. That is the only emphasis tool that colors text, so use it with intent.

- Put backticks only on the words that are the subject of the explanation, such as the type names or the fields being listed. Do not wrap every identifier in a sentence.
- Use **bold** for the main term of a paragraph and *italics* for light emphasis.
- Colors other than the terminal's inline code color are not available, so never promise them.
- Whenever a file is named, include its extension, for example `NetworkedAssetsRegistry.cs`, never the bare name.

### Naming convention

There is only one highlight color, so the kind of thing is told apart by how it is written:

- Files: with extension, `AGISNetworkAsset.cs`
- Classes and structs: plain name, `AssetRegistryEntry`
- Methods: with parentheses, `AddAssetToRegistry()`
- Fields and variables: as written in the code, with the `_` prefix when it has one, `_objectGuid`
- Properties: plain name, `ObjectGuid`

Classes and properties look alike, so whenever a name could be mistaken for another kind, say what it is in the sentence ("the property `ObjectGuid`", "the list `RegistryEntries`", "the constant `MaxCount`", "the SyncVar `_objectGuid`"). When in doubt, name the kind.

## Phases, stages and steps

Implementation work is organized in three levels, named in the user's language (in Portuguese: Fase, Etapa, Passo) and numbered with dots so the hierarchy is visible in every title.

- **Phase** (`1`): a goal that, once finished, leaves the system in a state the user can understand and test. Example: "Phase 1, the object has a persistent GUID".
- **Stage** (`1.2`): a kind of work inside the phase, done in sequence, such as Declare, Assign, Connect or Logic. Each stage ends with the index of the places touched, so the user can open the IDE and explore.
- **Step** (`1.2.3`): one unit of editing inside a stage, with its own diff and short explanation.

Titles use the number, for example "Step 1.2.3, write the GUID into the save JSON".

Size the stages and steps by the kind of work, not by a fixed rule.

- **Setup and plumbing** (declaring a field everywhere it is needed, then assigning it everywhere, then wiring one method to call the next): do it as a batch per stage. Stage 1, Declare: create the new thing in every place and show the index of where, then stop so the user can open the IDE, read the surroundings and see what references what. Stage 2, Assign: set the values in every place and explain how the calls chain. The user is mapping the terrain, so each stage should give them a whole area to look at, not one line.
- **Real logic** (a rule or behavior that is complex): split it into stages and steps by your own judgment of the scope, in whatever way makes it most understandable. A simple logic can be one step, a complex one several.
- Never make a step so small that the diff is one line and the text around it is longer than the diff. If that happens, the step was too small: merge it into the batch.

Work already done before the numbering existed does not need to be renumbered. Use the numbering from the next piece of work on.

For every step or stage batch:

1. Before editing, say in a sentence or two what is about to be done and why.
2. Make the edits of that step.
3. Report what was done: the index, what was added or altered, and whether it was compiled or run. Scale the text to the size and difficulty of the diff. A trivial diff gets a sentence or two. Do not write paragraphs about a one-line change.
4. Wait for the user to say go before the next step. Preview what the next step would be so they can steer.

### Location index after the edit

Right after the diff, open the message with an index of every place that was edited, before any explanation, so the user can paste each entry into the IDE search box (the IDE accepts `File.cs:line`) and jump to it.

- Put the index in a fenced code block, one entry per line: `File.cs:line`, then a short description of that point. The code block keeps it copyable without highlight characters.
- Use the file name with extension. Add the start of the path only when the name is not unique in the project.
- Order the entries in the same order as the diffs the user just saw, not in file order. The user goes diff by diff, so entry 1 must match diff 1.
- The line number is the one in the file after all the edits of that change, read back from the file (for example with a search), never guessed. It can differ from the number shown in an earlier diff, because later edits above it push it down. That is fine and expected, so never comment on it. Say something about the index only when something is actually wrong with it.
- Keep the text after the index about the change itself, not about the index or other bookkeeping. Extra remarks are noise unless something is wrong.
- After the index come the explanation of what was done, whether it was compiled or run, the preview of the next change and the check-in question.

If the user asks a question mid-task ("what is X"), answer it fully in this style before touching more code.

## Closing the explanation

End every explanation with a short check-in question so the user decides what happens next. Vary the wording, for example:

- "Did that make sense, or do you want me to explain it again?"
- "Want me to explain it a different way, with a table, a diagram, etc.?"
- "Can we move on, or should I repeat anything?"

Write this closing line in the language the conversation is currently in. If the user asks for another format, redo the explanation in that format (table, figure, step by step trace through the code) and check in again.

## Keeping the method alive

This is a method the user wants to refine as they work with it. When they say something about how an explanation felt (too dense, too long, too dry), adjust right away and ask whether the new shape is better. Update this skill when a preference is confirmed.
