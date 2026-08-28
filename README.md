# hassan-stock

- `index.html` — the full scanner: barcodes, quantities, placement.
- `lite.html` — **the lite sweep.** Walk the shop, record video and photos per
  section, tap a shelf number to mark where you are, move on. No barcodes, no
  quantities, nothing touched. Products are read out of the media afterwards.

Both write the same location codes: `FIXTURE.FACE.SECTION.LEVEL`, L1 the bottom
shelf, cap ends with no section segment (`G1.CAP.L7`).
