---
title: "VCF 9 Holodeck LAB: come ho sfruttato al massimo 128 core per un laboratorio Full Stack"
date: 2026-11-12T00:00:00+01:00
draft: true
description: "Come ho riconfigurato la CPU topology di sette host ESX nested in Holodeck per utilizzare i 128 core della licenza personale VCF ottenuta tramite VCP-VCF Architect e VMUG Advantage."
slug: "vcf-9-holodeck-128-core-topologia-cpu"
translationKey: "vcf-9-holodeck-128-core-cpu-topology"
tags: ["VMware", "VCF 9", "Holodeck", "Homelab", "Licensing", "Nested ESX", "VMUG"]
categories: ["VMware"]
showToc: true
TocOpen: false
cover:
  image: "/img/vcf-holodeck-128-cores/00-cover.webp"
  alt: "Laboratorio VMware Cloud Foundation 9: quattro host Management e tre Workload, per un totale di 128 core licenziati"
  caption: "VCF 9 in Holodeck: Full Stack in 128 Cores"
  relative: true
---

## Il problema non erano i core. O forse sì?

Allestire un laboratorio VMware Cloud Foundation completo è un esercizio interessante: bisogna fare i conti con requisiti infrastrutturali, risorse disponibili e, naturalmente, licensing.

Nel mio caso stavo lavorando con **Holodeck e VMware Cloud Foundation 9**, in un ambiente nested composto da **sette host ESX**: quattro per il **Management Domain** e tre per un **Workload Domain**. Era proprio l'architettura che volevo mantenere per studiare e sperimentare le funzionalità della piattaforma.

L'infrastruttura era operativa, ma al momento di verificare la copertura delle licenze è emerso un problema: **i 128 core della mia licenza VCF personale erano già interamente assorbiti dal Management Domain**. Per i tre host del Workload Domain non rimaneva alcuna capacità.

<!-- IMG 01 — Copiare la propria schermata di inventario in static/img/vcf-holodeck-128-cores/01-holodeck-inventory.webp -->
![Inventario Holodeck con quattro host ESX nel Management Domain e tre nel Workload Domain](/img/vcf-holodeck-128-cores/01-holodeck-inventory.webp)
*Figura 1 — La topologia del laboratorio Holodeck: sette host ESX nested suddivisi fra i due domini.*

Prima di ridurre il numero di host o rinunciare al secondo dominio, ho voluto capire **come VCF calcolasse effettivamente i core da licenziare**.

## Licensing VCF: il minimo di 16 core per CPU

Il licensing di VMware Cloud Foundation (VCF) e VMware vSphere Foundation (VVF) prevede un **minimo di 16 core per CPU fisica (socket)**, anche quando un processore ne possiede meno. Per un host con più processori, il minimo si applica a ciascuna CPU.

In forma semplificata, per ciascun host:

`Core da licenziare = somma, per ogni socket, di max(16, core del socket)`

| Topologia della CPU | Core presenti | Core da licenziare |
|---|---:|---:|
| 1 socket × 8 core | 8 | 16 |
| 1 socket × 12 core | 12 | 16 |
| 1 socket × 20 core | 20 | 20 |
| 2 socket × 12 core | 24 | 32 |
| 2 socket × 16 core | 32 | 32 |
| 2 socket × 24 core | 48 | 48 |

Su un host fisico il numero di processori è una caratteristica dell'hardware. In un **ambiente ESX nested**, invece, il sistema che esegue ESX è una VM: il numero di socket e di core che vengono presentati al guest dipende dalla configurazione del virtual hardware.

Non è soltanto una considerazione teorica. La [Broadcom KB 431120](https://knowledge.broadcom.com/external/article/431120/resolving-unexpected-license-capacity-re.html) descrive proprio il caso di host ESX virtualizzati il cui conteggio delle licenze, in VCF Operations 9.0, risulta superiore alle aspettative a causa della CPU topology.

## Il mio Holodeck: quattro host consumavano tutti i 128 core

La configurazione iniziale degli host del Management Domain prevedeva **2 socket virtuali da 16 core ciascuno**, ossia **32 vCPU per host**.

| Management Domain | Topologia iniziale | Core richiesti |
|---|---|---:|
| ESX 01 | 2 socket × 16 core | 32 |
| ESX 02 | 2 socket × 16 core | 32 |
| ESX 03 | 2 socket × 16 core | 32 |
| ESX 04 | 2 socket × 16 core | 32 |
| **Totale** | **4 host** | **128** |

Non era necessario sommare anche il Workload Domain per individuare il problema: **128 core su 128 erano già impegnati**.

<!-- IMG 02 — Screenshot delle impostazioni originali oppure schema esplicitamente indicato come ricostruzione: static/img/vcf-holodeck-128-cores/02-mgmt-cpu-before.webp -->
![Configurazione iniziale di un host ESX nested: 32 vCPU, due socket e 16 core per socket](/img/vcf-holodeck-128-cores/02-mgmt-cpu-before.webp)
*Figura 2 — Configurazione iniziale del Management Domain: 2 socket × 16 core per host.*

Va considerato che questa configurazione iniziale era coerente con un’impostazione prestazionale corretta: in scenari di questo tipo, ESX tende infatti a presentare una topologia CPU allineata il più possibile all’hardware sottostante, anche in ottica **NUMA** e località della memoria.

Detto questo, **passare da 2 × 16 a 1 × 32 non avrebbe ridotto il consumo di licenza**: in entrambi i casi il totale sarebbe rimasto pari a 32 core. Per ottenere un vantaggio reale era quindi necessario intervenire non solo sulla distribuzione tra socket virtuali, ma anche **sul numero complessivo di vCPU assegnate**.

## La scelta: 80 core al Management, 48 al Workload

Il mio obiettivo era **sfruttare al meglio i 128 core concessi per studio e sperimentazione**, preservando l'architettura a sette host e accettando un compromesso sulle risorse computazionali virtuali.

Per il Management Domain ho modificato ogni host da **2 socket × 16 core** a **1 socket × 20 core**. Il fabbisogno è così passato da 32 a 20 core per host: **4 × 20 = 80 core**.

<!-- IMG 03 — Copiare la configurazione finale CPU della VM ESX Management in static/img/vcf-holodeck-128-cores/03-mgmt-cpu-after.webp -->
![CPU topology di un host ESX Management dopo la modifica: 20 vCPU e un socket da 20 core](/img/vcf-holodeck-128-cores/03-mgmt-cpu-after.webp)
*Figura 3 — Management Domain: 20 vCPU, presentate come un unico socket da 20 core.*

Rimanevano **48 core**, sufficienti per configurare i tre host del Workload Domain con **1 socket × 16 core ciascuno**.

<!-- IMG 04 — Copiare la configurazione finale CPU della VM ESX Workload in static/img/vcf-holodeck-128-cores/04-wld-cpu-after.webp -->
![CPU topology di un host ESX Workload: 16 vCPU e un socket da 16 core](/img/vcf-holodeck-128-cores/04-wld-cpu-after.webp)
*Figura 4 — Workload Domain: 16 vCPU e un socket per ciascuno dei tre host.*

| Dominio | Host | CPU topology finale | Core da licenziare |
|---|---:|---|---:|
| Management Domain | 4 | 1 socket × 20 core | 80 |
| Workload Domain | 3 | 1 socket × 16 core | 48 |
| **Totale** | **7** | | **128** |

**128 core disponibili, 128 core utilizzati.** Nessun host eliminato e nessun workload domain sacrificato.

## Procedura: modificare la CPU topology di un ESX 9 nested

La modifica **non si esegue dall'interno dell'ESX nested**. Bisogna intervenire sulle impostazioni della **VM ESX eseguita dal ESX installato sul Sever fisico**, collegandosi al vSphere Client dell'infrastruttura sottostante, quella che ospita Holodeck.

### 1. Verificare CPU e virtual hardware

Nel vSphere Client del livello *outer*, selezionare la VM ESX e aprire **Edit Settings**. Prendere nota del numero totale di vCPU, della compatibilità virtual hardware e dei core per socket.

Il percorso del parametro dipende dalla versione di vSphere e dal virtual hardware della VM:

- **vSphere 8 e virtual hardware versione 20 o superiore:** `Edit Settings → VM Options → CPU Topology`.
- **Virtual hardware precedente:** il parametro può trovarsi in `Edit Settings → Virtual Hardware → CPU`.

La posizione dei controlli è spiegata nella [Broadcom KB 407648](https://knowledge.broadcom.com/external/article/407648/cores-per-socket-setting-location-has-be.html). Il fatto che il guest sia ESX 9 non cambia il livello al quale si applica la modifica: conta la versione dell'hypervisor che ospita la VM.

### 2. Preparare l'arresto dell'host

Prima di intervenire, verificare dove risiedono le VM e i servizi del cluster. Se è presente vSAN, valutare la modalità di manutenzione e le conseguenze sulla disponibilità e sull'accessibilità dei dati. Non modificare contemporaneamente tutti gli host di un dominio senza aver verificato le dipendenze.

Arrestare correttamente il sistema ESX nested e **spegnere la relativa VM** sul livello outer.

### 3. Impostare la nuova topologia

Con la VM spenta, riaprire **Edit Settings** e applicare uno dei due profili:

| Parametro | Host Management | Host Workload |
|---|---:|---:|
| **CPU totali (vCPU)** | **20** | **16** |
| **Cores per Socket** | **20** | **16** |
| Socket virtuali risultanti | 1 | 1 |

Se è attiva l'assegnazione automatica della CPU topology (*Assigned at power on* / *Automatic vTopology*), impostare manualmente i core per socket desiderati e verificare il numero risultante di socket.

Salvare, riaccendere la VM ESX e attendere che l'host torni operativo nel relativo cluster. Ripetere l'operazione **un host alla volta**, secondo lo stato dei servizi.

### 4. Controllare la topologia rilevata da ESX 9

Dal vSphere Client che gestisce il laboratorio nested, selezionare l'host e verificare nella sezione **Hardware / Processors** la topologia presentata al sistema.

Il risultato atteso è:

- **Management:** 1 socket, 20 core.
- **Workload:** 1 socket, 16 core.

<!-- IMG 05 — Copiare la vista Hardware / Processors dell'host ESX nested in static/img/vcf-holodeck-128-cores/05-esx9-guest-topology.webp -->
![Hardware processori nell'host ESX 9 nested: topologia CPU realmente rilevata dopo la modifica](/img/vcf-holodeck-128-cores/05-esx9-guest-topology.webp)
*Figura 5 — La verifica nel guest ESX 9: la topologia deve corrispondere a quella impostata sull'outer hypervisor.*

### 5. Verificare il conteggio in VCF Operations

Infine, controllare la capacità richiesta dal sistema di licensing. In VCF 9 il licensing viene gestito centralmente tramite VCF Operations e VCF Business Services Console: non basta che le impostazioni della VM siano corrette, bisogna verificare il dato effettivamente rilevato dalla piattaforma.

Il percorso nell'interfaccia cambia con la release:

- **VCF Operations 9.0:** `License Management → Registration`.
- **VCF Operations 9.1:** `Manage → Licensing → Licenses & Registration`.

In base alla modalità connessa o disconnessa, potrebbe essere necessario aggiornare o importare i dati della licenza. Verificare poi la capacità assegnata ai vCenter e il numero di core richiesti dai sette host.

<!-- IMG 06 — Copiare il pannello Licensing VCF Operations con capacità totale di 128 core in static/img/vcf-holodeck-128-cores/06-vcf-ops-128-cores.webp -->
![VCF Operations: capacità di licensing e conteggio degli host per un totale di 128 core](/img/vcf-holodeck-128-cores/06-vcf-ops-128-cores.webp)
*Figura 6 — Risultato finale da verificare in VCF Operations: 80 core Management + 48 core Workload.*

## Non è un'ottimizzazione delle prestazioni

Questa scelta aveva una finalità precisa: **utilizzare al meglio il licensing disponibile per le attività di laboratorio**.

Riducendo le risorse assegnate ai sistemi nested, ho rinunciato a una parte della capacità computazionale virtuale originaria. È possibile che, sotto carico, questa configurazione accentui fenomeni di CPU contention o aumenti alcuni tempi di risposta; non dispongo però ancora di misure che ne quantifichino l'impatto reale.

Un laboratorio nested aggiunge inoltre variabili proprie, come la schedulazione delle vCPU sull'hypervisor outer e il rapporto con la topologia NUMA dell'hardware sottostante.

Per me è un compromesso accettabile in un ambiente di studio. **Non è un criterio di dimensionamento da trasferire automaticamente a un'installazione di produzione**, dove hardware, workload, supportabilità e licensing devono essere valutati insieme.

## La licenza da 128 core: VCP-VCF Architect e VMUG Advantage

I 128 core non provengono da una normale licenza commerciale. Nel mio caso, dopo aver ottenuto la certificazione **VCP-VCF Architect** e aderito a **VMUG Advantage**, ho avuto accesso all'entitlement **VMware Cloud Foundation da 128 core per uso personale**.

È un'opportunità concreta per chi vuole studiare la piattaforma con un'installazione reale e non limitarsi alle sole esercitazioni guidate.

Il programma descritto da VMware e VMUG prevede una licenza **non-production, senza supporto, valida inizialmente un anno**, con possibilità di rinnovo annuale fino a tre anni alle condizioni del programma, mantenendo attiva la membership richiesta. I requisiti e le condizioni sono soggetti ad aggiornamenti: vale sempre la pena controllarli prima di pianificare il proprio laboratorio.

Giovanni Dominoni ha raccontato bene i benefici e il percorso di accesso nell'articolo [*Homelab VMware License: a great opportunity for VVF and VCF certified people*](https://www.giovannidominoni.it/homelab-vmware-license-a-great-opportunity-for-vvf-and-vcf-certified-people/). È una lettura che consiglio per approfondire il tema, mentre per i dettagli aggiornati è utile consultare la documentazione [VMUG Advantage](https://www.vmug.com/advantage).

<!-- IMMAGINE FACOLTATIVA 07 — Se vuoi aggiungere il riepilogo della licenza, inserisci uno screenshot anonimizzato in static/img/vcf-holodeck-128-cores/07-vmug-entitlement.webp e decommenta la riga seguente. NON pubblicare license key, token, ID o dati personali. -->
<!-- ![Entitlement VCF da 128 core per uso personale nel portale Broadcom](/img/vcf-holodeck-128-cores/07-vmug-entitlement.webp) -->

## E adesso? Misurare il comportamento del laboratorio

La riconfigurazione ha risolto il problema di licensing, ma apre una nuova attività di studio: **come si comporta VCF quando le risorse CPU sono limitate?**

Nei prossimi test voglio analizzare CPU utilization, CPU Ready, co-stop, eventuale contention e comportamento delle appliance, distinguendo per quanto possibile gli effetti del dimensionamento da quelli introdotti dalla virtualizzazione nested.

I risultati di un laboratorio come questo non saranno automaticamente rappresentativi di sistemi fisici. Possono però aiutare a capire meglio i compromessi di un'architettura e a formulare domande più interessanti quando ci si trova a dimensionare una piattaforma reale.

## Conclusioni

Questa esperienza mi ha ricordato quanto sia importante distinguere tre grandezze: **risorse fisiche, risorse virtuali e capacità da licenziare**. Sono correlate, ma non coincidono.

Modificando la CPU topology delle VM ESX sono riuscito a utilizzare tutti i 128 core della mia licenza personale VCF, mantenendo i sette host e la separazione fra Management e Workload Domain.

Il compromesso è stata una minore dotazione di vCPU, che valuterò con test mirati. In cambio ho mantenuto un ambiente completo, correttamente licenziato e utile per sperimentare.

**A volte sono proprio i limiti del laboratorio a insegnarci qualcosa.**

## Riferimenti

- [Broadcom KB 313548 — Counting Cores for VMware Cloud Foundation and vSphere Foundation](https://knowledge.broadcom.com/external/article/313548).
- [Broadcom KB 431120 — Resolving unexpected License capacity requirements in VCF Operations](https://knowledge.broadcom.com/external/article/431120/resolving-unexpected-license-capacity-re.html).
- [Broadcom KB 407648 — Cores per Socket setting location in vSphere](https://knowledge.broadcom.com/external/article/407648/cores-per-socket-setting-location-has-be.html).
- [Broadcom KB 453634 — Adding ESX host to vCenter 9.x: license capacity and VCF Operations paths](https://knowledge.broadcom.com/external/article/453634/adding-esx-host-to-vcenter-9x-fails-with.html).
- [VMUG — VCP-VCF License Access for Personal Use (programma e requisiti)](https://www.vmug.com/wp-content/uploads/2025/03/VCP-VCF-License-Access-for-Personal-Use-Advantage-Benefit_March_2025.pdf).
- [Giovanni Dominoni — Homelab VMware License: a great opportunity for VVF and VCF certified people](https://www.giovannidominoni.it/homelab-vmware-license-a-great-opportunity-for-vvf-and-vcf-certified-people/).
