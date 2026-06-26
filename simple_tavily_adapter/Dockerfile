# Pinned to the s6-overlay-based release we verified works (2026-06-26).
# To upgrade:
#   1. docker pull nousresearch/hermes-agent:latest
#   2. Verify patch anchors in run_agent.py still match (grep _disable_streaming)
#   3. Verify privilege-drop tool: grep for s6-setuidgid in /command/ (was gosu pre-s6)
#      and check entrypoint-adapter.sh uses the right binary.
#   4. Update the digest below and rebuild.
FROM nousresearch/hermes-agent@sha256:daebdb9d03203caddc2ca7bc1d9f6727bcab5ad21c41a6135b4be15b91de9a85

USER root

RUN apt-get update \
 && apt-get install -y --no-install-recommends curl \
 && rm -rf /var/lib/apt/lists/*

RUN uv pip install --no-cache-dir --python /opt/hermes/.venv/bin/python3 \
    "trafilatura>=1.12,<3.0" \
    "lxml>=4.9" \
    "beautifulsoup4==4.12.2" \
    "sse-starlette==2.1.3"

# Workaround for vLLM streaming tool_call bug (vllm#27641, fix in PR #35449 — not
# yet merged as of v0.14.1). When stream=true, large prompts route tool-call JSON
# into reasoning_content instead of the structured tool_calls array, so Hermes
# thinks the model is narrating instead of calling tools. Non-streaming works
# fine.
#
# Patch: make streaming opt-out via HERMES_DISABLE_STREAMING=1 (env var) rather
# than flipping the hardcoded default — so non-vLLM providers (Anthropic,
# OpenRouter, etc.) are unaffected unless the var is set.
# Set HERMES_DISABLE_STREAMING=1 in your compose/env when pointing at vLLM.
RUN python3 - <<'PY'
import re
path = '/opt/hermes/run_agent.py'
src = open(path, encoding='utf-8').read()
old = 'prefer_stream=not bool(getattr(self, "_disable_streaming", False))'
new = (
    'prefer_stream=not bool(getattr(self, "_disable_streaming", False)'
    ' or __import__("os").environ.get("HERMES_DISABLE_STREAMING") == "1")'
    '  # patched: vLLM streaming tool_call bug (vllm#27641)'
)
assert old in src, 'streaming toggle anchor not found — upstream changed; update the patch'
open(path, 'w', encoding='utf-8').write(src.replace(old, new))
print('patched: streaming gated on HERMES_DISABLE_STREAMING env var')
PY

RUN mkdir -p /app && chown hermes:hermes /app
COPY --chown=hermes:hermes . /app/

COPY --chown=root:root docker/entrypoint-adapter.sh /entrypoint-adapter.sh
RUN chmod +x /entrypoint-adapter.sh

WORKDIR /app
EXPOSE 8000

ENTRYPOINT ["/entrypoint-adapter.sh"]
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
