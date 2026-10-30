# DoctorinaAI LiteLLM patch and upgrade record

This repository is DoctorinaAI's maintenance fork of [BerriAI/litellm](https://github.com/BerriAI/litellm). The current backport keeps the upstream package version at `1.103.4`; the published company release uses a separate tag for its built artifacts

## Current patch: aiohttp session recovery ownership

| Field | Value |
| --- | --- |
| Upstream base | [`v1.103.4` at `65e36b02981d6ec9335ec45be225f8ad2f0dacb2`](https://github.com/BerriAI/litellm/commit/65e36b02981d6ec9335ec45be225f8ad2f0dacb2) |
| DoctorinaAI patch commit | [`3bda3c260b82a863586c9ace2e1a1680888b19db`](https://github.com/DoctorinaAI/litellm/commit/3bda3c260b82a863586c9ace2e1a1680888b19db) |
| Maintenance branch | `fix_aiohttp_session_recovery_ownership` |
| Push ownership | DoctorinaAI team |
| Default branch | The maintenance branch becomes default when these docs are published |
| Published company release | [`v1.103.4-session-recovery.1`](https://github.com/DoctorinaAI/litellm/releases/tag/v1.103.4-session-recovery.1) |
| Published package index | [`index.html`](https://github.com/DoctorinaAI/litellm/releases/download/v1.103.4-session-recovery.1/index.html) |
| Python package version | `1.103.4` |
| Source file | `litellm/llms/custom_httpx/aiohttp_transport.py` |
| Test file | `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py` |

The patch hunk in `LiteLLMAiohttpTransport.handle_async_request` changes only retry session selection. It replaces the unconditional `self.client = self._rebuild_session()` followed by `client_session = self.client` with `retry_client_session: Final = self._get_valid_client_session()`, then passes that local to the single retry. The helper can reuse a valid session installed by another request and rebuilds when its existing validity checks require it. Capturing and disposing the session that faulted, the one-retry limit, and propagation of unrelated `RuntimeError` values were already present and remain unchanged. This patch changes no dependency constraints or Langfuse settings, and it keeps the aiohttp transport

The patch changes only `litellm/llms/custom_httpx/aiohttp_transport.py` and `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py`

The key regression is `test_session_closed_retries_share_one_replacement`. It forces overlapping requests to encounter the same closed session and checks that both retries use one replacement that remains open until the transport closes it. Related coverage includes concurrent replacement ownership, unrelated runtime errors, a second failed retry, shared-session cleanup, and request or response-stream cancellation

This is conditional code-path evidence. The regression reproduces the ownership failure under forced overlap; it does not attribute a natural production incident or establish how often the condition occurs

From the LiteLLM checkout root, contributors can rerun the mapped transport module with the repository environment:

```bash
uv run pytest tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py -v
```

Sol's final source snapshot receipt is **39 passed, 0 skipped**. The installed macOS arm64 wheel also passed the mapped module with 39 tests. Two original Linux amd64 probes were one-off checks from a clean install with builds disabled; each observed 2 sessions created and 0 left open. Those probes have no checked-in command and are not part of the rerun procedure. The mapped module was not run on Linux. The focused mapped module is the rerunnable repository regression check; this evidence does not claim that the full upstream suite passed

## Upgrading the fork

1. Fetch upstream tags and select the candidate release. Inspect its aiohttp transport and existing mapped transport tests for the same failed-session ownership and shared-replacement behavior.
2. Before applying the source change, run the equivalent regression if the candidate already has it. Otherwise, port `test_session_closed_retries_share_one_replacement` into the candidate's existing mapped transport test module, including its required imports and helper classes `MockAiohttpResponse` and `MockContent`. If the test layout has moved, locate and extend the existing mapped transport module; do not create a new test file just because the regression is absent. In a disposable candidate checkout, run the test with:

   ```bash
   uv run pytest tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py::test_session_closed_retries_share_one_replacement -v
   ```

   Use the candidate's mapped test path if its layout has changed. A collection or setup failure means the regression did not run and shows no behavior gap. Only a collected failure of the ownership behavior demonstrates the gap. A collected pass means the candidate preserves the invariant, so keep the regression and drop the redundant source delta.
3. If the regression demonstrates the gap, cherry-pick patch commit `3bda3c260b82a863586c9ace2e1a1680888b19db` onto a clean checkout of the candidate release, resolving conflicts deliberately. Keep one copy of the regression and rerun the full mapped transport module plus the checks required by that upstream revision.
4. Update this ledger with the new upstream base, patch commit, test command and result, and any remaining delta

Do not weaken the one-retry limit, session ownership checks, or the concurrent replacement regression to make an upgrade apply cleanly

## Published release and artifact verification

The published release contains three artifacts built from source snapshot [`3bda3c260b82a863586c9ace2e1a1680888b19db`](https://github.com/DoctorinaAI/litellm/commit/3bda3c260b82a863586c9ace2e1a1680888b19db): a Linux amd64 manylinux 2.28 wheel, a macOS arm64 wheel, and an sdist. GitHub displays a SHA-256 digest for each release asset. Match each digest to the corresponding `#sha256=` value in the [published package index](https://github.com/DoctorinaAI/litellm/releases/download/v1.103.4-session-recovery.1/index.html)

Build from the repository root with the pinned Maturin `1.15.0`, Rust `1.98.0`, and `litellm-rust/Cargo.lock`. The Linux builder image is `ghcr.io/pyo3/maturin:v1.15.0@sha256:0d1711efd2d3aab276f738bdcda48d378bceef2eeb7f8a8caf59451defddee72`. It advertises manylinux 2.17; `--compatibility manylinux_2_28` sets the requested output wheel tag

Run the Linux build from the repository root inside that pinned builder:

```bash
maturin build --release --locked --jobs 2 --interpreter /opt/python/cp312-cp312/bin/python --compatibility manylinux_2_28
```

Run the native macOS arm64 wheel build and sdist build from the repository root:

```bash
maturin build --release --locked --jobs 2
maturin sdist
```

For each wheel, verify it contains the reviewed aiohttp transport and that the native module imports. Install a supported wheel in a clean environment with `uv pip install --no-build <wheel>`, replacing `<wheel>` with that platform's wheel path. Run the mapped transport module against installed artifacts where supported; do not use the one-off Linux probes as a rerun command or treat a source-checkout test as proof of wheel contents

## Langfuse FileDeleted callback

| Field | Value |
| --- | --- |
| Release target | `v1.103.4-session-recovery.2` |
| Python package version | `1.103.4` |
| Patch set | Aiohttp session recovery above and this callback patch |
| Source file | `litellm/integrations/langfuse/langfuse.py` |
| Test file | `tests/test_litellm/integrations/test_langfuse.py` |

The Langfuse callback now handles the real OpenAI SDK `FileDeleted` response by reading its `id` attribute and passing `{"id": ...}` to the existing timestamped logging ID helper. The regression checks trace and generation linkage, the timestamped returned ID, a metadata override of the sink ID, an unchanged SDK response, and no callback error log

The focused regression constructs the real SDK response and covers both metadata cases. From the repository root, rerun it and the combined callback and transport suite with:

```bash
uv run --frozen --no-sync pytest tests/test_litellm/integrations/test_langfuse.py::test_log_event_on_langfuse_logs_file_deleted -q
uv run --frozen --no-sync pytest tests/test_litellm/integrations/test_langfuse.py tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py -q --tb=short
```

Recorded verification: the focused regression passed 2 cases, and the combined suite passed 156 tests. A Langfuse generation records callback telemetry; it does not prove provider-side deletion or the provider's deletion outcome

Release target `.2` groups both patches at package version `1.103.4`. Keep the published `.1` wheels, sdist, and checksum index unchanged

## Updating the consumer project

From the consumer project's `backend/` directory, point the index named `litellm-session-recovery` in its uv configuration to `https://github.com/DoctorinaAI/litellm/releases/download/v1.103.4-session-recovery.2/index.html`. Regenerate the lock without `--frozen`:

```bash
uv lock
```

Check `uv.lock` to confirm the `litellm` registry URL is the company release `index.html`, the Linux wheel, macOS wheel, and sdist hashes match their `#sha256=` values in that index, and every unrelated dependency version is unchanged. After the lock is verified, reinstall an environment already pinned to package version `1.103.4` from the consumer project's `backend/` directory:

```bash
uv sync --frozen --reinstall-package litellm
```

Consumers configuring uv with `--no-build` need a compatible wheel before using an unsupported platform. Changed artifact bytes require a new versioned release; do not replace published assets in place
