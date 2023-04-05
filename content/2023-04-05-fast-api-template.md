+++
title = "Python FastAPI template"
weight = 1
order = 1
date = 2023-04-05
insert_anchor_links = "right"
[taxonomies]
tags = ["python", "api", "setup", "tldr"]
+++

The repository can be found there: [github.com/aurelien-clu/template-python-fast-api](https://github.com/aurelien-clu/template-python-fast-api)

## Target

Implement quickly a Python API with an [hexagonal architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)) with the following already setup:

- [fastapi](https://fastapi.tiangolo.com/)
- [dependency injection](https://github.com/ets-labs/python-dependency-injector)
- [pydantic serialization & deserialization](https://docs.pydantic.dev/)
- [pydantic settings](https://docs.pydantic.dev/usage/settings/)
- [Behavior Driven Development](https://en.wikipedia.org/wiki/Behavior-driven_development) [tests](https://github.com/behave/behave)

## Layout

```
.
├── Makefile
├── pyproject.toml
├── src
│   ├── api                 <= API routes
│   │   └── health.py
│   │
│   ├── config              <= defaults + values from environment
│   │   └── server.py
│   │
│   ├── infra/              <= repositories, caches, etc.
│   │
│   ├── svc                 <= logic exposed by API routes
│   │  └── health.py
│   │
│   ├── app.py
│   ├── container.py        <= "registry" of dependencies to inject
│   ├── errors.py
│   └── main.py
│
└── tests
    └── bdd
        ├── health.feature  <= Gherkin tests
        └── steps/          <= Gherkin steps implementations
```

## How to reuse

```bash
git clone https://github.com/aurelien-clu/template-python-fast-api <your-project>
```

## Efficient test writing

Using [Behavior Driven Development](https://en.wikipedia.org/wiki/Behavior-driven_development), it is easy to reuse parts of tests, alike querying an API endpoint, validating the response, etc.

Once a set of steps has been written (python code below) you are able to write many tests quickly using [Gherkin](https://cucumber.io/docs/gherkin/reference/) language which ressemble natural language with the formalism of `Given`, `When`, `Then`.

```ini
// Gherkin test definition
Feature: Health
  Background:
    Given an API client
  Scenario: Health check: GET
    Given path: /
    When getting
    Then response code is 200
    And json response is "ok"
```
[github.com/aurelien-clu/template-python-fast-api/tests/bdd/health.feature](https://github.com/aurelien-clu/template-python-fast-api/blob/main/tests/bdd/health.feature)


```python
# reusable steps across tests (repository hold few more)
@step("path: {path}")
def step_impl(context, path: str):
    context.request_path = path

@step("getting")
def step_impl(context):
    path = context.request_path
    headers = getattr(context, "request_headers", None)
    context.response = context.client.get(path, headers=headers)

@step("response code is {code}")
def step_impl(context, code: str):
    check = context.response.status_code == int(code)
    assert check

@step('json response is "{text}"')
def step_impl(context, text: str):
    check = context.response.json() == text
    assert check
```
[github.com/aurelien-clu/template-python-fast-api/tests/bdd/steps](https://github.com/aurelien-clu/template-python-fast-api/tree/main/tests/bdd/steps)
