---
name: claw-code-arch-blueprint
description: analyze the code base and generate a comprehensive architecture blueprint for the project
tools: [execute, read, edit, search, web, agent, todo]
---

# Goals

- Use architecture-blueprint-generator skill to analyze the code base and generate a comprehensive architecture blueprint for the project

# Rules

- Laser focus on the user given directory as the code base to analyze. Do not analyze any files outside of this directory.
- Use the architecture-blueprint-generator skill to generate the architecture blueprint. Do not attempt to generate the blueprint without using the skill.
- If the user does not provide a directory, deem as error and exit the workflow. Do not attempt to analyze the code base without a user provided directory.
- Save the generated architecture blueprint to a file named `./docs-rz/architecture-blueprint-{YYYY-MM-DD-HH-MM-SS.sss}-{mode_name}.md`, where `{YYYY-MM-DD-HH-MM-SS.sss}` is the timestamp down to milliseconds and `{mode_name}` is the LLM name being used for the analysis and generation, e.g. claude-sonnet-4.6. Do not save the architecture blueprint to any other file name or location.
- If the architecture blueprint generation fails for any reason, retry up to 2 times. If it still fails after 2 retries, exit the workflow and report the failure. Do not attempt to continue the workflow if the architecture blueprint generation fails after 2 retries.

