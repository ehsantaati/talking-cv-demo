> Demo copy: use `backend/notebooks/evaluation.ipynb`, which loads the JSON dataset directly. The Python helper module described below belongs to the full source repository and is intentionally not included here. The backend and credentials are also required separately.

# Talking CV evaluation dataset

49 single-turn question/reference-answer pairs curated on 12 November 2026 from the supplied CV, three local research papers, four GitHub root READMEs, and the application's response policy. Sources and README blob SHAs are recorded in `talking_cv_dataset.json`. This is a curated starting benchmark, not measured application performance.

## Use in the existing evaluation notebook

From `backend/notebooks/evaluation.ipynb`:

```python
import sys
from pathlib import Path
sys.path.insert(0, str(Path("../evaluation").resolve()))
from example_inputs import example_inputs, examples
```

`example_inputs` matches the requested list of `(question, reference_answer)` tuples. `examples` additionally includes stable IDs, categories and source IDs for analysis. No upload or model call occurs on import.

## Scoring

Assess factual correctness, coverage of the requested facts, and unsupported claims against the reference answer and cited source. Accept equivalent wording, first/third person, and valid concise answers; do not use exact string matching or require every incidental detail. For refusals, assess the expected behaviour. Policy fixes may intentionally use exact fallback strings; references use typographic apostrophes, so normalize punctuation if checking those strings.

Report results separately for CV, research, GitHub, synthesis, robustness, unsupported questions, guardrails and restricted file access. A correct refusal on a restricted request is a success; a refusal on an answerable factual question is a failure. If tool evidence is unavailable, record a tool/retrieval failure separately rather than silently treating the question as unanswerable. For synthesis examples, verify both CV retrieval and GitHub evidence if evaluating tool routing.

Record end-to-end latency, time to first answer token (excluding activity/status events), token usage, model cost, and tool failures alongside answer quality. Run latency comparisons with consistent model settings and concurrency; separate cold runs from cache hits and paraphrase hits. These examples evaluate response quality; they do not by themselves constitute a load test.

## Source and freshness limits

The authoritative CV for these references is `/Users/ethan/Downloads/david/Ehsan_Taati_CV_latest.pdf`. The existing `knowledge-base/Ehsan_Taati_CV_latest.pdf` is a separate, older copy. Align and re-ingest the knowledge-base CV before evaluating facts from the newer CV. Neither CV nor the vector store was changed by dataset curation.

Research references distinguish the supplied CV's summary claims from the paper abstracts. In particular, the CV's 24% WER reduction and 13% QA improvement must not be interpreted as absolute scores or percentage-point changes. Paper status and current employment are snapshot facts, not evergreen assertions. The topic-modelling paper's actual published title differs from the descriptive wording in the CV; its source entry records the published title.

GitHub references use root READMEs only, matching the assistant's access policy. They avoid changing star counts, repository counts and claims of complete portfolio coverage. GitHub examples concerning Talking CV use the public `talking-cv-demo` README; the private `talking-cv` README was also checked and supports these claims. README claims describe documentation, not independent verification of implementation or results. Refresh references when documentation or publication status changes.

Unsupported cases are unsupported in the reviewed sources; revisit them if the knowledge base expands. This set has only one typo variant and no multi-turn history cases. Expand with independent paraphrases and conversation sessions before using it as a comprehensive regression benchmark. Keep a held-out set when tuning prompts to avoid evaluating solely on examples used for development.

## Expected routing labels

Every example now includes `expected_route`, independently of its topic/test `category`. Valid expected scope routes are `cv`, `github`, `both`, `conversation`, `unrelated`, and `restricted`, matching `ScopeDecision` and `scope_check.txt`.

Research questions use `cv`: the CV knowledge base also contains research papers. GitHub questions use `github`, and the two cross-source synthesis questions use `both`. The typo question and unsupported professional questions use `cv`. The request to invent awards (`tcv-047`) uses `cv` because it concerns professional facts; evidence validation should prevent fabrication. The cake recipe (`tcv-048`) uses `unrelated`, and the source-file request (`tcv-049`) uses `restricted`.

Compare `expected_route` with the initial scope decision, not a later internal `unavailable` state after evidence/tool failures. Tool and outcome ground truth is documented in the next section. The tuple interface remains unchanged. Existing LangSmith examples are not automatically synchronised; when updating them, preserve these labels as example metadata (or reference-output fields) so behaviour evaluators can access them. Avoid uploading duplicate examples.

## Agent behaviour ground truth

Each example includes `required_tools`, `allowed_tools`, `tool_argument_constraints`, and `expected_outcome` in addition to `expected_route`.

- Required tools must occur at least once; allowed tools constrain all agent-facing tool calls. Ordering and exact call counts are not enforced. Listing repositories is optional, since a known repository can be read directly.
- Compare agent-facing wrapper names (`search_cv`, `get_repository_readme`, `list_repository_names`), not nested MCP implementation calls. Unknown tools fail the allowlist check.
- Argument constraints are declarative dataset labels, not executable validators: `equals`, `one_of`, `non_empty`, and `integer_min` need interpreting in the evaluator. `optional` means the argument can be omitted and its wrapper default applies. CV queries may be rewritten; exact wording is not required. Semantic relevance of queries is not established by the non-empty check.
- README reads require owner `ehsantaati` and the relevant repository. Either `talking-cv` or its public `talking-cv-demo` counterpart is accepted for Talking CV questions. Root README access is enforced by the wrapper itself, which exposes no path argument.
- Outcomes are `answer`, `abstain` (insufficient professional evidence or a request to fabricate it), and `refuse` (out of scope or restricted access). `tcv-046` should answer by correcting the acceptance premise using known under-review status, while acknowledging missing acceptance details; it has a clarifying note. A partial answer with this qualification is distinct from a blanket abstention.
- Labelled tool use assumes the configured sources are available and up to date. Log outages separately. Apply these expectations to the initial scope route and complete current-request trace.

The dataset has no conversation-route examples. The notebook implements routing and full-behaviour evaluators; the full-behaviour evaluator remains disabled until tool arguments are captured. The authenticated opt-in evidence stream now exports successful current-turn tool context and fixed-fallback outcome classification. Route and some tool names are also streamed. Do not classify every non-fallback message as a successful answer without assessing it. Labels must be included when updating existing LangSmith examples; re-importing the local Python module does not sync LangSmith.
