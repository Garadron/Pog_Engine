# Ricerca: alternative a Whisper per la trascrizione (step 2)

Data della ricerca: 2026-09-30. Fatta per capire se conviene cambiare motore di trascrizione e come ridurre le ripetizioni di frasi che Whisper genera nello step 2.

## Situazione attuale

- Motore: whisper.cpp (`models\Release\whisper-cli.exe`, build cublas) con il modello `ggml-large-v3.bin`.
- Lingua: prima fissata a inglese (`-l en`), ora configurabile con `TRANSCRIPTION_LANGUAGE` in `pipeline_config.py` (nel fork vale `it`).
- La pipeline lancia `whisper-cli.exe` una finestra da 30 minuti alla volta e si aspetta un `.srt` in uscita.
- Contro le ripetizioni ci sono già: rilevamento dei cicli lunghi con ritrascrizione automatica (step 2), fusione delle frasi duplicate adiacenti (step 3), soglie nel comando (`--entropy-thold 2.6`, `--logprob-thold -0.8`, `--no-speech-thold 0.7`).
- Il comando usa `-mc -1` (contesto del testo già scritto attivo).

## Perché Whisper ripete le frasi

Whisper scrive una parola alla volta, scegliendo la più probabile dopo quelle già scritte. Nei tratti con solo rumore, musica o mormorio nessuna parola è davvero probabile, e la scelta "sicura" diventa ripetere l'ultima frase, che poi si autoalimenta. Il testo già trascritto passato come contesto può trascinare la ripetizione nel pezzo successivo.

## Confronto Parakeet-TDT-0.6B-v3 (NVIDIA) contro Whisper large-v3

| | Parakeet v3 | Whisper large-v3 |
|---|---|---|
| Italiano, errori (scheda ufficiale Parakeet) | 3,00% FLEURS, 10,08% MLS, 3,69% CoVoST | 2,31% FLEURS (fonte secondaria, non verificato) |
| Media su 5 lingue europee (Open ASR Leaderboard) | 4,81% | 4,81% (fonte secondaria, non verificato) |
| Velocità | circa 3.300 volte il tempo reale | molto più lento |
| Ripetizioni | il decodificatore può emettere "nessun testo" sul silenzio, quindi niente cicli | cicli noti su silenzio e rumore |
| Lingue | 25 europee, italiano compreso | 99 |
| Windows | documentazione ufficiale: Linux; installazione via NeMo più delicata | già integrato e funzionante |
| Licenza | CC-BY-4.0 | MIT |
| Durata per inferenza | fino a circa 24 min con attenzione completa (A100 80GB), fino a 3 ore con attenzione locale | finestre da 30 min già gestite dalla pipeline |

Sull'accuratezza in italiano è sostanzialmente un pareggio. I vantaggi di Parakeet sono velocità e assenza di cicli.

## Altri modelli emersi

- IBM Granite Speech 4.1 (5,33% errori), Cohere Transcribe 03-2026 (5,42%), NVIDIA Canary-Qwen-2.5B (5,63%) in cima alla classifica Open ASR. La classifica è soprattutto inglese: supporto all'italiano non verificato.
- Voxtral (Mistral): 7,05% (Mini 3B) e 6,62% (Small 24B) sulla stessa classifica. Non meglio di Parakeet.
- Whisper large-v3-turbo: stesso Whisper con decodificatore più leggero, circa 6-8 volte più veloce, qualità simile. Esiste in formato whisper.cpp, quindi sarebbe un cambio di file del modello.
- faster-whisper / WhisperX: stessi modelli Whisper con motore più veloce e strumenti anti-allucinazione migliori; richiedono di sostituire `whisper-cli.exe` con codice Python.
- Google Speech-to-Text / Chirp: cloud a pagamento, non open source. Esclusa perché contraddice il "100% offline".

## Cosa non è stato verificato

- I numeri di Whisper large-v3 per l'italiano (2,31% e la media 4,81%) vengono da riassunti secondari, non dalla scheda del modello (che non li riporta).
- Il paper di Parakeet/Canary rimanda i dati per lingua in un'appendice che non sono riuscito a leggere; i valori italiani di Parakeet sono quelli della scheda Hugging Face.
- Se Parakeet produce timestamp per frase nel formato che serve alla pipeline.
- Come farlo girare su Windows (NeMo, NeMo-Speech.cpp, porting ONNX): solo ipotesi.
- Tutti i benchmark sono su parlato pulito. Il caso reale (voce separata con Demucs da gioco e musica, VOD di ore) può dare risultati diversi.

## Piano proposto

1. Non cambiare motore alla cieca.
2. Prova A/B sulla stessa finestra da 30 minuti già estratta (una delle 5 in `step1_extract_mic_audio\1_mic_transcription_chunks`): Whisper large-v3 attuale, stesso modello con `-mc 0`, e large-v3-turbo. Confrontare cicli, velocità e qualità sull'italiano reale. Serve la GPU libera.
3. Valutare Parakeet solo se i cicli restano un problema: richiede un adattatore che produca lo stesso SRT e far funzionare NeMo o una versione ONNX su Windows.

## Fonti

- [Canary-1B-v2 & Parakeet-TDT-0.6B-v3 (arXiv)](https://arxiv.org/pdf/2509.14128)
- [Scheda ufficiale nvidia/parakeet-tdt-0.6b-v3](https://huggingface.co/nvidia/parakeet-tdt-0.6b-v3)
- [Scheda openai/whisper-large-v3](https://huggingface.co/openai/whisper-large-v3)
- [Open ASR Leaderboard (arXiv)](https://arxiv.org/pdf/2510.06961)
- [Parakeet vs Whisper (localaimaster)](https://localaimaster.com/blog/parakeet-vs-whisper)
- [Whisper Keeps Repeating Itself (localaimaster)](https://localaimaster.com/blog/whisper-hallucination-fix)
- [Best open source speech-to-text 2026 (Northflank)](https://northflank.com/blog/best-open-source-speech-to-text-stt-model-in-2026-benchmarks)
