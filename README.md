# Codex macOS: Connection refused (os error 61)

Seeing this error?

```text
stream disconnected before completion: Connection refused (os error 61)
```

One possible cause of this Codex connection failure on macOS is a stale local proxy setting pointing to a service that is no longer running. This guide walks through symptoms, diagnosis, a targeted fix, verification, and rollback for that specific cause.

**Independent community guide. Not affiliated with or endorsed by OpenAI. This is not a universal fix for every stream disconnection.** There is no installer or automatic repair script in this repository.

## Contents

- [Symptoms and scope](#symptoms-and-scope)
- [Before changing anything](#before-changing-anything)
- [Diagnosis](#diagnosis)
- [Fix](#fix)
- [Verification](#verification)
- [Rollback](#rollback)
- [Documented case and evidence limits](#documented-case-and-evidence-limits)
- [Sources and further help](#sources-and-further-help)

## Symptoms and scope

Use this guide when Codex shows `Connection refused (os error 61)`, including the full stream-disconnection message above, and your configuration may contain an obsolete loopback proxy such as `http://127.0.0.1:3213`.

The error text alone does not identify the cause. First establish whether a stale proxy is configured and whether its local service is listening. If the diagnosis does not match, stop before editing settings.

## Before changing anything

- Save your work and let active tasks finish before quitting the app.
- If your workplace, school, or network requires a proxy, ask its administrator before removing settings. Restoring the intended proxy service may be the right solution.
- Keep all checks local. Do not post your full environment, shell configuration, process output, logs, or screenshots containing secrets.
- The example port below is **3213**. Use the actual port from your own configuration when checking your machine.
- These steps do not require deleting app data, resetting your account, reinstalling macOS, or disabling security tools.

## Diagnosis

### 1. Find candidate proxy settings

This command prints only the names of existing files containing proxy-variable names. It does not print their contents:

```sh
for file in "$HOME/.zshrc" "$HOME/.zprofile" "$HOME/.zshenv" \
            "$HOME/.profile" "$HOME/.bash_profile" "$HOME/.bashrc" \
            "${CODEX_HOME:-$HOME/.codex}/.env"; do
  if [ -f "$file" ]; then
    grep -El '(^|[^[:alnum:]_])(HTTP_PROXY|HTTPS_PROXY|ALL_PROXY|NO_PROXY|http_proxy|https_proxy|all_proxy|no_proxy)([^[:alnum:]_]|$)' "$file"
  fi
done
```

Open any matching file locally in an editor and inspect the relevant lines. Matches can include comments, so a match alone does not prove a setting is active. This is a list of common locations, not an exhaustive search of every startup mechanism.

In the diagnosed case, the relevant lines were equivalent to:

```sh
export HTTP_PROXY="http://127.0.0.1:3213"
export HTTPS_PROXY="http://127.0.0.1:3213"
export http_proxy="$HTTP_PROXY"
export https_proxy="$HTTPS_PROXY"
```

`127.0.0.1` means your own Mac. If the local proxy is not listening at that port, a client trying to use it can receive a connection-refused error.

### 2. Check the local listener and system settings

For the example port:

```sh
lsof -nP -iTCP:3213 -sTCP:LISTEN
```

A result indicates a visible listening process. No output means this command found none; an error or permissions limitation is inconclusive. A listener's presence alone does not prove that it is a working proxy.

You can inspect macOS system proxy settings locally with:

```sh
scutil --proxy
```

System proxy settings and environment variables are different configuration sources. Empty system proxy settings do not rule out a proxy in an app's environment. Likewise, an empty `launchctl getenv HTTPS_PROXY` result does not prove the already-running app has no proxy variables.

If you found no relevant stale proxy, stop here. The error may have a different cause; do not modify unrelated configuration just to match this example.

## Fix

### 1. Back up and edit only confirmed stale settings

If the obsolete settings are in `~/.zshrc`, make a backup first:

```sh
backup="$HOME/.zshrc.backup-$(date +%Y%m%d-%H%M%S)"
cp -p -n "$HOME/.zshrc" "$backup"
printf 'Backup path: %s\n' "$backup"
```

Confirm that the copy succeeded before editing. Open the original file in a local editor:

```sh
nano "$HOME/.zshrc"
```

Add `#` to the beginning of **only the four confirmed stale HTTP/HTTPS export lines**. Preserve `PATH`, `NO_PROXY`, unrelated settings, and other proxy settings unless you have separately established that they are obsolete. Save the file.

If your settings are in a different file, back up and edit that specific file instead. Avoid automated bulk replacement across your home directory. Do not source the entire shell configuration merely to test this change; it may run unrelated commands.

Editing a file does not remove variables already inherited by running shells or applications.

### 2. Fully quit and launch with a clean proxy environment

Quit the desktop app using its Quit menu or `Command-Q`, rather than just closing its window. Use Activity Monitor to check that the main app has exited. Do not interrupt unrelated processes.

The working installation in this case used `/Applications/ChatGPT.app`. Your app name or installation location may differ. Confirm the actual bundle in Finder and adjust the `APP` value below. The command reads the bundle's executable name rather than guessing it:

```sh
APP="/Applications/ChatGPT.app"
EXECUTABLE=$(/usr/libexec/PlistBuddy -c 'Print :CFBundleExecutable' "$APP/Contents/Info.plist")
if [ -n "$EXECUTABLE" ] && [ -x "$APP/Contents/MacOS/$EXECUTABLE" ]; then
  env -u HTTP_PROXY -u HTTPS_PROXY -u ALL_PROXY \
      -u http_proxy -u https_proxy -u all_proxy \
      "$APP/Contents/MacOS/$EXECUTABLE"
else
  printf '%s\n' 'App executable not found. Check the APP path before continuing.'
fi
```

This removes six proxy variables only for this launch and its inherited environment. It leaves `NO_PROXY` and `no_proxy` intact. It does not remove proxy settings loaded later from a file, change system proxy settings, or permanently modify your shell.

**Use this test only if bypassing these proxy overrides is appropriate for your network.** Keep the launching terminal open during the test. Any console output is for local inspection only and may contain private information.

## Verification

1. Start a new, small Codex task, such as asking it to reply with `OK` without changing files.
2. Confirm that it actually completes a response. An open app window or connected-device indicator alone is insufficient.
3. When convenient, fully quit and reopen normally, then repeat the test. This checks whether the fix survives your usual launch path.

If a clean launch works but a normal launch fails, another startup source may still be supplying proxy settings. Revisit configuration sources rather than repeatedly reinstalling the app.

If it still fails, record only a sanitized summary: app and macOS versions, exact error text, whether a stale loopback proxy was found, whether its port was listening, and the outcome of the clean-launch test. Consult the official troubleshooting page below. The same error can have causes outside this guide's scope.

## Rollback

Compare your backup with the current configuration locally and restore **only the lines you changed**, or remove the comment markers you added. Do not blindly overwrite the current file with an old backup if you have made other changes since. Fully quit and reopen the affected app after restoring the intended configuration.

## Documented case and evidence limits

Four global exports in `~/.zshrc` set `HTTP_PROXY`, `HTTPS_PROXY`, `http_proxy`, and `https_proxy` to `http://127.0.0.1:3213`. The stale proxy was also detected in the running desktop app's environment, but there was no listener on port 3213. macOS system proxy settings did not show a configured proxy.

The first clean-environment launch alone did not resolve the problem. The user was instructed to back up the shell configuration and reported saving the edit that commented out just those four exports. Backup completion was not independently verified. After a full restart with the proxy variables removed from the launch environment, the user reported that it worked.

This supports stale proxy configuration as the cause in this case. We did not establish which program originally added the exports, or precisely how the app reacquired them during the earlier attempt. No particular VPN, proxy client, or other tool is blamed.

The edit and restart were tested together. A completed Codex response and persistence after a normal launch were not independently recorded. See [the evidence notes](docs/evidence.md) for the full limits of the report.

## Sources and further help

- [Related macOS report in openai/codex issue 38885](https://github.com/openai/codex/issues/38885): a separate user report describes the same error with a stale proxy in `$CODEX_HOME/.env`. Its configuration source differs from the `~/.zshrc` case documented here. The issue is supporting community evidence, not official confirmation that all occurrences have this cause.
- [Official ChatGPT desktop troubleshooting](https://learn.chatgpt.com/docs/reference/troubleshooting): general troubleshooting and reporting guidance; it also advises reviewing logs for sensitive information before sharing them. It does not prescribe this specific shell-configuration fix.

See [the evidence notes](docs/evidence.md) for the limits of this case report and [contribution guidance](CONTRIBUTING.md) before sharing a reproduction.
