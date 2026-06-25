---
title: "AD CS AutoEnrollment e CryptoAPI: caso reale di blocco causato da middleware crittografico"
date: 2026-06-25T14:00:00+02:00
draft: false
description: "Un client Windows 11 in dominio non completava il rinnovo del certificato macchina per EAP-TLS. La CA funzionava. Il template era corretto. Il blocco era locale, silenzioso e causato da un middleware crittografico che intercettava CryptoAPI prima che certreq potesse completare la richiesta."
translationKey: "ad-cs-autoenrollment-bit4id"
tags:
  - PKI
  - AD CS
  - AutoEnrollment
  - Windows
  - CryptoAPI
  - EAP-TLS
  - 802.1X
  - Security
  - Troubleshooting
  - Enterprise Architecture
slug: "ad-cs-autoenrollment-bit4id"
cover:
  image: "img/ad-cs-autoenrollment-bit4id.png"
  alt: "ad-cs-autoenrollment-bit4id"
  relative: false
---

## Il contesto

Un client Windows 11, aggiornato e in dominio, non riusciva a completare il rinnovo di un certificato macchina per l'autenticazione EAP-TLS su rete 802.1X. Il certificato era emesso su template AD CS dedicato, con provider Microsoft RSA SChannel, EKU Client Authentication, store target `LocalMachine\My`.

La CA emetteva senza problemi. Il template era configurato correttamente. I permessi di enroll/autoenroll per i computer erano presenti. Ma il client non aveva il certificato.

Questo è il tipo di problema che in un ambiente enterprise con centinaia o migliaia di client può essere lungo da isolare, soprattutto quando la pressione operativa porta a intervenire in modo aggressivo prima di aver capito dove sta il blocco.

## Il problema

Il sintomo era apparentemente semplice: `certreq -enroll -machine -q "Machine_AutoEnroll"` non completava. Nessun output utile, nessun errore esplicito, nessun certificato nello store.

Prima di guardare il client, sono stati verificati tutti i componenti a monte: raggiungibilità della CA, FQDN corretto, pubblicazione del template, permessi di enroll/autoenroll, EKU, SAN, renewal period, chain, revocation, store `LocalMachine\My` e `Request`, accessibilità del provider Microsoft RSA SChannel, servizi crittografici, esecuzione nel contesto `NT AUTHORITY\SYSTEM`. Niente da segnalare su nessuno di questi punti.

Il blocco quindi era locale. Per isolarlo, è stato usato un approccio più granulare: `certreq -new` con un INF minimale, esplicitando il provider, catturando stdout, stderr e il process tree tramite un wrapper con timeout configurato a 120 secondi.

Il file INF era deliberatamente essenziale:

```ini
[Version]
Signature="$Windows NT$"

[NewRequest]
Subject = "CN=CLIENT-01"
MachineKeySet = TRUE
KeyLength = 2048
Exportable = FALSE
KeySpec = 1
KeyUsage = 0xa0
ProviderName = "Microsoft RSA SChannel Cryptographic Provider"
ProviderType = 12
RequestType = PKCS10
Silent = TRUE
```

Nessun template nell'INF: il template doveva essere passato solo in fase di submit, per non coinvolgere il path template-driven durante la generazione locale. Il provider era dichiarato esplicitamente. Il test era ridotto al minimo necessario.

Risultato al termine dei 120 secondi: timeout. Stdout vuoto, stderr vuoto, nessun file `.req` creato, nessuna pending request nello store `Request`.

## L'evidenza che ha cambiato la direzione

Il processo `certreq.exe` al momento del timeout mostrava questo process tree:

```
cmd.exe (wrapper)
└── certreq.exe
    └── kchain.exe [C:\Program Files (x86)\Bit4id\UKC\UKC\bin\kchain.exe]
        └── kchain_gui.exe [C:\Program Files (x86)\Bit4id\UKC\UKC\bin\kchain_gui.exe]
```

`certreq.exe` stava aspettando `kchain.exe`, che aveva avviato `kchain_gui.exe`. Durante il test era comparso anche un popup dell'UKC di Bit4id. Il kill tree al termine del timeout confermava la catena: `kchain_gui.exe` figlio di `kchain.exe`, `kchain.exe` figlio di `certreq.exe`.

La CSP list pre-rimozione mostrava due provider Bit4id registrati nel sistema CryptoAPI locale, entrambi di tipo `PROV_RSA_FULL`, lo stesso tipo del provider Microsoft richiesto dall'INF:

```
Bit4id UKC Service Provider          - PROV_RSA_FULL
Bit4id Universal Middleware Provider - PROV_RSA_FULL
```

In parallelo erano registrati anche un Bit4id Key Storage Provider sul layer CNG e la libreria `bit4p11.dll` come PKCS#11 provider.

Il software installato era composto da tre componenti:

- Bit4id Universal MW 1.4.10.645
- Bit4id UKC 1.17.7.5
- Bit4id Firma4ng-InfoCamere 1.6.14

A questo punto l'analisi con **Process Monitor** ha mostrato che `certreq.exe` tentava di scrivere in un path `C:\4log` che non esisteva sul sistema. Ho creato il path per raccogliere evidenza ulteriore: una volta presente, `bit4p11.dll` ha iniziato a scrivere log binari. Il file creato durante il test era `bit4p11.dll.06_25-09_50_19.pid00002434.log.bin`, generato alle 11:50:19, esattamente nel secondo in cui `certreq.exe` era partito (StartTime=2026-06-25T11:50:17). La correlazione temporale confermava che `bit4p11.dll` era attiva durante l'operazione.

La diagnosi era chiara: il middleware Bit4id stava intercettando il path CryptoAPI prima che `certreq.exe` potesse completare la generazione della richiesta, bloccandosi in attesa di un token o di interazione utente tramite `kchain_gui.exe`, in un contesto (`NT AUTHORITY\SYSTEM`, modalità silente) in cui quella interazione non poteva avvenire.

## La decisione

Con change autorizzata, rimozione ufficiale dei tre componenti Bit4id/UKC, seguita da riavvio controllato. Nient'altro: nessun reset TPM, nessun rejoin, nessuna cancellazione manuale di MachineKeys o provider dal registro.

Dopo il riavvio, la CSP list mostrava solo i provider Microsoft nativi. Nessun processo o servizio Bit4id attivo. Le chiavi di registro Bit4id UKC e Universal Middleware risultavano rimosse; rimanevano alcune chiavi residue sotto `HKLM\SOFTWARE\bit4id` relative ai log e ai componenti PKI manager, senza impatto funzionale.

## Il risultato

Senza nessun intervento aggiuntivo, l'AutoEnrollment al primo ciclo successivo al riavvio ha acquisito automaticamente il certificato macchina. Gli eventi del log `CertificateServicesClient` erano precisi:

- **12:05:39** - Autenticazione al policy server LDAP completata
- **12:05:39** - Caricamento criteri dal server di enrollment completato
- **12:05:40** - Autenticazione alla CA `CASERVER.ACME.LOCAL\CA` completata
- **12:05:40** - Ricevuto certificato Machine_AutoEnroll con Request ID 124077 dalla CA
- **12:05:41** - AutoEnrollment per Sistema locale completato

Il certificato presente nello store `LocalMachine\My` al termine:

```
Template:    Machine_AutoEnroll
Issuer:      CN=CA, DC=ACME, LOCAL
NotBefore:   25/06/2026 11:55
NotAfter:    25/06/2027 11:55
Thumbprint:  A7F3C91E4B8D2065F0C9A13B7E42D58C6A90F1D3
Provider:    Microsoft RSA SChannel Cryptographic Provider
PrivateKey:  Non esportabile, associata, test crittografico superato
```

Store `Request` vuoto. Nessuna richiesta pendente residua.

## Cosa non fare in questi casi

La pressione a risolvere rapidamente un client che non si autentica in rete tende a far saltare la fase di analisi. Le azioni che vengono applicate per prime, spesso senza evidenza che servano, sono rejoin al dominio, reset del TPM, cancellazione manuale del MachineKeys.

Il rejoin è un'operazione con impatto reale: reset del secure channel, possibile rigenerazione di tutti i certificati macchina, effetti su configurazioni locali. In ambienti con BitLocker e TPM-bound keys il reset del TPM richiede una finestra di manutenzione dedicata.

Il principio è invertito rispetto all'istinto: prima si esclude tutto ciò che sta fuori dal client (CA, template, permessi, rete, chain), poi si isola il problema localmente con strumenti non mutativi, e solo alla fine si interviene con la modifica minima necessaria. In questo caso, quella modifica minima era la rimozione di tre package software.

## Trade-off e rischi della rimozione

Rimuovere Bit4id da un client aziendale non è neutro. In molti ambienti il middleware serve per l'accesso a smart card, CNS, token per firma digitale o accesso a servizi PA. La rimozione deve essere coordinata con chi gestisce queste esigenze, e in certi casi la soluzione non è rimuovere ma trovare una versione del middleware compatibile, oppure riconfigurarlo per non interferire con il path CryptoAPI nativo.

Nel caso specifico, la rimozione è stata la soluzione appropriata perché il middleware non era necessario per le funzioni primarie del client in quel contesto. La valutazione va fatta caso per caso.

## Implicazioni architetturali

Questo caso mette in luce un aspetto che raramente viene considerato nel design di ambienti enterprise con AD CS e AutoEnrollment: la coesistenza con middleware crittografici terzi non è trasparente.

I middleware per token e smart card registrano provider a livello di sistema, a volte sia su CryptoAPI (CAPI) che su CNG. Questo ha effetti globali: non solo sulle operazioni PKI esplicite, ma su qualsiasi path che attraversa il layer crittografico, incluso l'AutoEnrollment, TLS, Kerberos PKINIT. Quando un middleware si registra come `PROV_RSA_FULL`, può essere invocato da qualsiasi operazione crittografica di quel tipo, indipendentemente da quanto esplicito sia il provider dichiarato nell'INF o nella richiesta.

Le domande che un architetto dovrebbe porre prima del rollout, e che andrebbero incluse in un assessment:

- Quali middleware crittografici terzi sono presenti nella flotta? Sono standardizzati su una versione definita?
- È stato testato il comportamento di AutoEnrollment in loro presenza, nel contesto `NT AUTHORITY\SYSTEM` e in modalità silente?
- Le GPO di AutoEnrollment sono monitorate su eventi di fallimento, o il problema emerge solo quando un utente non riesce ad autenticarsi?
- Esiste un processo di change coordinato tra chi gestisce PKI e chi gestisce i client?

Il monitoraggio degli eventi `CertificateServicesClient` con alert sui failure di enrollment è una misura semplice che nella maggior parte degli ambienti non è attiva. Implementarla costa poco e riduce significativamente il tempo di rilevazione di questi problemi.

## Lesson learned

Il problema non era nella PKI. Non era nel template. Non era nella rete. Era in un software installato sul client che intercettava silenziosamente un meccanismo di sistema, in un contesto in cui non poteva completare la propria operazione.

La lezione non riguarda Bit4id specificamente: le evidenze raccolte riguardano una combinazione specifica di versioni, configurazione locale e sistema operativo, senza test comparativi su versioni alternative. Non è un advisory contro il prodotto.

La lezione è che i middleware crittografici terzi hanno visibilità e scope molto più ampi di quanto chi li installa generalmente considera, e che in un'architettura PKI enterprise questo non va dato per scontato ma mappato, testato e monitorato.

---

*Ambiente: Windows 11 25H2 (patch aggiornate al 25/06/2026). CA enterprise AD CS. Template Machine_AutoEnroll per EAP-TLS/802.1X.*
*Componenti Bit4id identificati: Universal MW 1.4.10.645, UKC 1.17.7.5, Firma4ng-InfoCamere 1.6.14.*
*Il caso è anonimizzato con valori generici. Non sono stati eseguiti test comparativi con versioni alternative del middleware.*
