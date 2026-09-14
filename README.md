# purl.openfaster.org

PURL (Persistent URL) redirect service for [OpenFASTER](https://openfaster.org)'s
ontologies and vocabularies — the identifier namespace stays stable even when
the underlying files/hosting move. Same idea as `purl.obolibrary.org`, adapted
for static/serverless hosting (Vercel) instead of Apache — see
[`ontologies`](https://github.com/OpenFASTER-Standard/ontologies)'s own
README/commit history for the research this is grounded in.

One dedicated service for the whole org, not one per project — real OBO
Foundry practice is a single central PURL system serving every OBO ontology,
not one per ontology; this repo follows that, not the old standalone
`institutional-ontology` repo's own (since consolidated into
[`ontologies`](https://github.com/OpenFASTER-Standard/ontologies)).

Not registered with [w3id.org](https://w3id.org) (the community-run
alternative) — deliberate, not yet pursued.

## How it works

`vercel.json`'s `redirects` array is the whole system — no build step, no
compiler, no YAML-to-config translation layer (OBO needs that because they
have hundreds of namespaces; we have 7, so it's still premature machinery —
revisit if the namespace count grows enough that hand-editing the rules
array becomes unwieldy).

- **Whole-file redirects** point at a **version-pinned**
  `raw.githubusercontent.com` URL for a tagged release — never `main`, so the
  identifier's target never silently changes underneath anyone. Bump the tag
  in `vercel.json` when a new release ships.
- **Term redirects** (e.g. `/io/IO_:id`) point at the term's anchor on
  self-hosted [WIDOCO](https://github.com/dgarijo/Widoco)-generated docs
  under `public/<namespace>/docs/` — WIDOCO's anchor IDs are the term's full
  IRI, so the destination fragment is just the requested URL itself
  (`/io/docs/#https://purl.openfaster.org/io/IO_0000001`). Not currently
  instantiated for any namespace — none of the 7 has real curated content
  yet, so there's nothing for WIDOCO to document; add the rule (and the
  `docs/` tree) once a namespace actually has terms worth documenting.
- **Bare namespace** (`/io`) redirects to the ontology's own GitHub repo.

## Adding a new ontology/namespace

1. Add its `robot`-built OWL file's raw GitHub URL as a new whole-file
   redirect rule (pinned to a tag).
2. Regenerate its WIDOCO docs, drop the output under
   `public/<namespace>/docs/`.
3. Add a term-redirect rule for its ID pattern, plus a bare-namespace
   redirect.
4. Update `public/index.html`'s namespace list.

## Regenerating the docs for `institutional-ontology`

From the `ontologies` repo, after
`ontologies/scripts/build-module.sh institutional institutional institutional-ontology`:

```
java -jar widoco.jar -ontFile institutional-ontology.owl \
  -outFolder /tmp/widoco-out -lang en-de -uniteSections -rewriteAll \
  -getOntologyMetadata
cp /tmp/widoco-out/index-en.html /tmp/widoco-out/index.html
rsync -a --delete /tmp/widoco-out/ path/to/purl/public/io/docs/
```

(`-webVowl` is intentionally never passed — it pulls in a bundled OWL2VOWL
dependency with a disclosed, unresolved vulnerability report as of this
writing. Everything else WIDOCO produces is unaffected.)

Known limitation: WIDOCO's `-lang en-de` translates its own UI chrome
("Classes" → "Klassen", etc.) but does not know about `IAO:0000118`
("alternative label") — so `index-de.html`'s actual term labels still show
the English `rdfs:label`, not the German alt label from the ontology. Not
fixed here; a real gap in WIDOCO, not something to silently paper over.
