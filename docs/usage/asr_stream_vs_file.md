# ASR vLLM : comparaison du mode **stream** (WebSocket) vs **POST fichier audio** (HTTP)

Ce document trace le chemin d’exécution côté serveur, depuis la réception réseau jusqu’aux classes modèle (arrêt à `VoxtralForConditionalGeneration` / `VoxtralRealtimeGeneration`).

## 1) Point d’entrée API

### A. Mode stream (WebSocket `/v1/realtime`)
- Endpoint : `vllm/entrypoints/openai/realtime/api_router.py`.
- Le routeur crée un `RealtimeConnection` qui gère le protocole événementiel (`session.update`, `input_audio_buffer.append`, `input_audio_buffer.commit`).
- Le service métier est `OpenAIServingRealtime`.

### B. Mode fichier (HTTP `POST /v1/audio/transcriptions`)
- Endpoint : `vllm/entrypoints/openai/speech_to_text/api_router.py`.
- FastAPI parse un formulaire multipart (`TranscriptionRequest`), lit `request.file.read()` puis délègue à `OpenAIServingTranscription.create_transcription(...)`.
- Réponse possible:
  - JSON final (non streaming)
  - SSE (`text/event-stream`) si `stream=true`.

---

## 2) Pipeline stream WebSocket (audio chunké en continu)

1. **Handshake + session**
   - `RealtimeConnection.handle_connection()` accepte le WebSocket et envoie `session.created`.
2. **Réception audio**
   - `input_audio_buffer.append` transporte un chunk base64 PCM16 16kHz mono.
   - Le serveur décode et convertit en `np.float32`, puis empile dans `audio_queue`.
3. **Commit de génération**
   - `input_audio_buffer.commit` déclenche `start_generation()` (ou finalise le flux avec `final=true`).
4. **Pont vers le moteur vLLM**
   - `OpenAIServingRealtime.transcribe_realtime(...)` transforme le flux audio en `StreamingInput`.
   - Cette transformation appelle `model_cls.buffer_realtime_audio(...)`.
5. **Boucle de génération**
   - `RealtimeConnection._run_generation(...)` appelle `engine_client.generate(prompt=<async generator>)`.
   - En retour:
     - émission d’événements `transcription.delta`
     - puis `transcription.done` avec `usage`.

### Où le modèle intervient (Voxtral realtime)
- Avec une architecture realtime Voxtral, la méthode clé est:
  - `VoxtralRealtimeGeneration.buffer_realtime_audio(...)` dans `vllm/model_executor/models/voxtral_realtime.py`.
- Cette méthode:
  - instancie un buffer temps-réel,
  - applique padding audio gauche/droite,
  - mélange audio entrant et tokens déjà générés (`input_stream`),
  - produit des `PromptType` incrémentaux.

---

## 3) Pipeline HTTP fichier (offline / batch par requête)

1. **Upload multipart**
   - `create_transcriptions(...)` lit tous les octets audio du fichier.
2. **Prétraitement ASR**
   - `OpenAISpeechToText._preprocess_speech_to_text(...)`:
     - validation (langue, taille max),
     - décodage + resampling via `librosa.load(..., sr=...)`,
     - découpage en chunks si nécessaire (`split_audio`) selon `SpeechToTextConfig`.
3. **Construction des prompts**
   - Pour chaque chunk audio, appel à `model_cls.get_generation_prompt(...)`.
   - Puis rendu moteur via `renderer.render_cmpl_async(...)`.
4. **Génération moteur**
   - `engine_client.generate(...)` est lancé pour chaque prompt/chunk.
5. **Assemblage réponse**
   - Soit réponse finale JSON,
   - soit flux SSE avec `transcription.chunk` (+ usage optionnel).

### Où le modèle intervient (Voxtral offline)
- Dans `vllm/model_executor/models/voxtral.py`:
  - `VoxtralForConditionalGeneration.get_speech_to_text_config(...)`
  - `VoxtralForConditionalGeneration.get_generation_prompt(...)`
  - `VoxtralForConditionalGeneration.get_num_audio_tokens(...)`
- En offline, Voxtral encode la requête transcription via son tokenizer/audio-encoder et retourne un `TokensPrompt` multimodal (`multi_modal_data["audio"]`).

---

## 4) Différences clés stream vs fichier

- **Transport**
  - Stream: WebSocket, événements applicatifs (`append`/`commit`).
  - Fichier: HTTP multipart unique.

- **Granularité de traitement**
  - Stream: incrémental, piloté par chunks audio successifs.
  - Fichier: audio complet reçu avant prétraitement.

- **Méthode modèle appelée**
  - Stream: `buffer_realtime_audio(...)` (interface `SupportsRealtime`).
  - Fichier: `get_generation_prompt(...)` (interface transcription classique).

- **Sortie**
  - Stream: événements WS (`transcription.delta`, `transcription.done`).
  - Fichier: JSON final ou SSE (`transcription.chunk`).

- **Contexte autoregressif en stream**
  - Stream maintient une boucle audio↔tokens avec `input_stream` pour enchaîner les segments.
  - Fichier traite les chunks de façon plus “offline” dans la requête.

---

## 5) Cartographie rapide des fichiers

- Entrée WebSocket: `vllm/entrypoints/openai/realtime/api_router.py`
- Gestion connexion stream: `vllm/entrypoints/openai/realtime/connection.py`
- Service stream -> moteur: `vllm/entrypoints/openai/realtime/serving.py`
- Entrée HTTP ASR: `vllm/entrypoints/openai/speech_to_text/api_router.py`
- Service ASR (prétraitement + génération):
  `vllm/entrypoints/openai/speech_to_text/speech_to_text.py`
- Adaptateur transcription/translation:
  `vllm/entrypoints/openai/speech_to_text/serving.py`
- Modèle Voxtral offline: `vllm/model_executor/models/voxtral.py`
- Modèle Voxtral realtime: `vllm/model_executor/models/voxtral_realtime.py`
