# AI-Heavy Projects

Use this reference only when AI or machine learning is central to the product and the backend contains multiple model-driven components, providers, pipelines, or repeated experimentation workflows.

Do not use this reference for a conventional backend with one isolated AI feature, such as one summarization endpoint, transcription feature, embedding call, or simple hosted-model request. Keep that behavior inside its normal module.

## Component Pipelines

Build AI-heavy workflows as pipelines of independently replaceable components with stable inputs and outputs.

```text
Input
  -> Preprocessing
  -> Model or provider
  -> Normalization
  -> Validation
  -> Business decision
  -> Persistence
```

The module's service or pipeline owns the workflow. Each component owns one transformation or inference responsibility.

## Example

An AI-heavy research system may use:

```text
modules/
`-- research/
    |-- router.py
    |-- schemas.py
    |-- service.py
    |-- repository.py
    |-- pipeline.py
    `-- components/
        |-- ingestion.py
        |-- chunking.py
        |-- embeddings/
        |   |-- local.py
        |   `-- hosted.py
        |-- retrieval/
        |   |-- semantic.py
        |   `-- hybrid.py
        |-- reranking/
        |   |-- cross_encoder.py
        |   `-- llm_reranker.py
        |-- generation/
        |   |-- local_model.py
        |   `-- hosted_model.py
        `-- citation_validation.py
```

Create only the components and folders the project actually needs.

## Replaceable Components

Alternative implementations of one component must accept equivalent inputs and return the same output shape:

```text
Chunks -> Local embeddings -> Embedding result
Chunks -> Hosted embeddings -> Embedding result
```

This lets the team change one method or model while preprocessing, retrieval, validation, and persistence remain unchanged. Do not create separate request and response types for every component function; define shared component contracts only where interchangeable implementations need them.

When a component has alternative implementations, use small instance-based component classes with the same calling contract so the pipeline can replace one implementation without changing the other stages. Keep that convention consistent within the project.

## Orchestration and Decisions

Keep pipeline ordering and business decisions in `service.py` or `pipeline.py`. Model components perform inference and return structured results; they do not own HTTP behavior, persistence, or product policy.

Treat model output as untrusted input. Validate its shape and keep deterministic business rules separate from probabilistic inference.

## Comparison and Diagnosis

When comparing models or methods, keep the other components and input data unchanged so differences can be attributed to the component being evaluated.

Make the selected component implementation, model or configuration version, execution time, and failure stage observable enough to identify regressions. Do not log sensitive raw input or output unnecessarily.

## Process Boundaries

Components can remain ordinary functions or classes in the same application. Use a separate inference process only when heavy dependencies, specialized hardware, isolation, or independent scaling genuinely require it.
