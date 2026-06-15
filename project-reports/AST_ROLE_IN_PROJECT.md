# 🌳 Role of AST in This Project

**Yes — AST actively inspects/scans uploaded Python code files.** Here's exactly how:

---

## 🔍 What is AST here?

Python's built-in `ast` module is used in `backend/mvp_engine.py`. It converts raw Python **source code text → a tree of structured nodes** that the engine can programmatically inspect — without ever executing the code.

```python
import ast
tree = ast.parse(source_code)  # source_code is a string of the uploaded .py file
```

---

## ⚙️ What exactly does it scan/inspect?

| **What it finds** | **How** | **Used for** |
|---|---|---|
| **Functions** | `ast.FunctionDef` nodes | Extract function names, params, return types, line numbers |
| **Classes** | `ast.ClassDef` nodes | Extract class names and their methods |
| **Imports** | `ast.Import` / `ast.ImportFrom` | Find dependencies for mocking in generated tests |
| **Docstrings** | `ast.get_docstring(node)` | Use as test/function descriptions |
| **Argument counts** | `ast.arguments` fields | Decide what kind of test to generate (smoke, edge, etc.) |
| **Return expressions** | `ast.Return` statements | Detect what a function returns logically |
| **Math operations** | `ast.BinOp`, `ast.Mod`, `ast.Compare` | Detect even/odd logic patterns to catch bugs |
| **Literal values** | `ast.literal_eval()` | Safely parse test input strings into Python values |

---

## 🔄 The Pipeline — Step by Step

```
User uploads .py file
        ↓
  ast.parse(source_code)
        ↓
   tree (AST object)
        ↓
  ast.walk(tree) — visits every node
        ↓
  ┌──────────────────────────────────┐
  │ Found FunctionDef?               │
  │  → extract name, params, types   │
  │ Found ClassDef?                  │
  │  → extract class + method names  │
  │ Found Import?                    │
  │  → record dependency             │
  │ Found Return with BinOp(Mod)?    │
  │  → check even/odd correctness    │
  └──────────────────────────────────┘
        ↓
  Auto-generate unit tests + edge tests
  based on discovered structure
```

---

## 🧠 The Smart Part — Logic Bug Detection via AST

This is the most impressive use. In `modulo_return_details()` and `infer_even_odd_intent_cases()`, AST is used to **read the actual math in the return statement** of a function:

```python
# If a function is named "is_even" but its return says:
return n % 2 == 1  # ← bug! should be == 0

# AST catches this:
# ast.BinOp(left=Name('n'), op=Mod(), right=Constant(2))  → divisor=2
# ast.Compare(ops=[Eq()], comparators=[Constant(1)])       → remainder=1
# → "name says even, but logic says odd" → BUG flagged!
```

---

## 🔑 Key Takeaway

> **AST never runs the uploaded code.** It only reads it structurally — like an X-ray of the code. This makes it **safe, fast, and deterministic** for generating tests and detecting logic bugs before execution.
