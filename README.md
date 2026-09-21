# itb-plugin-fhir-terminology

The OTB Gherkin dialect for **FHIR terminology servers**: the operations of the
terminology module, in the shape the HL7 terminology-ecosystem test set uses
them.

This plugin deploys nothing. The terminology server is the system under test,
so there is no simulator to run, and the one judgement a test cannot make for
itself — does this response match the expected one? — is a matchetype
comparison that the [fhir-validator plugin](https://github.com/OpenTestBed/itb-plugin-fhir-validator)
already serves at `/itb/matchetype`. What is here is vocabulary.

## The language it adds

One actor kind, `terminology-server`, and fifteen verbs:

```gherkin
Background:
  Given TxServer is the system under test at "https://tx.example.org/r4"
  And FHIRValidator is a fhir-validator at "http://fhir-validator:8080"
  And Client is infrastructure
  And Client fetches the test material from "https://raw.githubusercontent.com/HL7/fhir-tx-ecosystem-ig/main/tests/"
  And TxServer is given the resources:
    | resource                      |
    | simple/codesystem-simple.json |
    | simple/valueset-all.json      |

Scenario: simple-expand-all
  When Client expands on TxServer with:
    """
    { "resourceType": "Parameters", "parameter": [
        { "name": "url", "valueUri": "http://hl7.org/fhir/test/ValueSet/simple-all" },
        { "name": "excludeNested", "valueBoolean": true } ] }
    """
  Then $response.status should be 200
  And $response should match the pattern in "simple/simple-expand-all-response-valueSet.json"
```

| Verb | Endpoint |
|---|---|
| `<Client> expands on <Server> with:` | `POST /ValueSet/$expand` |
| `<Client> validates a code on <Server> with:` | `POST /ValueSet/$validate-code` |
| `<Client> validates a code against the code system on <Server> with:` | `POST /CodeSystem/$validate-code` |
| `<Client> looks up a code on <Server> with:` | `POST /CodeSystem/$lookup` |
| `<Client> tests subsumption on <Server> with:` | `POST /CodeSystem/$subsumes` |
| `<Client> translates on <Server> with:` | `POST /ConceptMap/$translate` |
| `<Client> compares value sets on <Server> with:` | `POST /ValueSet/$compare` |
| `<Client> validates a batch on <Server> with:` | `POST /ValueSet/$batch-validate-code` |
| `<Client> reads the capability statement of <Server> as $x` | `GET /metadata` |
| `<Client> reads the terminology capabilities of <Server> as $x` | `GET /metadata?mode=terminology` |

and five that are about the test material rather than the server:
`fetches the test material from`, `is given the resources:`,
`should match the pattern in "file"`, `should contain the pattern in "file"`,
`should match one of the patterns in:`.

## How a request is built

The way the HL7 runner (`TxTester`) builds it: the `Parameters` written in the
step, plus one `tx-resource` parameter for every resource the server was given.
Nothing is installed on the server, so a run leaves nothing behind.

The operation verbs do **not** assert the HTTP status. For these tests the
status is an outcome, so a scenario states it: `$response.status should be 200`,
or `should match "^4"` where the server is expected to refuse.

## How a response is judged

The upstream expected-response files are already matchetypes — they use `$id$`,
`$uuid$`, `$instant$`, `$choice:…$` and `$optional-properties$` for what may
legitimately differ between servers and runs. A test fetches the file at run
time and hands it to the validator's matchetype service as the pattern, so the
expectations cannot drift from the source.

**How the comparison is asked for.** The expectation files are stored in the order the
terminology test runner's sorters produce, so a comparison that ignored that would fail on
array order alone. The steps therefore ask the validator's matchetype service to normalise
the response first (`normalize=tx`), which is the same normalisation the runner applies:
server detail such as narrative, meta and diagnostics removed, and parameters, parts,
expansion contents, designations and issues sorted.

Two shapes, matching the service's two modes:

| Step | Mode | For |
|---|---|---|
| `should match the pattern in "file"` | `complete` | the response must match the file, wildcards aside |
| `should contain the pattern in "file"` | `partial` | the file states a minimum; the capability statements |

**Still open** (measured against the validator build of 2026-09-21, and worth re-checking
before relying on a result):

1. `ValueSet.expansion.parameter` is sorted by the normalisation but the stored files are
   not in that order, so an expansion response that is right still fails. Reproduce by
   comparing an expectation file with itself: fill its placeholders, drop the `$optional…`
   markers, and compare with `normalize=tx`. It passes without the input and fails with it.
   Normalising a file that already matches should be a no-op.
2. What a file marks optional still has to be present, in both forms. An array item marked
   `$optional$` fails on the item count when absent, and a property listed in
   `$optional-properties$` fails as "missing element" when absent. Reproduce with an
   expectation file as the actual, with those elements removed.

Measured effect on the `simple-cases` suite against a conformant server: the validator's own
runner passes all of its tests, while the same comparisons through the service pass 16 of 37.
The 21 that fail are every expansion test (defect 1) and both lookups (defect 2). No failure
in that suite is the server's.

## Where the files go

`dialect/` is canonical. `itb-cli/src/sync-dialects.mjs` copies it into every
app that compiles features — the authoring workbench and the ITB manager, each
under `public/components/fhir-terminology/`. Edit it there and the next sync
overwrites you.

Generation 2 of the language only; there is no `steps-v1.yml`.

## Who uses it

The HL7 terminology-ecosystem test set, generated into features and a FHIR
TestPlan by `scripts/tx-tests.mjs` and `scripts/build-test-ig.mjs` in the
authoring workbench: 38 suites, 1182 test cases.
