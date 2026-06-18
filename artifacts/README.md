# Synthetic-user upload artifacts

Drop real files here for the personas to upload during a run (photos of items,
receipts, PDFs, etc.). This folder is the **default** `artifacts_dir` for the
`synthetic-user` skill.

The skill lists this folder and picks a file whose type fits each upload slot
(image for an item photo, receipt/PDF for a document). Override at invocation
with `artifacts_dir: /some/other/path` to use a different set.

If this folder has only this README, uploads are skipped and logged as friction —
add at least one image and one receipt/PDF to exercise upload flows.
