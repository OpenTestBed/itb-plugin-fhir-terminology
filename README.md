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

**Where this stands** (measured 2026-09-21 against FHIRsmith 0.13.2 at `/tx/r5`, with the
validator build of that afternoon):

| | |
|---|---|
| The server, judged by the validator's own `txTests` runner | 3 failures in 664 general-mode tests |
| The same tests through this dialect and the matchetype service | 524 of 671 general-mode tests pass |

So the server is in good shape and the remaining ~144 differences are in the comparison
path, not in it. They cluster: `OperationOutcome: unexpected element issue` (103), missing
elements inside `parameter.resource.issue` (121), and `array item count differs at
Parameters.parameter` (106). Two candidates worth separating before blaming the comparer:
this dialect fetches the test material from the IG's `main` branch while the runner used
the published package `1.9.3`, and the runner may send requests these tests do not
reproduce exactly.

**Use an endpoint whose version matches the files.** The expectation files are R5-shaped.
Against an R4 endpoint they demand R5-only elements (`expansion.property`, for one) that no
R4 server can return: `simple-cases` scored 26 of 37 against `/tx/r4` and 37 of 37 against
`/tx/r5`, same server, same build. The generator therefore targets an R5 endpoint and
selects the R5 test set by default.

**Mode-gated suites are not failures.** `sct-ecl`, `tx.fhir.org`, `snomed`, `icd-11`,
`omop` and `mimetypes` apply only to a server that carries that content, and they are 325 of
the 1178 tests. Judge a server on the general-mode suites unless it claims the rest.

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
