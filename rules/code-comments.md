---
paths:
  - "**/*.{go,ts,tsx,js,jsx,nix,py,rs,sh}"
---
 
# Code comments
 
Write no comment unless the code cannot carry the information.
 
Allowed:
 
- **Why**, when the reason is not visible in the code: a workaround for an upstream bug, named; a deliberate deviation from the obvious approach; a constraint found the hard way.
- **A hazard** the next editor would otherwise trip over, such as an ordering requirement or a lock held across a call.
- **Doc comments on exported identifiers**, where the language convention expects them, such as godoc or TSDoc. One sentence saying what the caller gets. No parameter-by-parameter restatement of the signature.
Never write:
 
- A comment restating the line below it.
- Change narration: `// now uses X instead of Y`, `// fixed the race here`. This goes in the commit message, and the diff already shows it.
- Section banners such as `// --- helpers ---`.
- Dates or your own name.
One line where one line does. If the explanation needs a paragraph, rename or restructure the code instead.
 
When editing existing code, leave existing comments alone unless the change makes them wrong, in which case fix or delete them.
 
```go
// Bad
// increment the counter
count++
 
// Good
// Retry before the 30s timeout in vendor/client.go.
count++
```