---
mode: agent
---
Define the task to achieve, including specific requirements, constraints, and success criteria.

## Common Development Rules - ALPS TTP3
## Core Rules (Must Not Be Changed)

Do not write comments in code

### Do not generate .md files unless explicitly requested

Do not run any git commands (commit, merge, checkout, reset, etc.) or anything that modifies git history


Do not use icons in chat or code. This includes symbols like checkmarks, crosses, targets, rockets, etc.

## Personality & Behavior

Explain using existing logic from similar functions

Do not create new files or new logic unless there is a clear request






Do not write comments in code


## Code Formatting Rules

- No trailing whitespace on any line
- No blank lines with only spaces or tabs
- Maximum one blank line between functions
- Use consistent indentation (4 spaces for Python)
- Remove all linting errors before committing (use `black` or `autopep8`)
- Run `flake8` to check for PEP 8 violations before final code

# note for create md summary ticket
khi tạo link hãy tạo theo format 
ví dụ: [tên hàm hoặc tác dụng của dòng này](file:///home/thang/Documents/rsu/alps-ttp3-frontend/src/components/Device/Detail/BillCycleTab/components/AssignContractButton.tsx#L248)

1.At the very top of the file, include the ticket ID and the file name.

2.Provide an overview of the feature flow, following the same pattern as the file:///home/thang/Documents/rsu/copilot-rules/TTPNEO-906.md

Use a diagram for better readability. The diagram should be organized by module, file, and function, and clearly describe the purpose, what needs to be done, and any existing issues.
Do not include or quote detailed code in this diagram.

3.Create an implementation plan for each part, divided into clear and concise phases. Each phase should state what needs to be done.
Quote the file that needs to be edited according to the pattern I showed above, Do not include or quote any code.

4.The total length of the file must be less than 200 lines.

5. Do not arbitrarily create a ticket name. You can leave it blank, I will add it myself



