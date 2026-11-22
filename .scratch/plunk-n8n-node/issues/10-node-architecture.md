# Node architecture

Type: grilling
Status: open
Blocked by: 01, 03, 05
Map: [../map.md](../map.md)

## Question

Declarative, programmatic, or hybrid style; how is the code split into modules (resources, operations, shared transport, types); where do the strict seams go so tests and fallow boundaries are natural?

## Notes from Engineering standards

- fallow needs one explicit zone per resource (no `autoDiscover`) and `requireAllFiles`; decide the zone names here.
- Only `transport/` may perform HTTP; resources never import each other; `shared/` depends on nothing. Strict type-checked ESLint makes a typed transport wrapper returning `unknown` and typed parameter getters the natural seams (see ticket 05's findings).
- Coverage demands 100 % for `transport/` and `shared/`, so keep them small and pure.
