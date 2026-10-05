# Evidence and limitations

This guide is based on a single macOS troubleshooting session plus the separately linked public issue. It is a practical case report, not a controlled test of every app version or network setup.

## Observed in the documented case

- Four global HTTP/HTTPS proxy exports in `~/.zshrc` pointed to the loopback endpoint `http://127.0.0.1:3213`.
- A check detected that endpoint in the running desktop app's environment without publishing the environment contents.
- The local listener check found no listening service at that port.
- The macOS system proxy check showed no configured proxy.
- An initial clean-environment launch did not resolve the problem.
- The user was instructed to back up the configuration and reported saving the edit that commented out only the four stale exports. Backup completion was not independently verified.
- After another full clean-environment launch, the user reported success.

## What remains uncertain

The original installer of the exports and the exact mechanism that reintroduced the proxy during the earlier launch were not established. Shell startup reintroduction is a possible explanation, not a proven implementation detail. The successful recovery combined an edit and a restart; their individual effects were not isolated.

The reported success did not include a recorded normal-launch persistence test or an independently captured completed Codex response. Readers should perform both verification steps in the README. Do not use this report to claim compatibility with untested macOS or app versions.

No raw logs, transcripts, account identifiers, personal device names, credentials, or private process output are included. Commands in the README are manual examples, not an automatic repair tool.
