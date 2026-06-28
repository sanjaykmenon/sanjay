# EB-1A OCR and extraction pipeline blog playbook

Status: draft planning notes. This is not the final post.

## Working title

How we built an OCR and legal extraction pipeline for 5,000 EB-1A decisions

## Central thesis

We were not trying to avoid powerful models. We were trying to build a pipeline where quality could be measured, costs were predictable, and legal errors were visible.

Frontier models such as Claude and OpenAI models are useful as judges, benchmarks, and silver label generators. They are not a replacement for a system that has OCR checks, strict schemas, legal prompts, validation, and review.

The production path needed control, repeatability, and cheap iteration across about 5,000 public AAO decisions.

## What the post should explain

The post should explain how we turned public USCIS AAO EB-1A PDF decisions into structured legal data.

The hard part was not only OCR. The hard part was preserving legal meaning through OCR, prompt design, schema design, validation, and review.

The post should make clear that this is a legal extraction system, not a normal document extraction task.

## Legal primer

The reader needs enough EB-1A context to understand why a normal extraction prompt fails.

Cover these points:

* EB-1A is an immigrant visa category for people with extraordinary ability.
* A petitioner usually has to meet at least 3 of 10 regulatory criteria.
* Meeting 3 criteria is not the end of the analysis.
* USCIS then performs a final merits review.
* Final merits asks whether the full record shows sustained national or international acclaim.
* AAO decisions often discuss the Service Center finding, the petitioner argument, and the AAO finding. These are different things.
* Some decisions say a criterion was satisfied but still deny the case at final merits.
* Some decisions do not reach every criterion because the case fails somewhere else.
* Redactions mean the model must not fill in missing names, employers, or facts.

This section should explain why the prompts separated metadata, criterion evidence, adjudicator findings, and final merits.

## Corpus section

Use the actual numbers from the project.

* 5,001 PDFs.
* 35,777 pages.
* About 7.2 pages per document.
* 5,001 extracted text documents existed in the local corpus.
* The corpus quality report flagged 676 documents.
* The corpus had duplicate text groups, so source level deduplication mattered.
* Text quality varied. Some documents had OCR damage, weak pages, or low text density.

The point is that the corpus looked simple at first, but legal extraction exposed the weak spots.

## OCR decision path

This should be a major section.

Tell the story in order:

1. Start with embedded PDF text because it is fast and cheap.
2. Add Tesseract fallback for weak pages.
3. Test RapidOCR with a page level bake off.
4. Notice that surface quality metrics can mislead. Fluent text is not always legally complete text.
5. Move toward PaddleOCR-VL because the goal is document parsing, not only raw text.
6. Try Ollama for PaddleOCR-VL and find that it is not a real path for this model. The model uses a custom architecture and does not expose image input through the normal Ollama vision API.
7. Check hosted options. NVIDIA build looked promising at first, but PaddleOCR there was self hosted, not a free hosted API.
8. Consider Baidu AI Studio as a hosted PaddleOCR-VL option, with account and data concerns.
9. Decide that Modal or a similar GPU self host path is the best full run option.

The lesson is that the best OCR path was not obvious. It came from testing the real constraints.

## Local, cloud, and hosted API tradeoffs

This section should compare the options plainly.

### Local

Local gives privacy and zero API cost. It also gives full control over the model and the prompts.

The problem is wall clock time. Full local OCR and extraction could take 2 to 3 weeks on the Mac. OCR and extraction also compete for the same local GPU, so they cannot both run quickly at the same time.

### Self hosted cloud on Modal or a similar platform

Modal gives most of the control of self hosting without managing servers.

The estimated full run cost was about $15 to $30 for OCR plus extraction. It would likely fit within free credits. It also turns a multi week local run into a run that can finish in hours.

This path keeps model choice under our control. It also lets us rerun the same pipeline when prompts or validation change.

### Hosted model APIs

Hosted APIs are the simplest operationally.

They are useful because strong frontier models can reason well over legal text, handle hard cases, and judge calibration examples.

But they have tradeoffs:

* They cost more than GPU time for repeated multi pass extraction.
* They bill by tokens, and the pipeline resends context across passes.
* They may require a model swap, which means we must revalidate quality.
* Their behavior can change over time.
* Privacy becomes more important if the same pipeline later handles private client documents.
* They do not remove the need for schemas, validation, and review.

The post should not say hosted models are bad. It should say they have the best role as judges, benchmarks, and silver label generators.

## Role of Claude and OpenAI models

Use powerful hosted models where they are strongest:

* Review hard calibration cases.
* Act as a Tier B legal correctness judge.
* Generate silver labels for the 50 document calibration set.
* Benchmark the local or self hosted pipeline.
* Help find prompt failures and schema gaps.

Do not use them as a reason to skip system design.

A stronger model can still confuse Service Center findings with AAO findings. It can still miss a redaction. It can still return a plausible but wrong legal extraction. The system still needs grounding, schemas, and review.

## Extraction design

Explain the movement from one pass to multi pass extraction.

The one pass extractor asked the model to produce metadata, all criteria, final merits, quotes, and reasons in one response. It caused problems:

* Invalid criteria such as unknown.
* Quote grounding issues.
* Long document truncation.
* Confusion between Service Center findings and AAO findings.
* Missing criteria or final merits.

The improved system used an evidence first multi pass design:

1. Metadata.
2. Document map.
3. Criterion evidence inventory.
4. Criterion adjudication.
5. Final merits.
6. Deterministic merge and validation.

Each pass had a smaller legal job. The model was not asked to solve the whole case at once.

## Prompt design

The prompts should be presented as legal reasoning instructions, not generic extraction prompts.

Important rules:

* Extract only facts grounded in the decision.
* Do not inspect or pretend to inspect the underlying exhibits.
* Separate the petitioner argument from the evidence.
* Separate Service Center findings from AAO findings.
* Do not treat final merits reasoning as an initial criterion finding.
* Preserve weak or rejected evidence.
* Use exact quotes when possible.
* Return not reached when the AAO did not reach a criterion.
* Do not infer redacted names or missing fields.

This section should include one short example. For example, if the Service Center found judging met but the AAO did not reach judging on appeal, the extraction cannot say the AAO found judging met.

## Quality bar

Use the two tier quality frame.

### Tier A

Tier A is deterministic grounding.

A document passes Tier A when:

* The schema is valid.
* Paragraph references exist.
* Quotes are grounded.
* Quote found rate is at least 0.90.

The measured result was 47 out of 50, or 94 percent, under OCR tolerant quote matching.

Strict quote matching was only 42 percent. That was a measurement artifact. Many quote failures came from OCR corruption or overly strict matching, not model paraphrase.

### Tier B

Tier B is legal correctness.

It asks whether:

* Findings match the decision text.
* Evidence completeness is good.
* Final merits linkage is right.
* The extraction reflects the legal reasoning in the decision.

The code checker gave 56 percent on v2, but the project notes say that checker is brittle. Certifying Tier B needs a stronger reading judge, such as Opus, or human review.

The point is that grounding is necessary but not enough. A grounded extraction can still be legally wrong.

## Experiment results to include

Use these results in the post:

* The first full multi pass attempt failed because of context size errors.
* Reloading Gemma at 16,384 context and capping output fixed truncation.
* The resilient runner made the 50 document run complete.
* Hard failures went from blocking the run to 50 out of 50 documents completing.
* Strict Tier A grounding was 42 percent.
* OCR tolerant Tier A grounding was 94 percent.
* Metadata posture reached 100 percent after OCR tolerant whole document scanning.
* Role and field remained weak at 40 percent and 18 percent.
* Tier B code checkable correctness was 56 percent on v2, but that checker could not certify legal correctness.

The post should make clear that measurement itself had to be debugged.

## Hard issues section

Cover these issues directly:

* Context window failures.
* JSON truncation.
* Schema violations.
* Quote matching false negatives.
* OCR corruption in legal citations.
* Cross criterion bleed.
* Role and field extraction failing because the key text was OCR damaged.
* Post processing heuristics overriding a correct model answer.
* Keyword based legal correctness checks reaching their limit.

## Ending

End with the system lesson.

The system worked because it combined domain prompts, smaller extraction passes, strict schemas, deterministic validation, a calibration set, error analysis, and cost tests.

The lesson is not that one model solved the problem. The lesson is that legal document extraction needs a measured system around the model.

## Suggested length

Target 2,500 to 3,500 words.

Suggested balance:

* Legal primer: about 20 percent.
* Pipeline and experiments: about 60 percent.
* Cost and deployment tradeoffs: about 20 percent.
