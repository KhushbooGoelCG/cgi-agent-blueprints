# GitHub Copilot Instructions

## Instruction Files

This repository uses scoped instruction files located in `.github/instructions/`.
Always check and apply the relevant instruction file(s) from that folder based on
the files you are working on.

| File | Scope |
|------|-------|
| `api.instructions.md` | Banking.Api and Banking.Application (`src/Banking.Api/**`, `src/Banking.Application/**`) |
| `sql.instructions.md` | Banking.SqlDb sqlproj (`src/Banking.SqlDb/**`) |
| `react.instructions.md` | banking-app react vite app (`src/banking-app/**`) |
| `tests.instructions.md` | Banking.Api.Tests and Banking.Application.Tests (`tests/Banking.Api.Tests/**`, `tests/Banking.Application.Tests/**`) |



When generating or editing code:
- Always load the instruction file that matches the target file path.
- If multiple instruction files match, apply all of them.
- Project-scoped instructions take precedence over language-scoped instructions
  when there is a conflict.

## General Rules (apply to all files)

- Target framework: .NET 10
- Language: C# 14
- Nullable reference types are enabled — always respect nullability annotations.
- Follow existing conventions in the file being edited before applying defaults.
- Do not introduce new NuGet packages without noting it explicitly in your response.
- Do not leave TODO comments unless asked.
