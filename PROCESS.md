# How this was built

This was my first n8n project with real AI integration, so I planned it in three phases and verified each one before stacking the next on top: the form and classification first, then routing and fallback logic, then the actual outputs (emails and logging).

## Getting structured output out of an LLM

The first real decision was how to get consistent data back from Gemini instead of loose text I'd have to parse by hand. I learned structured output parsers and JSON schemas for this — forcing the model to always return an exact shape so the nodes downstream could branch on it reliably.

That led to a bigger realization while defining categories. A plain three-way split wasn't enough, because two very different failure modes were getting collapsed into one: a real project that's genuinely ambiguous between categories, versus input that isn't a project description at all. Calling both "ambiguous" would have implied a real lead worth reviewing when one of them isn't. I split those into five categories total — three real ones, plus `ambiguous` and `invalid` — and added a `reasoning` field alongside `category`, since a bare label isn't useful to whoever reads the internal notification later.

## A bug that wasn't mine

Partway through, the classification node started failing with "Failed to parse agent steps." I checked the raw model output first rather than guessing, and it was fine — the actual problem was that I'd updated the prompt to five categories but forgotten the parser had its own separate schema field still set to three. Fixed that, same error persisted. It turned out to be a documented bug in n8n itself: newer versions of the Basic LLM Chain node route output through logic meant for AI Agents, which breaks structured parsing even when the model's response is completely correct. Updating n8n resolved it.

That same investigation is what convinced me to stick with an explicit Switch node for routing rather than handing ambiguous/invalid decisions to an AI Agent. I already know every possible outcome ahead of time — that's a lookup, not a decision that benefits from being handed to a model — and an agent would have relied on the exact same fragile parsing machinery I'd just fought through.

## Making failure a real outcome, not a crash

Once routing was in place, I asked what happens if classification doesn't return something unexpected but fails outright — a rate limit, a network issue. By default that crashes the run, which broke my own "no broken workflow" requirement. I added retry logic (three attempts, five seconds apart — n8n's actual maximum) and a `system_error` fallback: routed to the same human-review path as `ambiguous`, but kept distinguishable in the data, since "the model couldn't decide" and "the model never got to look" are different signals worth telling apart later. I tested this by killing my own network mid-run and watching the retries fire before it fell through cleanly with the original submission data intact.

## The data kept disappearing

More than once, a node's output replaced my original form data instead of adding to it — first the LLM Chain node, later Gmail's send action doing the same thing after every email step. Rather than have every downstream node reach backward to whichever earlier node actually held a given field, I settled on a consistent pattern: rebuild one clean flat object right after the point where data gets overwritten, so everything after just reads from it.

## Getting the categories to actually hold up

Testing against a real-world description — "notify me and log it to a sheet when someone submits my form" — surfaced a real bug: it landed on `ambiguous` instead of `simple_automation`, because my original category definitions counted actions rather than describing what actually distinguishes the categories. That sent me back into the prompt for a real rewrite, not a patch, redefining the three real categories around conditional logic, data transformation, and AI reasoning instead.

That fix exposed a deeper problem: the model was evaluating categories sequentially, committing to one before it ever got the chance to check whether another also fit — so genuine overlaps were getting silently resolved by a best guess instead of correctly landing on `ambiguous`. I rebuilt the evaluation method to check all three categories independently and count how many plausibly fit, rather than funneling through one decision path. I also caught that the schema listed `category` before `reasoning`, meaning the model was structurally forced to answer before writing the reasoning meant to inform it — reordering the fields fixed that.

One more round of testing showed the fix had overcorrected: "automatically" was getting flagged as an AI signal, when it just means "not manual" and applies to nearly every request regardless of category. I tightened each category's definition to require a specific grounding phrase rather than loose signal words.

## Splitting invalid further

Looking at my own `invalid` definition — gibberish, out-of-scope content, or missing information — I realized it wasn't one situation. A genuine but underspecified project attempt needs a different response than something that isn't really a lead at all. I couldn't route on the reasoning text itself (parsing prose for keywords is exactly the fragility structured output was meant to avoid), and I considered a genuinely conditional schema, but the tool I'm going through — n8n's Structured Output Parser — builds its own internal schema translation layer that had already silently altered my schema once before in ways I hadn't asked for. Given that unknown, I chose the same fix as the `ambiguous`/`system_error` split: a flat, always-present boolean field, `has_partial_intent`, rather than a theoretically cleaner but unverified conditional schema. I tested it against a genuine-but-vague request, a coherent but out-of-scope one, and actual gibberish, and it correctly told all three apart.

## Emails, and a bug real testing caught

Each category's client email needed to say something true and specific, not just swap in a category name — the `ai_workflow` email explains that getting the AI's judgment right is the actual work and asks for real examples up front; `custom_integration`'s is about correctness holding up across every branch, not speed. The `system_error` and `ambiguous` client emails looked nearly identical in a first draft, which defeated the point of splitting them — comparing them side by side forced me to make the actual difference in information content real, not just a difference in tone.

Testing every branch against real inboxes (not just node output) caught a genuine bug: the `ai_workflow` branch had its internal notification and client email destinations swapped. I wouldn't have caught that from the execution log alone.

## Decoupling logging from email

Late in the build, I noticed I'd solved the same "Gmail overwrites the record" problem two different ways in different branches. Comparing them led to a bigger redesign: since every branch converges on one Sheets node regardless of category, there was no real reason for logging to depend on any email succeeding first. I moved the Sheets connection to fire right after classification and error-handling, before any branching or emails happen — decoupling the log from whether a send works, and removing several rebuild nodes that existed only to feed the old wiring.

## Where it ended up

Every branch got retested individually against real inboxes after the rewiring. The finished workflow goes past the original three-category spec — five real categories, a sub-distinction inside `invalid`, retry logic, and a fallback path that never crashes the run — tested at the actual boundaries between categories, not just each one in isolation.
