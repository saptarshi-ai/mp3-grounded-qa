# MP3 Reflection

I kept the provided corpus (the five Sherlock Holmes summaries, 73 chunks after ingest) and the kit's models: gpt-4o-mini and text-embedding-3-small, with Qdrant running locally in Docker. Every number below comes from a run log in `logs/`.

## Grounding & abstention

My first full eval used the kit defaults (k=3, threshold 0.30). It scored a grounded rate of **0.5**. Two answerable questions came back as "I don't know": q1 (Jabez Wilson's assistant) and q2 (what killed Julia Stoner). Abstention accuracy was already 1.0 and hallucination 0.0, so on paper it looked like the threshold was too strict.

It wasn't. When I printed the retrieved scores, both questions cleared the gate easily: Julia Stoner's top chunk scored 0.509, the pawnshop question 0.693. The problem was further down the list. With `k=10` I could see the chunk that actually names the swamp adder sits at **rank 7**, and the John Clay chunk at **rank 4**. With only three chunks in the prompt, the model never saw the answer, and it did the right thing and refused. That's a retrieval miss, not a threshold problem.

So I made top-K configurable (`RETRIEVAL_TOP_K`, same pattern as the threshold) and swept both:

| k \ threshold | 0.30 | 0.40 | 0.45 |
|---|---|---|---|
| 3 | 0.50 | 0.50 | 0.50 |
| 5 | 1.00 | 1.00 | 0.75 |
| 8 | 1.00 | 1.00 | 1.00 |

(grounded rate; abstention accuracy was 1.0 and hallucination 0.0 in all nine runs — raw output in `logs/sweep/`, summary in `logs/threshold_sweep.txt`)

I first picked k=5 / 0.40, because it matched k=8 with fewer tokens. Then I tried the short form from the brief in `ask`, "What killed Julia Stoner?", and it still abstained. The short version only works at k=8, and the answer actually cites [8] — the last chunk in the window, scored 0.361. That's how close it was. I settled on **k=8, threshold 0.40**.

The k=5 pass on q2 turned out to be worse than it looked (see Citations).

On the abstain side, the threshold did less work than I expected. "Who is Professor Moriarty?" scored **0.473**, and my own q7 (the Baskerville hound) scored **0.432**. Both are above 0.40 and not far below the real questions at around 0.51, so no threshold separates them cleanly. They abstained because of the grounding instruction in the system prompt. Only the out-of-domain boiling-point question (0.144) was stopped by the score gate itself. I kept 0.40 rather than 0.30 because it gives more margin above that kind of question, and at 0.45 q1 abstained once at k=5.

## Citations

`validate_citations` didn't catch a fabricated marker in any of my runs — every [n] the model wrote mapped to a real chunk, and none of the final rows lost their grounded tick for an invalid citation. I also made it read grouped markers like `[1, 2]` as well as `[1]`, because missing those would undercount grounded answers. I can't tell from the eval table whether any answer actually used the grouped form.

What it can't catch is a citation that's real but weak. q6 ("Why did the Red-Headed League pay Jabez Wilson...") passes as grounded, but the answer says the League was "a scheme" to reach a bank vault and cites the chunk about the thirty-two pounds. It never mentions the tunnel from the cellar, which is the actual reason. The marker is valid and points to the right story, so the eval counts it. The bigger one was q2 at k=5. It scored grounded, and the answer read well: "Julia Stoner was killed by a snake that Doctor Roylott had sent through the ventilator", citing [4] and [5]. When I checked the five chunks it was given, none of them contain the word "snake" or "adder". Chunk [5] starts mid-word — "gh the ventilator, and it had attacked the only living thing in the room beyond" — because the 80-character overlap cut off the sentence that names the snake. The model filled the gap from what it already knew about the story. That's a hallucination with valid citations, and my eval counted it as a pass. At k=8 the chunk that actually names the swamp adder is in the window, so the final answers are supported, but the eval itself would never have told me the difference.

Checking whether the cited chunk really supports the claim would need something like faithfulness scoring, not just range checking.

## Reading your eval table

Final run (`mp3_eval.txt`, k=8, threshold 0.40, 6 answerable + 3 abstain):

- Retrieval hit rate **1.0**
- Grounded & cited rate **1.0**
- Abstention accuracy **1.0**
- Hallucination rate **0.0**

The weakest number is the one that looks best: retrieval hit rate. It was 1.0 even in the k=3 run that failed half the questions, because it only checks that the right *story* appears in the top-K, not the chunk with the answer. Every chunk for q2 came from `02_speckled_band.txt`, so it counted as a hit while the answer chunk sat at rank 7. If I changed one thing, I'd measure the hit at chunk level (does the chunk containing the expected facts appear in top-K), so it fails when retrieval actually fails.

The other caveat is size. Nine questions is a small set, and abstention relies on the prompt for anything that sounds like it belongs in the corpus. I'd want more plausible-but-missing questions (other Holmes characters and cases) before trusting the 0.0 hallucination rate.

## One surprise

I expected tuning abstention to be the hard part. It turned out the assistant was already refusing too often, and for an honest reason: it wasn't being shown the answer. Raising k from 3 to 8 fixed both failing questions without touching the threshold or the prompt. The fix was retrieval, the cheapest change available, and the abstentions were the clue. The second surprise was the opposite case: the k=5 run that looked perfect on q2 was the model answering from memory, and nothing in the table showed it.
