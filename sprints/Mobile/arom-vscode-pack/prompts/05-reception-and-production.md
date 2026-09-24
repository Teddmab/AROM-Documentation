# Sprint 05 Reception and production journeys

Read `docs/completion-plan/README.md` first.

## Reception

Preserve the existing repositories and mutation dependencies. Change the first step to `Qui livre les fruits ?` with recent producers, local search, and `Ajouter un nouveau producteur`.

Minimal producer creation collects only:

- name;
- village;
- telephone when available;
- supplied product.

Return automatically to the same reception draft. Then collect quantity, quality, price, photo or GPS when permitted, and show a plain-language recap and total. Save locally first and clearly distinguish draft, waiting to send, and synchronized.

## Production and quality

Derive one visible lot journey from existing production and `qualityControls` data:

- À produire;
- Production enregistrée;
- Contrôle requis;
- Libéré;
- En quarantaine;
- Rejeté.

Show exactly one next action based on state. Use the latest effective quality result and show repeat controls as history, not separate lots.

Add interruption, dependency, offline, resume, repeated-control, route, and accessibility tests.

