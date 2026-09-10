# Process notes

This was my first n8n project with real AI integration. The idea is simple on the surface: a client fills out a form, an LLM reads their project description, and the system routes it into a category with a tailored confirmation email, an internal notification, and a logged row in Google Sheets. Getting there properly took more thought than I expected.

I planned it in three phases: the form and AI classification first, then the routing logic with a fallback for uncertain cases, then the actual outputs, emails and sheet logging, each varying by category. Working this way let me verify each layer actually worked before stacking the next one on top.

## Phase 1: Classification

I set up the Form Trigger first and pinned a real test submission, a project description that's genuinely mixed, part simple notification, part AI classification work. That turned out to be a useful edge case to keep in mind throughout.

The first real design decision was how to get consistent, structured output from an LLM instead of loose text I'd have to parse. I learned about structured output parsers and JSON schemas, a way of forcing the model to always return data in an exact shape so downstream nodes could reliably branch on it. I built a Basic LLM Chain wired to a Gemini Chat Model and a Structured Output Parser.

While defining the three original categories, I realized a plain three-way split wasn't enough. What happens if someone submits something that doesn't fit at all, or fits more than one bucket? I needed to separate two different failure cases: a real project that's genuinely ambiguous between categories, versus input that isn't a project description at all. Treating those the same would have been misleading, since an "ambiguous" label implies a real lead worth reviewing and junk input isn't that. I landed on five categories total: three real ones, plus ambiguous and invalid.

I had to properly learn JSON schemas from scratch, since properties, type, enum, and required aren't nested the way I first assumed. Once it clicked, I added a reasoning field alongside category, since a bare label with no explanation wouldn't be useful for whoever reads the internal notification later.

Writing the classification prompt itself controls the model's judgment, not just the shape of its answer. I built it in four pieces: a role sentence establishing the assistant's one job, definitions for all five categories, an explicit tie-breaking instruction telling the model to choose ambiguous or invalid instead of guessing when torn, and finally assembling all four pieces with the input variable. The category definitions went through a real revision here: my first draft of invalid ("missing information") overlapped with what ambiguous should cover. The actual test turned out to be whether a human could still make a call with what's there (ambiguous) versus whether there's nothing usable to route at all (invalid).

### Bugs hit in Phase 1

The classification node kept failing with "Failed to parse agent steps." I checked the raw model output first and found Gemini's answer was fine, it was my Structured Output Parser that still had the old three-category schema. I'd updated the prompt to five categories and forgot the parser had its own separate schema field. Fixed that, and it still failed with the same error. That sent me digging deeper, and it turned out not to be my mistake at all: it's a documented bug in n8n itself, where newer versions of the Basic LLM Chain node route output through logic meant for AI Agents, which breaks structured parsing even when the model's response is completely correct. Updating n8n resolved it cleanly.

Separately, the LLM Chain node's output was replacing my original form data entirely, so name, email, and description were gone by the time classification finished. I merged everything into one clean flat object right after classification, rather than having every later node reach backward to different earlier nodes for different fields, so there's one clear point downstream where the record is complete.

Before moving on, I renamed every Phase 1 node, role first and specific to this workflow, so nothing on the canvas needed opening just to know what it does.

Before starting Phase 2, I also questioned whether a Switch node with a fallback was even the right tool, versus letting an AI Agent handle the ambiguous and invalid routing and email decisions itself. I reasoned through it and stuck with explicit branching: I already know the full set of possible outcomes ahead of time, I just don't know which one applies to a given input, and that's a lookup, not a decision that benefits from being handed to a model. An agent would also add a second layer of unpredictability on top of classification, and it relies on the same fragile "agent steps" parsing machinery I'd just fought through a real bug in.

## Phase 2: Routing

I questioned whether the Switch node's fallback output was even needed, reasoning that the Structured Output Parser would reject anything outside my five valid categories before it reached a Switch. That held up, but it led to the real question I'd missed: what happens if classification fails entirely, not just returns something unexpected? A hard failure (network issue, rate limit, a broken response) would crash the whole execution by default, which directly violates my own "not a broken workflow" requirement.

I decided a failed classification should route to the same human-review path as ambiguous, but stay distinguishable in the data as its own case, system_error, since "the model couldn't decide" and "the model never got to look" are genuinely different signals when reviewing this later. Before falling back, I added retry logic: 3 attempts, 5 seconds apart (n8n's actual maximum, lower than I first wanted). I tested it by killing my own network mid-run and watched the retries fire in the execution log before it gracefully fell through to the error-handling path with the client's original data intact.

I built the Switch itself: six rules, one per category, with ambiguous and system_error kept as separate rules but wired to converge on the same downstream node, matching stays granular while routing consolidates where it makes sense. I renamed the Switch node and all six rules using the same convention as Phase 1; the rules for ambiguous, system_error, and invalid also got a short suffix naming the consequence, not just the category, so I'd know at a glance what each one meant without opening it.

## Phase 3: Outputs

I initially planned to structure each branch the way I'd structured the first two phases, Sheets first then emails, one branch at a time. Thinking it through, I caught a real conflict: Sheets logging is identical logic across all five outcomes, just different data. Putting it first inside each branch meant either duplicating the same Sheets node five times, or converging everything after Sheets and then branching all over again just to send category-specific emails. Neither made sense, so I flipped the order: build each branch's confirmation email and internal notification first, since that's the part that's actually different per category, then let all five paths converge into one shared Sheets node at the end.

### Ambiguous / system_error branch

I built this branch first since it forces both categories through shared logic. The internal notification could stay shared, since my team needs the same information either way, they just need to clearly tell "needs manual categorizing" apart from "needs a systems check." The client email genuinely needed to split: a system_error submission hasn't actually been classified at all, so a generic "we've received your message" doesn't fit it, it needs to ask for more information instead, the way I'd treat an invalid submission. I split the client-email step into its own branch using an IF node checking whether the category equals system_error, rather than folding both messages into one conditional template.

My first drafts of the two client emails ended up almost identical, which defeated the point of splitting them. When that got flagged, I pushed back, since I realized the hedge in the system_error draft ("we may need a bit more detail") wasn't weak writing, it was accurate: a system_error submission's real outcome is still unknown at that point, it could turn out valid, ambiguous, or genuinely invalid once someone manually looks at it, and only the invalid case needs more information from the client. Comparing both drafts side by side afterward confirmed they now differ in real information content, not just tone.

I also hit a data-flow issue before wiring the IF node: sending through Gmail replaces the item's data with Gmail's own send-confirmation output, the same problem as the LLM Chain node overwriting data back in Phase 1. I added a Set node right after the internal notification to rebuild one clean object once, so everything downstream just reads from it directly.

Closed out the branch by renaming the IF node and all three email nodes, following the same role-first convention as Phase 1 and Phase 2's Switch.

### simple_automation branch

Simpler on paper, since it's a single category with no dual-audience split to untangle. I wrote the internal notification first: client info, category, and the original description, short and functional since there's only one reader and no ambiguity to explain. The client-facing confirmation email had a real constraint from the spec: "reference their category naturally," with no internal labels leaking into client language. I translated my own internal definition of the category into plain terms a client would want to read, and added honest turnaround-time framing, since fast delivery is something I can genuinely promise for this category specifically.

The same Gmail-overwrite issue showed up again, so I added another Set node to rebuild the flat object before it reaches the shared Sheets node later.

Testing this branch against a real description, "when a client submits the form on my website, notify me and log it to a sheet," surfaced an actual bug: it got classified as ambiguous instead of simple_automation, because my original category definition only allowed exactly one action off a trigger. That sent me back into the prompt for a real rewrite, not a patch.

I redefined the three real categories around what actually distinguishes them, conditional logic, real data transformation, and AI reasoning, instead of counting actions. That surfaced a deeper structural problem: my evaluation was sequential, forcing the model to commit to a single category before it ever got the chance to check whether another one also fit, so genuine overlaps were getting silently resolved by the model's best guess instead of correctly reaching ambiguous. I confirmed this with a real test case, "route incoming leads smartly based on their details," and watched the reasoning field admit it saw two valid readings and picked one anyway. I rebuilt the evaluation method around independently checking all three categories and counting how many plausibly fit, rather than funneling through one decision path.

That fix exposed a second issue: the schema listed category before reasoning, meaning the model was structurally forced to output its answer before ever writing the reasoning meant to inform it. Reordering the schema fields so reasoning comes first fixed that.

Testing again, the new AI-signal wording overcorrected: "automatically" got flagged as a possible AI signal, when it really just means "not manual" and applies to nearly every automation request regardless of category. I tightened the definitions to require a specific grounding phrase or word for each category, and explicitly excluded words like "automatically" from counting as AI signals.

Finally, I cleaned up a leftover tie-breaking section that duplicated the new evaluation method's logic, just less precisely, and reran both the original bug case and a genuine ambiguous case to confirm the prompt handled both correctly.

Before moving to the next branch, I did a full renaming pass, nine nodes across the two completed branches. Since I now had multiple "rebuild the record after Gmail overwrote it" nodes across different branches, I added branch-specific suffixes so they're distinguishable on the canvas instead of all sharing one generic name.

### ai_workflow branch

The internal notification followed the same mechanical shape as every branch before it. The client email needed more thought: what's actually different about this category isn't just that it takes longer, it's that getting the AI's judgment right is the work, which is a real reason a client deserves to hear rather than just a timeline to accept. I built the email around a scoping step that explicitly asks for real examples, since you can't judge whether an AI's output is good without seeing what "good" looks like for that client's case. I closed with an honest reframe: the extra time buys handling real-world cases, not just the happy path.

For testing, I wrote a description that stacked multiple category signals into one input on purpose: real conditional branching, multi-system orchestration (Gmail, a Data Table, Slack), and clear AI reasoning (sentiment extraction, summarization, drafting a reply) all at once. That checks whether the prompt correctly prioritizes AI reasoning over custom_integration when a description legitimately qualifies for both. The reasoning field showed the model explicitly weighing the branching and orchestration signals against the AI signals and correctly applying my own exclusion rule, that custom_integration only applies when logic is driven by fixed rules, to land on ai_workflow.

### custom_integration branch

I wrote the client email around a different honest pitch than the other two: not speed, not AI judgment, but correctness, making sure every path through the branching logic holds up, not just the common case. I tested it against a real branching scenario (budget-based lead routing across HubSpot, Slack, and Mailchimp) with deliberately no AI language in the description, to check the opposite failure mode from the ai_workflow test, that genuine complexity alone doesn't get miscategorized as ai_workflow just because it's multi-system. The reasoning correctly ruled out both other categories using my own exclusion language.

### invalid branch

Before writing anything here, I thought through whether invalid needed the same kind of split I'd already done for ambiguous and system_error. My own invalid definition, gibberish, out-of-scope content, or missing critical information, wasn't really one situation, it was at least two: a genuine but underspecified project attempt, versus something that isn't really a lead at all. Those need different client emails.

The classifier only output a single invalid category with no way to tell which sub-case triggered it. Routing on the reasoning text itself would mean parsing prose for keywords, which is fragile for the same reason structured output existed in the first place. I considered a genuinely conditional schema and found that Gemini's API now supports conditional schema keywords, but n8n's Structured Output Parser builds its own internal schema translation layer, the same layer that had already silently added an unexplained "output" wrapper to my schema earlier without either of us writing it. Given that unknown, I chose the same fix as the ambiguous/system_error branch: a flat, always-present boolean field, has_partial_intent, proven and consistent over a theoretically cleaner but unverified approach in this specific tool.

I tested it against three real cases: a genuine but vague project request ("help automating my workflow") correctly returned true; an out-of-scope but perfectly coherent request ("fix my car") correctly returned false, which turned out to be a sharper test than plain gibberish since it proved the model wasn't just treating "makes grammatical sense" as partial intent, it correctly reasoned about domain fit; and actual gibberish also correctly returned false, with the reasoning explicitly contrasting itself against the vague-but-real example.

## Wiring cleanup and a bigger redesign

While wiring the final Sheets node, I noticed the ambiguous/system_error branch was feeding Sheets from two different points off the same rebuild node, one path through the client emails, one straight to Sheets. It wasn't a bug, since the email path never reconnected to Sheets, but it revealed I'd solved the same "Gmail overwrites the item" problem two different ways across the workflow.

That comparison led to a bigger redesign. Since every branch converges on one Sheets node regardless of category, there was no real reason for Sheets to depend on any client email succeeding at all. I moved the Sheets connection back to the two points right after classification and error-handling, before any branching or emails happen. That decoupled logging completely from whether any Gmail send works, which matters given a real send could fail for its own reasons unrelated to classification. It also meant has_partial_intent only needed adding in two places instead of four, and let me remove every end-of-branch rebuild node that existed purely to feed the old Sheets wiring, keeping only the ones that genuinely fed a client email's own data. Before touching anything, I confirmed which nodes were safe to remove versus which still fed live email content, since deleting the wrong one would have silently broken a client-facing email.

## Final testing

I retested every branch individually after the rewiring, checking real inboxes rather than trusting node output alone. That caught a real bug in the ai_workflow branch, the internal notification and client email had their destinations swapped, sending the internal-only message to the client and vice versa. I confirmed the same mistake hadn't slipped into the already-tested simple_automation branch, since I'd used the same inbox-checking method there too.

Testing ambiguous specifically exposed a flaw in my own first test case: a deliberately vague description landed on invalid instead, and walking through the reasoning showed the classifier was actually right, vague language alone doesn't ground a real signal for any category, which is a different failure than two categories both being concretely, textually supported at once. I built a sharper test case with explicit signals for both ai_workflow and custom_integration in the same description, which correctly resolved to ambiguous.

For system_error, I needed a way to fail classification specifically without losing my whole network connection, since I needed live email delivery to verify the output. I temporarily broke the Gemini node's credentials instead, which forced the same retry-then-fallback path without touching anything else, and reverted the key afterward.

I closed out by testing the invalid/false path deliberately, confirming a genuinely out-of-scope input produces no emails at all and still logs to Sheets, which was the one silent case worth proving on purpose. I named the final Sheets node, the only node in the whole workflow that had gone unnamed through all the redesign.

Walking the finished build back against my original project spec, every stated requirement was met, and the classification model itself ended up going well past the original three-category ask: five real categories, a sub-distinction inside invalid, retry logic, and graceful fallback that never crashes the workflow, all individually tested, including the actual boundaries between categories, not just the categories in isolation.
