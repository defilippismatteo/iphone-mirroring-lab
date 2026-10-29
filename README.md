# Laboratorio remoto per Duplica iPhone

Esperimenti software eseguiti su VM macOS Intel temporanee di GitHub Actions. **Non è disponibile un bypass funzionante di Duplica iPhone.**

## Risultati verificati — 29 ottobre 2026

Ambiente delle due esecuzioni: macOS 15.7.9, build 24G830, architettura x86_64. Questa immagine non è la stessa build del Mac fisico dell'utente.

| Prova | Esito osservato |
| --- | --- |
| Creazione chiave P-256 software temporanea | Riuscita |
| Firma e verifica del messaggio sintetico | Riuscite |
| Verifica di un messaggio alterato | Correttamente rifiutata |
| Disponibilità Secure Enclave secondo CryptoKit | false |
| Generazione tramite token Secure Enclave | Fallita, OSStatus -4 |
| Importazione software ordinaria della chiave sintetica | Riuscita, stessa chiave pubblica e firma valida |
| Richiesta del token Secure Enclave durante l'importazione | Fallita, dominio CryptoTokenKit, codice -1 |
| Dati vuoti senza token | Falliti, NSOSStatusErrorDomain -50 |
| Dati vuoti con token Secure Enclave richiesto | Falliti, CryptoTokenKit -1 |

- [Esecuzione 1: chiavi, firme e analisi statica dell'app](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40880051033)
- [Esecuzione 2: confronto dell'attributo token](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40881046993)

Nell'ambiente provato, richiedere quell'attributo non ha trasformato la chiave software in una chiave utilizzabile tramite Secure Enclave. Il successo dei workflow indica che i controlli sperimentali sono stati eseguiti, non che sia stato aggirato il requisito hardware.

## Interpretazione del secondo test

Nel [sorgente pubblico Apple di SecKeyCreateWithData](https://github.com/apple-oss-distributions/Security/blob/main/OSX/sec/Security/SecKey.m), la presenza dell'attributo token instrada la chiamata al backend CryptoTokenKit senza passargli il parametro keyData. I risultati sono coerenti con due percorsi API differenti. Non provano che il Secure Enclave abbia esaminato e rifiutato crittograficamente il contenuto della chiave.

Il sorgente pubblico è una guida interpretativa: non dimostra da solo la corrispondenza esatta con tutti i dettagli del binario della VM.

## Esecuzione e isolamento

Da Actions, scegliere uno dei due workflow e premere Run workflow. L'esecuzione è manuale, sul runner standard macos-15-intel, con un limite di 10 minuti. Ogni esecuzione usa una nuova VM. I workflow non richiedono permessi di scrittura al repository.

I programmi sperimentali sono compilati ed eseguiti solo nella VM. Non usano file del Mac locale, Apple Account, iPhone o chiavi preesistenti dell'utente. Non modificano SIP, NVRAM, firmware o applicazioni di sistema.

Il primo test non serializza chiavi private. Il secondo serializza esclusivamente una nuova chiave sintetica nella memoria della VM per confrontare le importazioni; non ne stampa o salva i byte. Nei log sono pubblicati solo esiti, token e codici di errore.

## Limiti

La VM remota non permette di verificare il collegamento Bluetooth/Wi-Fi peer-to-peer con l'iPhone locale, né l'accettazione dell'attestazione da parte del telefono. Queste prove non certificano il funzionamento sul Mac fisico e non offrono una garanzia al 100%.
