---
title: "DOOM su VCF 9.1: un workload nerd per il Rightsizing"
date: 2026-10-21T10:15:00+02:00
draft: false
description: "Alpine Linux, DOSBox e DOOM in un Workload Domain VCF 9.1: un esperimento minimalista per esplorare Rightsizing e What-If in VCF Operations."
translationKey: "vcf-9-1-doom-rightsizing"
tags:
  - "VMware Cloud Foundation"
  - "VCF Operations"
  - "Rightsizing"
  - "Reclaim"
  - "Capacity Planning"
  - "Holodeck"
  - "Linux"
  - "DOSBox"
slug: "doom-su-vcf-9-1-rightsizing"
cover:
  image: "/img/vcf-9-1-doom-rightsizing.webp"
  alt: "DOOM in esecuzione su Alpine Linux nel laboratorio VMware Cloud Foundation 9.1"
  relative: true
---

**Minimal footprint, minimal effort, maximum fun.**

Nel mio laboratorio **VMware Cloud Foundation 9.1**, realizzato con **Holodeck**, volevo introdurre un workload minimo ma continuamente attivo nel *Workload Domain*. L'obiettivo non era misurare le prestazioni del gioco, ma dare a **VCF Operations** qualcosa di reale da osservare, per sperimentare **Rightsizing**, analisi della capacità e scenari **What-If**.

Potevo usare uno stress test o uno script. Ho preferito qualcosa di più nerd: **far girare DOOM**.

## Architettura: il minimo indispensabile

La scelta è ricaduta su **Alpine Linux**, senza un desktop environment completo: solo Xorg, un terminale grafico e **DOSBox Staging** per eseguire la versione shareware di DOOM.

| Componente | Configurazione del laboratorio |
|---|---|
| Infrastruttura | VCF 9.1 su Holodeck |
| VM | `vm-retro`, nel Workload Domain |
| Sistema operativo | Alpine Linux 3.24.1 x86_64 |
| Compute | 1 vCPU, 512 MB RAM |
| Storage | 8 GB, thin provisioned |
| Display | VMware SVGA / Xorg |
| Emulatore | DOSBox Staging 0.82.2 |
| Applicazione | DOOM Shareware 1.9 |

Il criterio architetturale era semplice: **ridurre footprint e complessità senza rinunciare a un processo applicativo osservabile**.

## Dalla console a DOOM

Con i repository Alpine correttamente configurati (incluso `community`), i componenti principali si installano così:

```sh
setup-xorg-base
apk update
apk add dosbox-staging sdl2 xterm
```

Nel mio caso è emerso un piccolo problema di compatibilità grafica: l'uso dello strato `sdl2-compat` basato su SDL3 causava problemi all'avvio dell'emulatore. La soluzione è stata utilizzare **SDL2 nativo**.

Dopo aver predisposto Xorg e la sessione con `xterm`, l'avvio è essenziale:

```sh
# Sessione grafica dell'utente non privilegiato
startx

# Da xterm
dosbox
```

DOSBox deve avere una directory Linux montata come unità DOS `C:`. Per esempio, **adattando il percorso alla propria installazione**:

```dos
MOUNT C /home/retro/dos
C:
CD \GAMES\DOOMS
DOOM.EXE
```

Nel mio ambiente il percorso di gioco era `C:\GAMES\DOOMS` e l'eseguibile utilizzato era `DOOMS.EXE`; nell'archivio shareware standard è normalmente `DOOM.EXE`.

DOOM offre una modalità dimostrativa automatica (*attract mode*): lasciandolo alla schermata iniziale, riproduce ciclicamente le demo senza richiedere interazione. Un carico reale, leggero, non perfettamente costante ma **ripetibile e osservabile**.

## Dalla telemetria alle decisioni: Reclaim, Rightsize e What-If

DOOM non è un benchmark: è un modo semplice per produrre attività osservabile. Il passo successivo è utilizzare il **Capacity Engine di VCF Operations**, che combina metriche, storico e policy per supportare decisioni operative. Le schermate seguenti riguardano **l'insieme delle VM del cluster di laboratorio**, non esclusivamente la VM che esegue DOOM.

### Reclaim — che cosa posso recuperare?

La vista **Reclaim** individua capacità non più utilizzata: VM spente o inattive, snapshot e dischi orfani. Nel cluster `cluster-wld01-01a` vengono rilevate **due VM spente** (`retro-vm-3` e `retro-vm-bck`) e **un disco orfano**: **3,15 GB** complessivamente recuperabili, con un **risparmio potenziale stimato di 6 €/mese**. Non è una riduzione di spesa già realizzata: il valore dipende dal modello di costo configurato. Prima di eliminare risorse, naturalmente, vanno verificate proprietà, dipendenze e retention.

![VCF Operations: Reclaim, capacità recuperabile nel cluster di laboratorio](/img/vcf-9-1-reclaim.webp)

### Rightsize — quante risorse servono veramente?

**Rightsize** confronta le risorse assegnate con il fabbisogno previsto, identificando VM *oversized* e *undersized*. Nell'esempio, `retro-vm-2` ha **4 vCPU e 4 GB RAM**, ma VCF Operations propone una riduzione di **2 vCPU e 2 GB**: configurazione suggerita **2 vCPU / 2 GB**. Nessuna VM risulta *undersized*. È una **raccomandazione**, non una modifica già applicata: va validata rispetto ai picchi applicativi e alle policy di rischio.

![VCF Operations: Rightsize, raccomandazione di riduzione CPU e memoria](/img/vcf-9-1-rightsize.webp)

### What-If — e se domani aggiungessi un workload?

Con **What-If Analysis** ho simulato *New-Retro-Quake*: **una nuova VM ipotetica**, distinta dalla piccola VM DOOM, da collocare nello stesso cluster. Lo scenario distingue **Allocation** (**2 vCPU, 6 GB RAM, 80 GB disco**) e **Demand** stimata (**600 MHz CPU, 614,4 MB RAM, 8 GB disco**). Questo evita di confondere risorse configurate e risorse effettivamente richieste.

Il risultato è **“The workload will fit”**: costo modellato **27 €/mese** e **Time Remaining superiore a un anno**. I picchi previsti con il workload aggiunto sono **39,11 / 48,24 GHz CPU**, **110,01 / 160,79 GB RAM** e **256,12 GB / 1,98 TB disco** (domanda prevista / capacità disponibile). È una **simulazione**, non una prenotazione di capacità né una garanzia di performance.

![VCF Operations: simulazione What-If per l'aggiunta di un workload HCI](/img/vcf-9-1-what-if.webp)

Per osservare anche l'andamento della previsione: [dettaglio delle proiezioni CPU, memoria e storage](/img/vcf-9-1-what-if-forecast.webp).

**Il punto dell'esercizio:** passare dal monitoraggio alle decisioni — *recuperare ciò che non serve, dimensionare ciò che esiste, pianificare ciò che arriverà*. Tenendo presente che le stime dipendono da **storico, policy, modello Demand/Allocation e profilo del carico**.

## Conclusioni

Una VM Alpine da 512 MB, un emulatore open source e un videogioco del 1993 sono bastati per trasformare un test di Capacity Management in un piccolo laboratorio di retrocomputing.

**Minimal footprint, minimal effort, ottima resa per l'esercizio.** E sì, vedere DOOM girare dentro VMware Cloud Foundation ha un fascino particolare.

### Fonti e repository

- [VMware Holodeck – repository e documentazione](https://github.com/vmware/Holodeck)
- [Alpine Linux](https://alpinelinux.org/)
- [DOSBox Staging – repository](https://github.com/dosbox-staging/dosbox-staging)
- [DOOM v1.9 Shareware – archivio Doomworld /idgames](https://legacy.doomworld.com/idgames/idstuff/doom/doom19s)
- [VMware – What's New in Capacity Management for VCF 9.0](https://blogs.vmware.com/cloud-foundation/2025/08/06/whats-new-in-capacity-management-for-vmware-cloud-foundation-9/)
- [VMware – Rightsizing VMs with VMware Cloud Foundation](https://blogs.vmware.com/cloud-foundation/2025/08/12/vmware-cloud-foundation-helps-you-optimally-size-vms-and-keep-your-infrastructure-and-workloads-running-smoothly/)

*Nota su licenze e utilizzo:* DOSBox Staging è distribuito sotto licenza **GPL-2.0-or-later**; **DOOM Shareware** rimane software proprietario di id Software ed è soggetto alle condizioni incluse nella distribuzione originale. Il laboratorio descritto è personale e non commerciale; questo articolo **non distribuisce file del gioco** né implica che la versione shareware sia liberamente utilizzabile in dimostrazioni commerciali. L'utilizzo di VCF deve essere coperto dalle relative licenze e autorizzazioni.
