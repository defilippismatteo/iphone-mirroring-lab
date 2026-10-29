# Laboratorio remoto per Duplica iPhone

Esperimenti su VM macOS Intel temporanee di GitHub Actions. **Non è stato ottenuto un bypass funzionante di Duplica iPhone.** Il successo di un workflow indica che la prova ha terminato e registrato il risultato, anche quando una funzionalità risulta non disponibile.

## Vincoli e ambiente

Le prove seguenti sono avvenute esclusivamente nelle VM GitHub, senza Apple Account, abbinamento iPhone, dati personali o accesso al Portachiavi del Mac fisico. Il repository pubblico contiene codice e risultati sintetici. Nessuna patch è stata applicata al Mac. Le VM usavano macOS 15.7.9, build 24G830, x86_64: non è la stessa build del Mac da rendere compatibile.

## Risultati verificati — 29 ottobre 2026

| Prova | Risultato osservato | Limite |
| --- | --- | --- |
| P-256 software effimera | Creazione, firma e verifica riuscite; messaggio alterato correttamente rifiutato | Non è un’identità hardware attestata |
| Generazione Secure Enclave | CryptoKit.SecureEnclave.isAvailable=false; generazione fallita, OSStatus -4 | La VM non offre quel backend |
| Importazione software ordinaria | Riuscita; stessa chiave pubblica e firma valida | Controllo positivo |
| Importazione con token Secure Enclave richiesto | CryptoTokenKit -1, anche con blob vuoto | Il sorgente Apple instrada al token; non prova che quei byte siano stati sottoposti a verifica crittografica |
| Record in Portachiavi file-based dedicato | Rename del label riuscito; file modificato, chiave pubblica e firma conservate | Solo archivio sintetico della VM |
| Update token su quel record | SecItemUpdate=0, ma token effettivo sempre none; confermato riaprendo la chiave da un nuovo processo | Nessuna conversione software→Secure Enclave |
| Data Protection nativo | Chiave effimera valida; inserimento persistente bloccato da -34018, missing signing entitlement | Update token non eseguito; non è un esito del bypass T2 |
| Analisi statica sharingd | Nomi di record/candidati, gestione LTK e chiamate AKS individuati | Non prova un abbinamento Mirroring accettato |

Il Portachiavi file-based dedicato è stato eliminato nella VM e la lista di ricerca ripristinata. La prova DP non ha interrogato record preesistenti. Non è stato esportato materiale privato dai record persistenti. Il primo esperimento di importazione ha serializzato esclusivamente una chiave sintetica in RAM, senza stamparne o salvarne i byte.

## Esecuzioni riproducibili

- [Generazione e firma software / Secure Enclave](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40880051033).
- [Instradamento importazione verso token](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40881046993).
- [Static map iniziale](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40889426883): il job statico è riuscito; il distinto job record aveva un errore di compilazione poi corretto.
- [Record file-based corretto e ricaricamento da nuovo processo](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40890655736).
- [Data Protection e riferimenti AKS](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40892774420).
- [Risoluzione CFString e confini delle funzioni](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40894749823).
- [Costruttori dei record e selettori risolti](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40895339247).
- [Mappa finale della cartella e persistenza](https://github.com/defilippismatteo/iphone-mirroring-lab/actions/runs/40895952293).

## Dove sono salvati i dati: cosa sappiamo davvero

Apple documenta queste implementazioni generali, non gli specifici record di Duplica iPhone:

| Archivio | Percorso generale |
| --- | --- |
| Portachiavi login file-based | `~/Library/Keychains/login.keychain-db` |
| Portachiavi System file-based | `/Library/Keychains/System.keychain` |
| Data Protection dell’utente | `~/Library/Keychains/<UUID>/keychain-2.db` |

Il percorso e lo schema DP sono dettagli interni, soggetti a cambiamento. Fonti: [Apple TN3137](https://developer.apple.com/documentation/technotes/tn3137-on-mac-keychains), [Apple DTS: SecItem Fundamentals](https://developer.apple.com/forums/thread/724023), [costruzione del percorso nel sorgente Security](https://github.com/apple-oss-distributions/Security/blob/main/OSX/utilities/SecFileLocations.c).

### Record Auto Unlock identificati staticamente

Nel binario `/usr/libexec/sharingd` della VM, SHA-256 `ce1cd6c397a2253d8deb1b029442aeafe1e199d7228b24bd26e0f22a7917b570`, abbiamo ricostruito la catena `ltkInfoForDeviceID:modern:attested:` → `attestedDictionaryForDevice:` → `baseDictionary` → `keychainDataForQuery:` → `SecItemCopyMatching`. Si tratta della costruzione di una query reale nel codice Apple, non soltanto della presenza di stringhe. Le funzioni sono state lette staticamente, senza invocarle.

| Campo della query attestata | Valore ricostruito |
| --- | --- |
| `kSecClass` | `kSecClassGenericPassword` |
| `kSecAttrAccessGroup` | `com.apple.continuity.unlock` |
| `kSecAttrAccount` | `com.apple.continuity.auto-unlock.attested` |
| `kSecAttrService` | Argomento device passato a `ltkInfoForDeviceID:modern:attested:` |
| `kSecAttrIsInvisible` | true; riguarda la visibilità |
| `kSecReturnData` | true, aggiunto da `keychainDataForQuery:` |

`baseLocalKeysDictionary` usa lo stesso costruttore generale e imposta account `com.apple.continuity.auto-unlock.localkeys`. `baseLocalKeysDictionaryForDevice:type:` aggiunge service=device e, se fornito, `kSecAttrKeyType`. La classe generale resta generic password: la ricerca dei soli elementi di classe key non copre questi record.

`storeKeychainItemWithAttributeDictionary:updateDictionary:addDictionary:` contiene le chiamate effettive a `SecItemAdd` e `SecItemUpdate`, con un percorso di recupero delete/add. È stata analizzata, mai eseguita. `useKeychainForLTKSyncing` restituisce true in questa build. La presenza dei costruttori dei file sottostanti non dimostra quindi che quei file siano il deposito attivo di ogni LTK.

**Limiti della mappa:** non abbiamo interrogato i record reali, individuato il database fisico che contiene quel particolare record, decifrato il valore, o dimostrato quali record siano esclusivi di Mirroring. Il gruppo Apple non concede accesso a un programma soltanto perché ne si copia il nome. I risultati delle prove generiche SecKey non sono una prova di sostituzione di questi record.

### Percorsi dei file Auto Unlock

Il costruttore statico chiama `NSSearchPathForDirectoriesInDomains(5, 1, true)`, poi aggiunge `Sharing` e `AutoUnlock`. L’interpretazione delle costanti Foundation e della [Library dell’utente documentata da Apple](https://developer.apple.com/documentation/foundation/url/librarydirectory) porta normalmente a:

- `~/Library/Sharing/AutoUnlock/ltk.plist`, tramite `remoteLTKStorageFilePath`.
- `~/Library/Sharing/AutoUnlock/pairing-records.plist`, tramite **`watchIDsToPairingRecordsStorageFilePath`**, quindi con un’associazione esplicita agli Apple Watch.

Sono percorsi ricostruiti dal codice della VM, non file letti sul Mac dell’utente. Non li presentiamo come deposito completo delle chiavi private o delle identità Mirroring, né abbiamo eseguito i costruttori che possono creare le cartelle. Non è stato applicato alcun reset o cancellazione di questi dati.

### Attestazione e impostazioni di test

Le chiamate `_aks_create_signing_key`, `_aks_create_signing_key_with_params`, `_aks_remote_session` e `_aks_remote_session_step` sono confermate nel disassemblato. La presenza di classi `SDAuthenticationLTKManager`, `SDAutoUnlockAKSManager` e di funzioni di generazione/verifica LTK attestate rende insufficiente estendere automaticamente i risultati delle API SecKey generiche a tutto il protocollo nativo.

`forcelocalAttestedLTKRegeneration` è effettivamente letto con `standardUserDefaults`/`boolForKey:` in `generateLocalAttestedLTKIfNeccessaryWithCompletion:`. Il ramo richiama la generazione della LTK attestata: non è un override hardware dimostrato. Non abbiamo impostato questa preferenza. Anche i messaggi “TESTING ONLY” individuati non dimostrano un’opzione che faccia accettare una chiave software dall’iPhone.

Il parser usa CFString con lunghezza verificata, le catene di fixup x86_64 riconosciute e `LC_FUNCTION_STARTS`; il run finale non ha segnalato formati fixup non supportati o pagine multi saltate. Quattro CFString restano non risolte; l’analisi è mirata, non una dimostrazione esaustiva dell’assenza di altri percorsi. I log comprendono i contesti annotati e 21 funzioni; una grande funzione Swift è troncata nel corpo mostrato, con conteggio e riferimenti espliciti.

## Perché scrivere il flag non basta

L’header Apple [SecItem.h](https://github.com/apple-oss-distributions/Security/blob/main/keychain/headers/SecItem.h) descrive il token come immutabile dopo la creazione e le chiavi Secure Enclave come generate nel relativo hardware. Nel sorgente [SecItem.m](https://github.com/apple-oss-distributions/Security/blob/main/OSX/sec/Security/SecItem.m) il record ricostruisce un oggetto del provider tramite token e object ID; non contiene semplicemente un booleano che trasforma una chiave software. [SecItemSchema.c](https://github.com/apple-oss-distributions/Security/blob/main/keychain/securityd/SecItemSchema.c) include dati cifrati e attributi autenticati; il riferimento token è autenticato. Queste fonti descrivono l’infrastruttura Security, senza dimostrare il formato privato esatto di una chiave Mirroring.

L’errore DP -34018 riguarda autorizzazioni di firma. Una firma ad hoc con identificatore scelto non equivale a un access group autorizzato da provisioning profile: [TN3137](https://developer.apple.com/documentation/technotes/tn3137-on-mac-keychains), [TN3125](https://developer.apple.com/documentation/technotes/tn3125-inside-code-signing-provisioning-profiles). Non sono stati aggiunti entitlement Apple, profili inventati o permessi estesi per aggirarlo.

Apple [documenta un fallback nel kernel per lo sblocco generico sui Mac senza Secure Enclave](https://support.apple.com/en-by/guide/security/sec6ab47ebfc/1/web/1), ma descrive anche l’applicazione della policy Mirroring nel Secure Enclave. Il fallback generico è una pista di analisi, non una dimostrazione che Duplica iPhone lo accetti.

## Verifica ancora mancante

Mancano un’identità alternativa accettata dal protocollo nativo e una sessione Mirroring riuscita su hardware privo di T2. Le VM GitHub non offrono l’iPhone locale, la vicinanza radio e un ambiente Apple autorizzato per verificare quel passaggio. I risultati disponibili non consentono di fornire una patch garantita o di dichiarare che modificare un database risolva il requisito hardware.
