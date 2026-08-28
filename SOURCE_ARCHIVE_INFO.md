# G1 DirectChat source archive

The build workflow consumes the first `*.zip` at the repository root (`source.zip`),
unpacks it over the tree, then runs: npm ci → Jest → RN bundle → gradle test →
assembleDebug + assembleRelease with CI signing.

## Current archive on this branch

Make-before-break transport migration fix + Aurora UI redesign.

SHA-256: `20546db71ce4de1a86d955deff5f475bd98657f36d8503efb94dce8e292fc5cd`

CI-verified green on runs 33139458440 and 33139455934 (2026-08-28).

Previous archive on `main` had SHA-256:
`8f333eda9c47aa76eed5da62fb7d379880bb597d7f54c9b7a2796b2873643dde`
