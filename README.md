# Securing Kubernetes with Open Policy Agent (OPA)

Build-in Kubernetes security is not enough for most organizations to enforce granular rules and policies to the workloads running in their clusters.

That is why, Kubernetes has pluggable mechanism for deploying additional validation for your resources.
These are the [admission controllers/admission webhooks](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) which allow you to deploy a webhook which will called by Kubernetes to check whether a given resource can be created/updated.

You can write your own admission webhook from scratch or use [Gatekeeper](https://github.com/open-policy-agent/gatekeeper) which allows you to deploy custom policies, written in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/) and evaluated by [Open Policy Agent](https://www.openpolicyagent.org/).

## Open Policy Agent

[Open Policy Agent](https://www.openpolicyagent.org/) is a general-purpose policy agent that evaluates JSON input against rules, written in Rego, and returns a JSON output based on the evaluation.

## Rego

Rego is a declarative query language that can be used to write policies about the data coming into the system.

A simple Rego rule that checks if the conference name of the input document is "BSides":

```rego
package bsides

default allow = false

allow = true {
    input.conference.name = "BSides"
}
```

If you want to play with the examples in the Rego playground, use the links in the `Link` column.

| Input                                      | Output             | Playground link                                       |
| ------------------------------------------ | ------------------ | ----------------------------------------------------- |
| `{"conference":{"name": "BSides"}}`        | `{"allow": true}`  | [Link](https://play.openpolicyagent.org/p/OqvsvG5BU7) |
| `{"conference":{"name": "SomethingElse"}}` | `{"allow": false}` | [Link](https://play.openpolicyagent.org/p/SzGf67ckQg) |

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

| Input                                                                | Output                                                                                   | Correct | Playground link                                       |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------- | ----------------------------------------------------- |
| `{"conference":{"name": "BSides", "venue": "UNWE"}}`                 | `{"violations": []}`                                                                     | ✅      | [Link](https://play.openpolicyagent.org/p/wY3h6EN6Dj) |
| `{"conference":{"name": "SomethingElse", "venue": "SomethingElse"}}` | `{"violations": [{"msg": "name and venue are wrong - [SomethingElse, SomethingElse]"}]}` | ✅      | [Link](https://play.openpolicyagent.org/p/JeV5Yg2mlw) |
| `{"conference":{"name": "SomethingElse", "venue": "UNWE"}}`          | `{"violations": []}`                                                                     | ❌      | [Link](https://play.openpolicyagent.org/p/ZPS6QPZZD7) |
| `{"conference":{"name": "BSides", "venue": "SomethingElse"}}`        | `{"violations": []}`                                                                     | ❌      | [Link](https://play.openpolicyagent.org/p/Xo4jlfyFii) |

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

| Input                                                              | Output                                                                  | Correct | Playground link                                       |
| ------------------------------------------------------------------ | ----------------------------------------------------------------------- | ------- | ----------------------------------------------------- |
| `{"conference":{"name": "BSides", "venue": "UNWE"}}`               | `{"violations": []}`                                                    | ✅      | [Link](https://play.openpolicyagent.org/p/WMxPOAnMqR) |
| `{"conference":{"name": "SomethingElse", venue: "SomethingElse"}}` | `{"violations": [{"msg": "name is wrong"}, {"msg": "venue is wrong"}]}` | ✅      | [Link](https://play.openpolicyagent.org/p/bOYYXRjkhY) |
| `{"conference":{"name": "SomethingElse", venue: "UNWE"}}`          | `{"violations": [{"msg": "name is wrong"}]}`                            | ✅      | [Link](https://play.openpolicyagent.org/p/iVHjvE6C4c) |
| `{"conference":{"name": "BSides", venue: "SomethingElse"}}`        | `{"violations": [{"msg": "venue is wrong"}]}`                           | ✅      | [Link](https://play.openpolicyagent.org/p/UB8NwmJ5e4) |

This covers rules all possibilities and we have at least one error message if some of the values is wrong.

**NOTE:** This is not a bug or a deficiency in Rego, this is just how Rego rules work and something we should be aware of when using the language.
