# Dependency Management Rules

## Strict Version Pinning

- ALL npm packages MUST use exact versions (no `^`, `~`, or `*` ranges)
- ALL Python packages MUST use `==` pinning (no `>=`, `~=`, or unpinned)
- The root `.npmrc` enforces `save-exact=true` — never override this
- Never run `npm update` or `pip install --upgrade` unless the user explicitly updates package.json or requirements.txt

## Install Commands

- Use `npm ci` (not `npm install`) for reproducible installs from lockfile
- Use `pip install -r requirements.txt --no-deps` for Python installs
- Always commit `package-lock.json` files to version control

## Axios Safety

- Pin axios to exactly `1.14.0`
- NEVER install axios `1.14.1` or `0.30.4` — these versions were compromised with a RAT (March 2026 supply chain attack)
- If axios needs updating, verify the new version on the official GitHub releases page before changing

## Adding New Dependencies

- When adding a new dependency, always specify the exact version number
- Verify the package name carefully to avoid typosquatting
- Prefer well-known, actively maintained packages with large download counts
- Do not add dependencies without explicit user approval
