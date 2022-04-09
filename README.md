# Securing Kubernetes with Open Policy Agent (OPA)

## Rego

A simple Rego rule that checks if the conference name is "BSides":

```rego
package bsides

default allow = false

allow = true {
    input.conference.name = "BSides"
}
```

| Input                                       | Output             |
| ------------------------------------------- | ------------------ |
| `{"conference":{"name": "BSides" }}`        | `{"allow": true}`  |
| `{"conference":{"name": "SomethingElse" }}` | `{"allow": false}` |

A bit more complex rule that checks if the conference name is "BSides" and conference venue is "UNWE" and if not outputs a message:

```rego
package bsides

violations[{"msg": msg}] {
    input.conference.name != "BSides"
    input.conference.venue != "UNWE"
    msg = sprintf("name and venue are wrong - [%s, %s]", [input.conference.name, input.conference.venue])
}
```

However, the message is only shown if both checks are true, e.g. if will not output anything if only one of the values is wrong:

| Input                                                               | Output                                                                                   | Correct |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------- |
| `{"conference":{"name": "BSides", "venue": "UNWE" }}`               | `{"violations": []}`                                                                     | ✅      |
| `{"conference":{"name": "SomethingElse", venue: "SomethingElse" }}` | `{"violations": [{"msg": "name and venue are wrong - [SomethingElse, SomethingElse]"}]}` | ✅      |
| `{"conference":{"name": "SomethingElse", venue: "UNWE" }}`          | `{"violations": []}`                                                                     | ❌      |
| `{"conference":{"name": "BSides", venue: "SomethingElse" }}`        | `{"violations": []}`                                                                     | ❌      |

In this case, this is not what we want, we want to have an error message if at least one of the two values is wrong.

However, Rego rules work by just chaining the expressions in an `AND` statement.

That is why, if we want to achieve this result, we need to tweek the rule a bit:

```rego
package bsides

violations[{"msg": msg}] {
    input.conference.name != "BSides"
    msg := "name is wrong"
}

violations[{"msg": msg}] {
    input.conference.venue != "UNWE"
    msg := "venue is wrong"
}
```

| Input                                                               | Output                                                                  | Correct |
| ------------------------------------------------------------------- | ----------------------------------------------------------------------- | ------- |
| `{"conference":{"name": "BSides", "venue": "UNWE" }}`               | `{"violations": []}`                                                    | ✅      |
| `{"conference":{"name": "SomethingElse", venue: "SomethingElse" }}` | `{"violations": [{"msg": "name is wrong"}, {"msg": "venue is wrong"}]}` | ✅      |
| `{"conference":{"name": "SomethingElse", venue: "UNWE" }}`          | `{"violations": [{"msg": "name is wrong"}]}`                            | ✅      |
| `{"conference":{"name": "BSides", venue: "SomethingElse" }}`        | `{"violations": [{"msg": "venue is wrong"}]}`                           | ✅      |

This covers rules all possibilities and we have at least one error message if some of the values is wrong.

NOTE: This is not a bug or a deficiency in Rego, this is just how Rego rules work and something we should be aware of when using the language.
