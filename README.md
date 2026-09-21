# Rhythm of Thought: Tracing the Dynamics of Deliberation in Reasoning

This repository holds the data and code for the paper.

In this paper, we reanalyse the think-aloud recordings of Byrd et al. (2023, Study 2), in
which 102 participants solved the 10 items of the verbal Cognitive Reflection Test while
speaking their thoughts aloud, with one recording per item. We transcribed each recording
with word-level timestamps, used a large language model to segment it into chunks (single
thoughts or reasoning steps), and then scored every chunk on four deliberation functions
(generation, justification, control, regulation), on expressed confidence, and on two
auxiliary dimensions (repetition of the item and answer selection). The resulting corpus
contains 4,011 scored chunks drawn from 969 trials.

## Citation

> Beucler, J., & De Neys, W. *Rhythm of Thought: Tracing the
> Dynamics of Deliberation in Reasoning*.

The recordings and the participant-level data were originally collected by:

> Byrd, N., Joseph, B., Gongora, G., & Sirota, M. (2023). Tell us what you really think:
> A think aloud protocol analysis of the verbal cognitive reflection test.
> *Journal of Intelligence*, *11*(4), 76. https://doi.org/10.3390/jintelligence11040076

## Repository structure

```
Analyses/     all scripts (see the pipeline below)
Data/         source data, recordings, and transcription output
Output/       everything the scripts produce: scored corpus, figures, tables
Environment/  R lockfile and pinned Python requirements
```

Variable definitions for the main analysis and human-validation datasets are provided in the
[Excel codebook](Data/metadata.xlsx). Human raters are labelled consistently as
`Rater01`–`Rater07` across both rating sheets and all measures.

## Pipeline

There are four consecutive stages that run in the order below.

### 1. Speech to text

`Analyses/speech_to_text_conversion.ipynb`

| | |
|---|---|
| Reads | `Data/Byrd_2023_Study_2.xlsx`, the mp3 files in `Data/Byrd_2023_Study_2_recordings/` |
| Produces | `Data/audio_data_with_transcriptions_and_alignment.pkl`, `Data/data_long.csv` |

This notebook transcribes each mp3 with WhisperX and aligns the transcript to word-level
timestamps (saved in `Data/audio_data_with_transcriptions_and_alignment.pkl`). It then
assembles the result into one row per participant-item trial (in `Data/data_long.csv`).

### 2. Chunking and scoring

`Analyses/chunking_and_scoring_LLM_calls.ipynb`

| | |
|---|---|
| Reads | `Data/data_long.csv` |
| Produces | `Output/chunked_transcriptions.csv`, `Output/aligned_chunks.csv`, `Output/scored_chunks.csv` |

This notebook segments each transcript into chunks (in `Output/chunked_transcriptions.csv`)
and realigns them to the word-level timestamps, assigning a start and an end time to each
chunk (in `Output/aligned_chunks.csv`). Each chunk is then scored on the seven measures,
producing the final scored corpus (in `Output/scored_chunks.csv`).
The scoring uses `gpt-5.1` (OpenAI) and samples the model eight
times per chunk, which is what the columns ending in `_mean` average over.

All analyses in stage 4 use `Output/scored_chunks.csv`.

### 3. Human validation

There are two scripts in this stage. The first scores a validation sample using several
prompt variants and model configurations; the second compares these scores with ratings
from seven human raters. The sample contains 99 chunks (originally 100, with one duplicate).

`Analyses/prompt_optimization_LLM_calls.ipynb`

| | |
|---|---|
| Reads | `Data/validation_ratings_summary.csv` |
| Produces | `Output/Prompt_optimization/*.csv` (10 files, one per configuration) |

We compare ten configurations, which differ in prompt wording, in the model used, and in
reasoning effort. The models are `gpt-5.1` (OpenAI) and `Llama-3.3-70B-Instruct`
(Hugging Face). The notebook saves the LLM ratings for the validation chunks in one CSV
per configuration (in `Output/Prompt_optimization/`).

`Analyses/prompt_optimization_analysis.rmd`

| | |
|---|---|
| Reads | `Data/chunk_validation_ratings.xlsx`, `Output/Prompt_optimization/*.csv` |
| Produces | `Output/LLM_scoring_validation/*.svg` |

This script assesses inter-rater reliability and evaluates each configuration by
correlating its scores with the mean human ratings from `Data/chunk_validation_ratings.xlsx`.
It saves the reliability and human–LLM comparison figures in `Output/LLM_scoring_validation/`.
This comparison allows us to select the final configuration used in the rest of the paper.

### 4. Main analyses

`Analyses/main_analyses.Rmd`

| | |
|---|---|
| Reads | `Output/scored_chunks.csv` |
| Produces | `Output/Figures_paper/`, `Output/Preprocessing/`, `Output/Trajectories/`, `Output/Confidence/`, `Output/Predicting_accuracy/`, `Output/Robustness_analyses/` |

This is the main analysis script. It fits generalized additive mixed models for the
deliberation trajectories and elastic-net logistic regressions for accuracy prediction, and
it runs the confidence and lagged analyses along with the robustness checks and
simulations. Outputs are grouped by analysis in the corresponding folders listed above.
The numbered figures in the paper are saved in `Output/Figures_paper/`.

### Supplementary: open-ended labelling

`Analyses/open_ended_labeling.ipynb`

| | |
|---|---|
| Reads | `Output/scored_chunks.csv` |
| Produces | `Output/Bottom_up/` |

This notebook runs a supplementary open-ended labelling analysis. It labels the chunks
with categories generated by `deepseek-v4-flash` (OpenRouter) and compares these categories
with the four predefined deliberation functions. The labels and comparison outputs are
saved in `Output/Bottom_up/`.

## Reproducing the analyses

### Reproducing the main results

The intermediate outputs and scored corpus are included in the repository. Open
`Deliberation.Rproj`, install the R packages as described below, and knit
`Analyses/prompt_optimization_analysis.rmd` for the validation results or
`Analyses/main_analyses.Rmd` for the main analyses, figures, and tables.
Both documents produce HTML and resolve data and output paths from the project root.

On its first run, `main_analyses.Rmd` estimates the models and saves them in
`Output/Saved_models/`. Subsequent runs load the saved models, reducing computation time.

### Re-running speech-to-text and LLM analyses

To regenerate the transcriptions and chunk scores, run the Python notebooks in stages 1
and 2. The notebook in stage 3 regenerates the LLM validation scores, and
`Analyses/open_ended_labeling.ipynb` runs the supplementary labelling analysis.
Each notebook resolves the project root in its first code cell and can be opened with
the working directory set to either the root folder or `Analyses/`. Run cells in order.

Enter your API keys in the corresponding `API_KEY = ""` placeholders in each LLM
notebook. Stage 2 requires an OpenAI key; stage 3 requires OpenAI and Hugging Face keys;
open-ended labelling requires an OpenRouter key.

Note that LLM outputs are not deterministic: re-running these analyses may produce
slightly different results.

## Requirements

### R

Open `Deliberation.Rproj` using R 4.5.1. From the repository root, install the
recorded package versions into your current R library:

```r
install.packages("renv")  # Only needed if renv is not already installed.
renv::restore(lockfile = "Environment/renv.lock", library = .libPaths()[1])
```

HTML rendering also requires Pandoc, which is included with RStudio.

### Python

Use Python 3.11.8. From the repository root, create a virtual environment and install
the recorded package versions:

```sh
python3.11 -m venv .venv
.venv/bin/python -m pip install -r Environment/requirements.txt
```

Select `.venv/bin/python` as the notebook kernel. On Windows, use
`py -3.11 -m venv .venv` and `.venv\Scripts\python.exe` instead.
Speech-to-text conversion also requires FFmpeg on the system path.

Only the dependency records are included and packages are installed locally with the
commands above.

## Licence

This material is released under the
[Creative Commons Attribution-NonCommercial 4.0 International Licence](LICENSE)
(CC BY-NC 4.0), which lets you share and adapt it for non-commercial purposes as long as
you give credit.

## Contact

Jérémie Beucler — jeremie.beucler@gmail.com
LaPsyDÉ, CNRS, Université Paris Cité
