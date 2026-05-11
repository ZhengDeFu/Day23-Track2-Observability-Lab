# Day 23 Lab Worklog

This file records the concrete work performed so the lab can be resumed later.

## Completed

- Read the main lab instructions, rubric, Makefile, Docker Compose file, and track READMEs.
- Confirmed the target is Core 100 points only, without bonus tasks.
- Ran `make setup`; it created `.env`, then failed because the current user cannot access `/var/run/docker.sock`.
- Verified direct Docker access also fails with `permission denied while trying to connect to the docker API`.
- Tried `sudo docker version`; it requires an interactive password, so this Codex session cannot fix Docker access.
- Ran `python3 00-setup/verify-docker.py` with escalated execution. It wrote `00-setup/setup-report.json`, currently showing Docker/RAM not OK because the daemon is not reachable.
- Ran `make drift` once; it generated `drift-summary.json` but skipped HTML because `evidently` was missing.
- Attempted to install `evidently==0.4.46`; the package index does not contain that version.
- Installed `evidently==0.4.40`, the nearest available 0.4.x release.
- Updated `requirements.txt` from `evidently==0.4.46` to `evidently==0.4.40`.
- Reran `make drift`; it generated both `04-drift-detection/reports/drift-summary.json` and `04-drift-detection/reports/drift-report.html`.
- Ran `make lint-dashboards`; all three Grafana dashboards passed JSON validation.
- Created `submission/screenshots/`.
- Rewrote `submission/REFLECTION.md` with completed drift results, setup status, tail-sampling math, and clear pending screenshot items.
- Ran `make verify`; current result is 3/12 checks passed. Passing checks are setup report existence, drift summary, and non-trivial reflection. Failing checks are all local HTTP service checks because the Docker stack is not running.

## Blocked

- Docker commands are blocked by daemon/socket permissions, so these commands could not be completed in this session:
  - `make up`
  - `make smoke`
  - `make load`
  - `make alert`
  - `make trace`
  - `make verify`
- Slack alert evidence is pending because the stack cannot start until Docker access is fixed.
- The `.env` file still contains the placeholder Slack webhook unless you replace it manually.

## Resume Steps

1. Fix Docker permissions outside this session, for example by ensuring the current Linux user can access the Docker daemon, then start a fresh shell.
2. Put the real Slack webhook into `.env`:

   ```text
   SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
   ```

3. Run:

   ```bash
   make setup
   make up
   make smoke
   make load
   make alert
   make trace
   make verify
   ```

4. Capture the pending screenshots listed in `submission/REFLECTION.md`.
5. Fill in your name and public GitHub URL in `submission/REFLECTION.md`.
