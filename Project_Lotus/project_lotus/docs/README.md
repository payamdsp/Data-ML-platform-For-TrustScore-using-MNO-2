Generating the docs
----------

Use [mkdocs](http://www.mkdocs.org/) structure to update the documentation. 

Build locally with:

    mkdocs build

Serve locally with:

    mkdocs serve

## Pages and purpose

This site documents data-platform operations; it does not execute the jobs.
The pages under `docs/` cover local setup, Bronze schema validation and the TU
PortPS feeds, Silver transformations, quarantine, transient EMR recovery, and
the Bronze-to-Silver-to-Gold orchestration. The orchestration page is a design
and handoff document: it distinguishes planned orchestration from code that
has been implemented.

## Build and preview

Run the documentation commands from this directory, where `mkdocs.yml` lives:

```bash
python3 -m pip install mkdocs
mkdocs build --strict
mkdocs serve
```

`mkdocs build` writes the generated site to `site/`; do not commit that build
output unless the release process explicitly requests it. Preview cross-links
and code examples before editing a runbook. For an end-to-end data run, follow
the specific Lambda, Silver, EMR, or Terraform procedures linked from the page
instead of running MkDocs.
