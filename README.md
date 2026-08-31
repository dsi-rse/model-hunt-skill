# model-hunt

A [Claude Code](https://claude.com/claude-code) skill for running disciplined, time-boxed searches
for the best supervised machine-learning model on a dataset.

A **model hunt** is a campaign, not a script. You hand Claude a dataset, a prediction target, an
evaluation metric, and a wall-clock budget; it comes back with the best model it could find, the
evidence that it is the best, and enough records that you (or a future Claude session) can
re-analyze the search months later without re-running it.

The skill applies to any *supervised* problem and any modality — tabular, text, image, audio, video,
graph, geospatial, time-series — and any prediction type: classification, regression, segmentation,
detection, ranking, forecasting. The defining criterion is that the model is a **predictive
function** learned from labeled examples and scored by a metric.

## A self-contained campaign

The skill is built for a **complete run from a blank slate**: you define the problem, and an
hours-long campaign returns its solution. It starts by helping you decide everything — what the
evaluation metric should be, how the folds are constructed, whether the winner is the strict metric
maximum or the simplest model statistically equivalent to it, what the candidate space even is — and
it treats all of those as open questions to be settled with you before any model is trained. That
design phase is most of the skill's value, and it is deliberately expensive.

That makes it the wrong tool for extending a search you have already run. If you have a framework,
a metric, and a validated protocol in place and you just want to add a few configurations, fold in
new data, or re-run last quarter's hunt against a new model family, the right move is to keep all of
those fixed and extend the existing harness. See [#Alternatives](Alternatives).

## Install

**As a plugin** (recommended):

```
/plugin marketplace add dsi-rse/model-hunt-skill
/plugin install model-hunt
```

**As a plain skill:**

```bash
git clone https://github.com/dsi-rse/model-hunt-skill.git
cp -r model-hunt-skill/skills/model-hunt ~/.claude/skills/
```

Use `.claude/skills/` inside a project instead of `~/.claude/skills/` to scope it to one repo.

## Use

Describe what you want, in plan mode if you want to review the campaign before it burns your
budget:

> I need a model to predict `churn` from the customer table in `data/`. Consider random forests,
> boosted decision trees, and neural networks. The metric is balanced accuracy — customers who
> churn are rare and I care about them equally. Rows from the same account are not independent.
> You have 8 hours, 16 CPU cores, and an RTX 3060. Put the final model in `models/`, detailed
> notes in `docs/experiments/`, and give me a short `summary.md` I can paste into the PR.

Anything you leave out, Claude will ask about. Anything it can find in the repo, it will find first.

## What it asks you

Front-load these in your prompt and the intake goes quickly:

- **The target and task type** — what predicts what, and what kind of prediction.
- **The evaluation metric.** The most important question, because it determines which model you get.
  If you want advice, ask for it — the skill will propose one with reasoning and flag it for your
  review rather than adopting it silently.
- **What makes two examples non-independent** — same subject, site, session, source image, adjacent
  in time or space. This decides whether folds are random, grouped, spatial, or temporal, and
  getting it wrong inflates every score and picks the wrong winner.
- **Which model families to consider**, and any hard deployment constraints (model size, latency,
  target hardware, interpretability, license).
- **The wall-clock time-box**, and the machine's resources (it will probe these and ask you to
  confirm).
- **Where every output goes**, by name. The skill assumes no filenames and no directory
  conventions — it asks.

## What it does

```
Phase 0  Intake            leading questions; infer from the repo first, then ask
Phase 1  Reconnaissance    repo conventions, prior PRs, data shape, hardware probe
Phase 2  Plan          ◀── a safe session boundary; the plan is self-sufficient
Phase 3  Harness           results ledger, resumability, smoke-test every branch
Phase 4  Round 1           broad, cheap, deliberately speculative — eliminate
Phase 5  Round 2           narrowed, light cross-validation — rank
Phase 6  Round 3           gold-standard cross-validation on all data — select
Phase 7  Final model       retrained on everything, plus the per-fold models
Phase 8  Documentation     the durable record, and a short summary
```

Throughout, it shepherds the run: checking on a set cadence that something is actually running, that
the GPU isn't sitting idle, that the log is advancing, and that nothing is blocked waiting for an
answer — fixing crashes and resuming from checkpoints rather than restarting.

## What you get

- **A final model** trained on all eligible data, plus the per-fold models that make its reported
  score reproducible.
- **An append-only results ledger** — one row per configuration × fold, with the full
  hyperparameters, per-fold scores, and enough per-example raw detail that a *changed metric
  threshold costs a re-analysis, not a re-run*.
- **Durable documentation**, git-committed, written for a future Claude session to query. It keeps
  the intermediate numbers and the configurations that lost, so the next campaign doesn't repeat
  your experiments.
- **A short summary** for pasting into a comment thread, if you want one — deliberately kept to a
  screenful.

## Design philosophy

- **The evaluation metric is the specification.** It decides which model you end up with, more than
  any architecture choice. It is never picked silently.
- **Input representation is a search axis.** How the data is presented to the model is frequently a
  bigger lever than the model, and it is the axis most often skipped.
- **Never eliminate on a difference smaller than its uncertainty.** Error bars come before rankings,
  and ties are broken toward the smaller, simpler model.
- **Retain every raw number.** Decisions must be re-derivable without re-running anything.
- **Report what failed.** The configurations that lost, and by how much, are results.

## Layout

```
.claude-plugin/          plugin and marketplace manifests
skills/model-hunt/
  SKILL.md               the campaign orchestrator
  references/
    intake.md            the leading questions, and what to do when they go unanswered
    search-space.md      what to vary; heuristic scales by family and modality
    protocol.md          rounds, splits, statistics, elimination, re-budgeting
    resources.md         hardware probing, parallelism, long runs, shepherding
    reporting.md         the ledger schema and the two documentation artifacts
  templates/
    plan.md  documentation.md  summary.md
```

## Contributing

Issues and pull requests welcome. The skill is documentation only — no runtime dependencies, no
pinned framework versions, nothing to break when a library changes. The heuristic hyperparameter
ranges in `references/search-space.md` are explicitly framed as priors rather than limits, and the
skill is instructed to check current prior art before trusting them.

## Alternatives

If your problem isn't quite the one above, these are probably better fits.

| If you want to… | Look at |
|---|---|
| **Search harder, with the metric already fixed.** Algorithmic budget-aware optimization, much better at the search itself than any prose instruction. | [FLAML](https://microsoft.github.io/FLAML/) (wall-clock `time_budget` is a first-class parameter), [AutoGluon](https://auto.gluon.ai/) (finds that stacked ensembles beat seeking a single best model), [auto-sklearn](https://automl.github.io/auto-sklearn/) (meta-learns from prior datasets). None of them will ask what your metric should be, choose a fold scheme from your data's dependence structure, or write anything a human reads. |
| **Just tune one model you have already chosen.** A script, not a campaign. | The various hyperparameter-tuning Claude skills on [claudemarketplaces.com](https://claudemarketplaces.com/skills/category/data-science) and [mcpmarket](https://mcpmarket.com/), or [Optuna](https://optuna.org/) directly. |
| **Track experiments over months.** Runs, metrics, artifacts, and comparison UI. This skill's results ledger is a deliberately minimal stand-in for these. | [MLflow](https://mlflow.org/), [Weights & Biases](https://wandb.ai/), [Neptune](https://neptune.ai/). |
| **Build institutional memory across many experiments and many people.** Capture what worked and what failed so the next person doesn't repeat it. | Sionic AI's `/advise` + `/retrospective` pattern, [described here](https://huggingface.co/blog/sionic-ai/claude-code-skills-training) — complementary to this skill rather than competing with it. |
| **Chase a leaderboard score on a clean, well-specified benchmark.** | Agentic ML systems such as AIDE, [ML-Agent](https://arxiv.org/abs/2505.23723), and [AutoMLGen](https://arxiv.org/abs/2510.08511), benchmarked on [MLE-bench](https://arxiv.org/abs/2410.07095). Note that a benchmark hands you the metric, the splits, and clean data — which is precisely the work this skill exists to do. |
| **Do unsupervised work, tune an LLM prompt or RAG pipeline, or apply a model you already have.** | Out of scope here; the skill will say so. |

## License

BSD 3-Clause. See [LICENSE](LICENSE).
