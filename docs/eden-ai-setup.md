# Eden AI Setup

[Eden AI](https://www.edenai.co/) is the managed gateway for Open WebUI chat, RAG embeddings, and VS Code custom models. Built-in Copilot and Brave Search remain separate.

## Endpoint and billing

- **Base URL:** `https://api.edenai.run/v3`
- **Chat completions:** `https://api.edenai.run/v3/chat/completions`
- Use separate `openwebui` and `vscode` keys with **monthly spending limits**. Enable account auto-refill; it is not a spending cap.
- Store the Open WebUI key in GitHub secret `EDEN_AI_API_KEY`; enter the VS Code key through its credential prompt. Never store literal keys in configuration files.
- Monitor usage in the [Eden dashboard](https://app.edenai.run/). Credit purchases carry a **5.5% fee**, plus applicable taxes; model prices can change.
- The global endpoint does **not** guarantee EU residency. Check the selected model's hosting region in Eden's catalog.

## Open WebUI

[Chat values](../applications/chat/values.yaml) and the [deployment workflow](../.github/workflows/deploy-apps.yml) configure one Eden connection. Chat and embeddings both use the `provider-api-keys` Secret, key `keys`.

| Setting | Value |
| --- | --- |
| Chat API base URL | `https://api.edenai.run/v3` |
| `RAG_EMBEDDING_ENGINE` | `openai` |
| `RAG_EMBEDDING_MODEL` | Provider-qualified embedding ID from [Chat values](../applications/chat/values.yaml) |
| `RAG_OPENAI_API_BASE_URL` | `https://api.edenai.run/v3` |
| `RAG_OPENAI_API_KEY` | Secret reference: `provider-api-keys/keys` |

**Admin settings persist in the database and can override Helm defaults.** Update connections and document/embedding settings in the admin UI when changing credentials or routing.

Keep the underlying embedding model and dimensions compatible with the existing index. Changing only the gateway does not require re-indexing when both remain unchanged. Back up persistent data before changing either.

Brave Search uses `BRAVE_SEARCH_API_KEY` and `brave_llm_context`; native function calling remains enabled.

## VS Code

On macOS, configure `~/Library/Application Support/Code/User/chatLanguageModels.json`:

- Keep the built-in **Copilot** provider and one **Eden AI** provider (`vendor: customendpoint`, `apiType: chat-completions`).
- Use the chat-completions URL above and VS Code's `${input:chat.lm.secret.…}` credential placeholder.
- Set context/output limits, vision, tool calling, and reasoning options according to the selected provider's current capabilities. Reserve output space within the context window.
- Utility settings (`chat.utilityModel` and `chat.utilitySmallModel`) use `customendpoint/<model id>`.
- VS Code remembers model selections separately from provider configuration. Extensions can also register providers; disable or uninstall unwanted provider extensions.

Check current capabilities and prices in Eden's public [chat catalog](https://api.edenai.run/v3/models) and [embedding catalog](https://api.edenai.run/v3/embeddings/models). After configuration changes, test chat, tool calling, vision, web search, and retrieval using non-sensitive inputs.
