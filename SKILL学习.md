## What should SKILL be included?
1. Focus on what the agent wouldn’t know without your skill: project-specific conventions, domain-specific procedures, non-obvious edge cases, and the particular tools or APIs to use.
2. Ask yourself about each piece of content: “Would the agent get this wrong without this instruction?” If the answer is no, cut it.
3. [https://agentskills.io/skill-creation/best-practices](https://agentskills.io/skill-creation/best-practices)
4. You can use link style to add reference, so Claude knows when to load it.
5. You can get argument by patten `$ARGUMENTS[0]`, also `$N` is the same thing.


## Meta parameters:
[Claude Code Reference] https://code.claude.com/docs/en/skills#frontmatter-reference
1. If you want to prevent claude code run SKILL automatically, add `disable-model-invocation: true` to meta block. Or else, if you only want LLM to invoke it, add `user-invocable: false`
2. The `context:fork` means the SKILL will create a new isolated context, then run it.

## Inject dynamic context
The ``!`command` `` syntax is used to run shell command before sending SKILL content to LLM.The output will replaces the placeholder in the skill content. Just link following coding
```
## Pull request context
- PR diff: !`gh pr diff`
```

## Writing

Complete a real task in conversation with an agent, and then extract the reusable pattern into a skill. Watch up the following content
1. Steps that worked
2. Corrections you made
3. Input/output formats
4. Context you provided
Feeding a body of existing knowledge into an LLM and ask it to synthesize a skill.
#### Tips
- Don’t over invest before the first round of results. You can expand it later.
- Vary the prompts
- Cover edge cases.
- Use realistic context.
- Add what the agent lacks, omit what it knows.
- Spec recommends Keeping SKILL.md under 500 lines and 5,000 tokens. It should only cover the core instructions the agent needs on every run. Move detail to `references/` or similar directories.
- The doc says “Give the agent some space to thinking or freedom” But are you sure?
- Provide defaults, not multi options or menus.

## Description

### How to write description
- **Use imperative phrasing** Frame 
	- Using instruction: just like “Use this skill when…” rather than “This skill does…”
- Focus on intent Not implementation.
- Keep it concise. The spec enforce hard limit 1024 characters. a few sentences to a short paragraph is usually right.
### How to test description
- Aim for about 20 queries: 8~10 that should trigger and 8~10 that shouldn’t.
- Run multi-times because of the model behavior is nondeterministic. You should calculate a trigger rate.
### How to avoid overfitting
- Split two query sets, and mix a proportional of should-trigger and should-not-trigger.
	- Train set (~60%): the queries you use to identify failures and guide improvements.
	- Validation set(~40%): queries you set aside and only use to check whether improvements generalize.

## Progressive disclosure

1. Metadata(~100tokens)
2. Instructions(< 5000 tokens recommended)
3. Resources(as needed)

A Skill should teach agent how to complete a class of tasks, not the specific problem.

```
<!-- Specific answer — only useful for this exact task -->
Join the `orders` table to `customers` on `customer_id`, filter where
`region = 'EMEA'`, and sum the `amount` column.
```
  
```
<!-- Reusable method — works for any analytical query -->
1. Read the schema from `references/schema.yaml` to find relevant tables
2. Join tables using the `_id` foreign key convention
3. Apply any filters from the user's request as WHERE clauses
4. Aggregate numeric columns as needed and format as a markdown table
```
  

### Templates for output format
Templates are more reliable than describe the format in prose/paragraph.

  

### Gotchas
Don’t write a lot of environment-specific content in the SKILL, they aren’t general advices. The agent may not recognize the tigger.
These a list of gotchas is not valuable.

  

### Checklist
A explicit checklist helps the agent track progress avoid skipping steps

### Plan-validate-execute
When You try to create a batch or destructive operations, adding a validation in the list, valid it against a source of truth, only then execute the operation

Code Review Skill Summary
1. Write the task list for this skill
write code style

## Reference
[AgentSkill](https://agentskills.io)
[code review skill](https://github.com/obra/superpowers/blob/main/skills/requesting-code-review/code-reviewer.md)