Generating the docs
----------

Use [mkdocs](http://www.mkdocs.org/) structure to update the documentation. 

Build locally with:

    mkdocs build

Serve locally with:

    mkdocs serve

## Pages and purpose

This MkDocs site is the Trust Score 0.5 model handover.
`docs/getting-started.md` covers Python/Spark setup and test commands, while
`docs/handover.md` describes lineage, feature, ML, and batch-scoring stages,
run ordering, current gaps, and operational cautions. These are SageMaker/EMR
ML-package instructions, not the separate Bronze-to-Silver Spark job
instructions.

## Build and preview

From this directory:

```bash
python3 -m pip install mkdocs
mkdocs build --strict
mkdocs serve
```

The rendered site is written to `site/`; it is generated output. To validate
code, use the commands in `docs/getting-started.md`. For a no-AWS walkthrough,
run the standalone notebooks in `../notebooks/README.md`; real-data and
SageMaker jobs require approved configuration, data access, and AWS roles.
