# SmartMail Extension

## Vision
SmartMail reformats Zimbra (webmail.polytechnique.fr) drafts with a local LLaMA model, keeping the user in the compose window while producing polished academic and professional emails.

## Architecture Overview
- **Manifest V3 core** (`manifest.json`) with service worker `background.js`, targeted content scripts, keyboard commands, and host permissions limited to Zimbra.
- **Service worker orchestration** controls warmup states, prompt assembly, streaming inference, error recovery, and logging.
- **Content layer** detects TinyMCE editor activity, injects UI elements, handles shortcuts, and synchronizes state with the service worker.
- **Prompt builder module** encapsulates the tag-oriented template for context, comments, and raw body content.
- **Local model client** talks to a llama.cpp HTTP server (quantized model) for warmup pings and generation with abort support.
- **Options page** surfaces configuration (server port, prompt overrides, warmup timers, model profile) stored in `chrome.storage`.

## Architecture Diagram
```mermaid
graph TD
    U[Zimbra Composer (TinyMCE)] --> CS[Content Scripts<br/>editorWatcher.js & modal.js]
    CS -->|Ctrl+Shift+W| UI[Modal UI<br/>Shadow DOM]
    UI -->|Prompt data| SW[Service Worker<br/>background.js]
    SW --> PB[Prompt Builder<br/>promptBuilder.js]
    PB --> SW
    SW --> MC[Model Client<br/>modelClient.js]
    MC -->|HTTP warmup/generate| LL[Local LLaMA Server]
    SW -->|State broadcast| CS
    UI -->|Insert formatted text| U
    SW --> OPT[Options Page<br/>options/]
    OPT -->|chrome.storage sync| SW
    OPT -->|chrome.storage sync| CS
```

## Key Modules
- `src/background/modelClient.js` - HTTP client for warmup, generate, and cancel requests.
- `src/background/stateMachine.js` - Maintains the idle -> warming -> ready -> generating lifecycle and broadcasts status.
- `src/content/editorWatcher.js` - Observes DOM focus and keystrokes in the compose view and notifies the service worker when the user is typing.
- `src/content/ui/modal.js` - Shadow DOM modal with context/comments inputs, generated text area, copy/insert buttons, and status feedback.
- `src/prompt/promptBuilder.js` - Produces structured prompts with `<RECIPIENTS>`, `<SUBJECT>`, `<CONTEXT>`, `<COMMENTS>`, `<BODY_RAW>`, `<INSTRUCTIONS>` tags.
- `src/options/` - Options UI (vanilla or lightweight framework) plus validation.
- `src/shared/logger.js` - Lightweight leveled logger shared across scripts.

## Prompt Template (Draft)
```text
<ROLE>Vous etes un assistant qui aide un etudiant de l'Ecole Polytechnique a rediger des mails academiques et professionnels.</ROLE>
<RECIPIENTS>{{recipients}}</RECIPIENTS>
<SUBJECT>{{subject}}</SUBJECT>
<CONTEXT>{{context|optional}}</CONTEXT>
<COMMENTS>{{comments|optional}}</COMMENTS>
<BODY_RAW>{{mail_body}}</BODY_RAW>
<INSTRUCTIONS>Reecrire le mail avec un ton professionnel clair, concis, poli et adapte a l'enseignement superieur francais. Conserver les informations essentielles et suggerer des ameliorations de structure si necessaire.</INSTRUCTIONS>
```

## Implementation Plan
1. Scaffold project structure, draft the manifest, define keyboard commands, permissions, and build tooling (esbuild or Vite).
2. Implement content scripts for editor detection, command handling, and UI injection (modal, buttons, status badges).
3. Build the service worker with warmup and generation handlers, state machine, and integration with the local llama server.
4. Develop the prompt builder and shared utilities; cover them with lightweight unit tests.
5. Finish the modal UX, wiring content scripts and the service worker, and support insertion into the TinyMCE body.
6. Create the options page for configuring the server address, template overrides, timeouts, and auto-warmup toggles.
7. Perform end-to-end manual QA on Zimbra, tune warmup timing, and document the workflow in this README.
