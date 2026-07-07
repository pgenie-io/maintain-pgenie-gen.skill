# maintain-pgenie-gen

An agent skill that creates or updates a [pGenie](https://github.com/pgenie-io/pgenie) generator from a reference design artifact.

## What this does for you

You don't need to learn Dhall or Haskell to implement a new pGenie generator. You just need an LLM and this skill.

1. Design your wish-to-be output artifact: a hand-written project showing exactly what the generator should produce for the input of the [`demo` repo](https://github.com/pgenie-io/demo) (`./queries`, `./migrations`, `./project1.pgn.yaml`). Browse the [`./artifacts`](https://github.com/pgenie-io/demo/tree/master/artifacts) directory in that repo for examples of what the generator should produce for other languages.
2. Invoke this skill, pointing it at your design artifact.
3. Answer the questions the agent asks.
4. You get a pGenie generator that is likely ready to use.

If it's not quite right, polish it by telling your LLM what you need changed **in the output artifact** - the skill updates the design first, then re-converges the generator. The design artifact stays the living spec; the acceptance test is a diff.

## Install

Claude Code:

```bash
git clone https://github.com/pgenie-io/update-gen.skill ~/.claude/skills/maintain-pgenie-gen
```

Then ask your agent to implement (or update) a pGenie generator, giving it your design artifact and a path for the generator repo.

## How it works

See [SKILL.md](SKILL.md). In short: the agent studies the [generator architecture](https://github.com/pgenie-io/gen-sdk/blob/master/docs/generator-architecture.md) and the existing generators ([java.gen](https://github.com/pgenie-io/java.gen), [rust.gen](https://github.com/pgenie-io/rust.gen)), validates that your design artifact builds and passes its tests, interviews you about everything the artifact doesn't pin down (type mappings, naming, how patterns extrapolate beyond the demo), then plans and executes the implementation as isolated tasks in fresh agent sessions, converging until the generator's output for the demo project becomes the same as in your design.

## Related repos

- [gen-sdk](https://github.com/pgenie-io/gen-sdk) - the SDK and the authoritative architecture doc (fetched live by this skill)
- [demo](https://github.com/pgenie-io/demo) - the canonical input project
- [java.gen-design](https://github.com/pgenie-io/java.gen-design), [rust.gen-design](https://github.com/pgenie-io/rust.gen-design), [c-sharp.gen-design](https://github.com/pgenie-io/c-sharp.gen-design), [ts.gen-design](https://github.com/pgenie-io/ts.gen-design) - design artifacts
