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

One actor kind, `terminology-server`, and fourteen verbs:

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

and four that are about the test material rather than the server:
`fetches the test material from`, `is given the resources:`,
`should match the pattern in "file"`, `should match one of the patterns in:`.

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

**Known gap.** The validator's `/itb/matchetype` endpoint (`MatchetypeValidator`)
compares arrays in order and does not honour `$optional-properties$` or an
array item's `$optional$`. The HL7 runner compares with `CompareUtilities`
after sorting the actual response (`TxTesterSorters`) and stripping
`issue.diagnostics` (`TxTesterScrubbers`). Until the endpoint does the same, a
server that is right but answers in another order fails here and passes there.
That belongs in the validator, not in this dialect.

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
