---
description: "Generate YAML test files from project structure. Use when user says 'generate test', 'test olustur', 'yaml olustur', 'create test file', 'test yaz', 'generate yaml'."
---

# Nazar YAML Test Generator

Auto-generate YAML UI test files from your project structure.

## How to run

1. Scan project structure:
   - React Native: look in `src/screens/`, `src/pages/`, `app/` for screen components
   - Flutter: look in `lib/screens/`, `lib/pages/` for widget files
   - Expo: check `app/` directory (file-based routing) or `src/screens/`
2. For each screen found:
   - Detect navigation elements (buttons, links, tab items)
   - Detect form inputs (TextInput, TextField, input elements)
   - Detect interactive elements (touchable, gesture detector, button)
3. Generate test scenarios:
   - Navigation flow: launch -> navigate to each screen -> verify visible
   - Form flow: navigate to form -> fill inputs -> submit -> verify result
   - Auth flow: if login screen found, generate login test
4. Write YAML files to `.nazar/tests/` directory:
   - One file per screen or flow
   - apiVersion: nazar/v1 format
5. Report: N test files generated, M screens covered, K interactions mapped
6. Suggest: "Run /nazar:live-test to execute these tests"
