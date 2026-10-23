# GEO measurement prototype (design)

Pre-work for the Fall 2026 SPARK research project on Generative Engine Optimization (GEO) at Northeastern University (Khoury College, San Jose; PI: Dr. Akram Bayat).

Written in August 2026, before the project started, this design asks one question: **can citation by AI search engines be measured precisely enough, at an affordable cost, to detect the effect of a realistic content change?** If the measurement floor sits above the effect size, a GEO score, a rewriting workspace and a citation dashboard would all be reporting noise.

The design ([docs/superpowers/specs/2026-08-10-geo-measurement-prototype-design.md](docs/superpowers/specs/2026-08-10-geo-measurement-prototype-design.md)) covers:

- an observational panel first, with a controlled site added later as a treated unit;
- sanctioned APIs only (no scraping of consumer AI interfaces or search result pages);
- repeated runs per query, clustered by query, instead of a single nightly run;
- a candidate risk set for the variance model, and a go/no-go rule based on the minimum detectable effect.

It is a design document only. The ideas were carried forward into:

- [FootnoteOne](https://github.com/shehral/footnoteone), a measurement-first command-line tool (October 2026);
- [Citation Equity](https://github.com/shehral/citation-equity), the open-source build of the project's original three-module design (October 2026).

Author: Mohammad Ali Shehral. License: MIT.
