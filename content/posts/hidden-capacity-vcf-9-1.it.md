---
title: "La capacità nascosta del data center: progettare VMware Cloud Foundation 9.1 prima di acquistare nuovo hardware"
date: 2026-08-04T17:40:00+02:00
draft: true
description: "Un cluster VMware al 65% di utilizzo non è necessariamente vicino alla saturazione. Come distinguere la capacità installata da quella utilizzabile prima di acquistare nuovo hardware, con VMware Cloud Foundation 9.1."
translationKey: "capacita-nascosta-vcf-9-1"
tags:
  - VMware Cloud Foundation
  - Private Cloud
  - Hybrid Cloud
  - Capacity Planning
  - vSAN
  - FinOps
  - Infrastructure
  - Enterprise Architecture
slug: "capacita-nascosta-vcf-9-1"
cover:
  image: "img/capacita-nascosta-vcf-9-1.png"
  alt: "Architect che analizza e ottimizza la capacità di un data center VMware Cloud Foundation 9.1"
  relative: true
---

## Introduzione

![Dal data center frammentato a una piattaforma governata](/img/vcf-capacity-transition.png)

Questo articolo nasce dall’esperienza maturata in molti anni di progettazione, trasformazione e gestione di infrastrutture VMware e, soprattutto, dai percorsi Architect di Broadcom. Questi corsi non si fermano alle funzioni dei prodotti: insegnano a rendere tracciabili le decisioni, a distinguere requisiti e vincoli e a dichiarare rischi e compromessi. Il metodo seguito qui è lo stesso: partire dal problema operativo, misurarlo e solo dopo scegliere la tecnologia.

Nel 2026 disponibilità e costo dell’hardware sono soltanto una parte del problema. Tempi di consegna incerti, crescita dei workload AI, maggiore densità computazionale e cambiamenti nei modelli di licensing rendono fragile una strategia basata sul semplice aumento del numero di server.

Quando un cluster VMware raggiunge il 60–70% di utilizzo, pianificare nuovi host è una reazione comprensibile, ma spesso anticipata. Quella percentuale non indica quale risorsa stia limitando il cluster, quanta capacità sia recuperabile, quanto pesino le VM sovradimensionate o quale margine debba restare disponibile per la resilienza. Non distingue neppure tra un limite di compute, memoria, storage, rete o architettura.

La domanda utile non è quanti server acquistare, ma quanta capacità utilizzabile esista già e quale architettura permetta di sfruttarla senza ridurre resilienza, supportabilità e prestazioni.

VMware Cloud Foundation 9.1 integra compute, storage, networking, sicurezza e operations in una piattaforma privata comune. Non corregge però, da solo, un disegno inefficiente. La tecnologia amplia le possibilità; il metodo stabilisce quali usare e perché.

Riferimenti Broadcom: [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html) · [Announcing VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/announcing-vcf-9-1-modern-private-cloud-built-for-efficiency-and-resilience/) · [Fleet Management](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/overview-of-vmware-cloud-foundation-9/what-is-vmware-cloud-foundation-and-vmware-vsphere-foundation/vcf-operations-overview/fleet-management.html)

## Il falso problema della percentuale di utilizzo

![La percentuale media non identifica il vero limite](/img/vcf-utilization-trap.png)

Un cluster al 65% non è necessariamente vicino alla saturazione. Un cluster al 40% non è necessariamente sano. Dietro la stessa media possono esserci memoria allocata ma poco utilizzata, vCPU in eccesso, datastore prossimi alla saturazione, latenze storage in crescita, distribuzione irregolare delle VM, vincoli NUMA, snapshot dimenticati o workload non più necessari.

Il primo errore è confondere la capacità installata con quella utilizzabile. In forma semplificata:

**Capacità utile = capacità fisica − riserva di resilienza − overhead − inefficienze − margine operativo**

La capacità fisica è il dato di targa. La capacità utile è quella assegnabile ai workload mantenendo il comportamento previsto durante manutenzioni, guasti e picchi. È questo valore, non la somma nominale delle risorse, a dover guidare capacity planning e investimenti.

Riferimenti Broadcom: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [Scale, Simplify, and Secure Private Cloud Operations with VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/scale-simplify-and-secure-your-private-cloud-operations-with-vcf-9-1/)

## Misurare il limite reale

![Analisi separata di compute, memoria, storage e rete](/img/vcf-four-domains.png)

Prima di introdurre nuovi host occorre individuare la risorsa che sta realmente esaurendo il proprio margine. In molti ambienti il compute appare saturo, mentre CPU e memoria dispongono ancora di capacità e il limite effettivo è lo storage: spazio netto, latenza, throughput, tempi di rebuild o dipendenza da un array arrivato a fine ciclo.

Aggiungere server in questo scenario aumenta la capacità computazionale ma lascia invariato il collo di bottiglia. Il risultato è nuovo hardware sottoutilizzato e un problema ancora aperto.

L’assessment va separato in quattro domini. Sul compute contano utilizzo medio e di picco, CPU Ready, Co-Stop, configurazione NUMA, rapporto vCPU/core e capacità residua dopo la perdita di un host. Sulla memoria vanno osservati working set, memoria attiva e consumata, ballooning, compressione e swapping. Sullo storage servono capacità netta, protezione, latenza, IOPS, throughput, crescita e tempi di ricostruzione. Sulla rete vanno verificati banda, ridondanza, oversubscription, packet loss, retransmission e latenza tra domini o siti.

Solo dopo questa analisi si può decidere se servano nuovo compute, nuovo storage, una revisione della rete o una migliore distribuzione delle risorse esistenti.

Riferimenti Broadcom: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html)

## Recuperare capacità attraverso il rightsizing

![Rightsizing come processo continuo di governo](/img/vcf-rightsizing.png)

Il rightsizing non è un taglio indiscriminato di RAM e vCPU. È un processo continuo di misura, correzione e verifica. Una VM sovradimensionata riduce l’efficacia di DRS, complica il posizionamento durante un failover, può creare configurazioni NUMA inefficienti e restringe il numero di host sui quali il workload può essere riavviato.

Le decisioni devono basarsi su dati storici che comprendano picchi, stagionalità, finestre batch, backup e crescita prevista. Ridurre una VM sulla sola media giornaliera è rischioso quanto lasciarla per anni al doppio della capacità necessaria.

Un processo solido osserva il comportamento, formula una proposta, coinvolge il referente applicativo, applica la modifica, verifica il risultato e mantiene un rollback praticabile. Il beneficio non è solo economico: una piattaforma più prevedibile è più semplice da proteggere, aggiornare e ampliare.

Riferimenti Broadcom: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [Scale, Simplify, and Secure Private Cloud Operations with VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/scale-simplify-and-secure-your-private-cloud-operations-with-vcf-9-1/)

## Memory Tiering: più capacità, ma non memoria gratuita

![DRAM e tier NVMe per estendere la capacità di memoria](/img/vcf-memory-tiering.png)

Memory Tiering è una funzione di ESX che utilizza dispositivi NVMe locali come livello di memoria secondario rispetto alla DRAM. Le pagine più attive restano nel tier più veloce; quelle meno attive possono essere collocate sul tier NVMe. L’obiettivo è aumentare la capacità effettiva quando la memoria è il vincolo reale e la CPU conserva margine. I dispositivi destinati al tier di memoria devono essere dedicati a questa funzione e non condivisi con datastore VM o con vSAN.

Non è equivalente allo swapping tradizionale e non sostituisce il rightsizing. Compatibilità delle VM, caratteristiche e endurance dei dispositivi NVMe, working set e sensibilità alla latenza devono essere verificati sul workload reale. Il beneficio va dimostrato attraverso metriche, test e osservazione operativa, non dedotto dalla sola quantità di memoria configurata.

Riferimenti Broadcom: [Memory Tiering considerations and best practices](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/9-1/vsphere-resource-management/memory-tiering-over-nvme/memory-tiering-considerations-and-best-practices.html) · [Using Memory Tiering for Resource Efficiency](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/using-resource-efficiency-for-memory-tiering.html) · [Configuring Memory Tiering in VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/19/more-memory-less-effort-configuring-memory-tiering-in-vcf-9-1/)

## Riutilizzare l’hardware, assegnandogli il ruolo corretto

![Hardware di generazioni diverse assegnato a ruoli coerenti](/img/vcf-capacity-analysis.png)

Riutilizzare hardware esistente non significa ignorare compatibilità, supporto o ciclo di vita. Significa separare il valore residuo del server dal ruolo che svolgeva in precedenza. Un host non più adatto ai workload più esigenti può restare valido per servizi infrastrutturali, ambienti di test o cluster con un profilo prestazionale prevedibile.

Anche il Management Domain è critico: non è un deposito naturale per hardware obsoleto. Il riuso è sostenibile solo quando capacità, disponibilità, driver, firmware e supporto restano coerenti con l’intero ciclo di vita previsto. La Broadcom Compatibility Guide e le indicazioni dell’OEM devono precedere la decisione economica.

Riferimenti Broadcom: [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html) · [Broadcom Compatibility Guide](https://compatibilityguide.broadcom.com/)

## Non forzare l’uniformità tra generazioni diverse

![Cluster omogenei e lifecycle separati](/img/vcf-infrastructure-domains.png)

Server dello stesso produttore ma di generazioni differenti possono richiedere firmware, driver, vendor add-on e impostazioni BIOS diversi. Trattarli come un unico insieme omogeneo aumenta il rischio e rende più complessa la remediation.

Con il lifecycle basato su immagini, l’obiettivo non è costruire un’immagine universale. È definire immagini coerenti per cluster omogenei e mantenere confini chiari tra gruppi hardware con requisiti differenti. L’eterogeneità non va eliminata per principio; va confinata, documentata e gestita.

Riferimento Broadcom: [Lifecycle Management in VMware Cloud Foundation](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/lifecycle-management-in-vmware-cloud-foundation.html)

## Scegliere lo storage in base al servizio, non alla moda

![Storage progettato in funzione delle classi di servizio](/img/vcf-storage-service-classes.png)

Adottare VCF non significa collocare ogni dato sullo stesso tipo di storage. La scelta dipende da prestazioni, protezione, resilienza, modello operativo, competenze, crescita, costo e compatibilità. Storage principale, storage supplementare, repository di backup e destinazioni di replica possono convivere, purché ogni ruolo sia esplicito.

vSAN ESA è una scelta naturale quando si vuole un’architettura HCI integrata e governata da policy, ma richiede hardware validato, rete adeguata e un calcolo realistico di capacità e resilienza. Una configurazione che si installa correttamente non è automaticamente una configurazione sostenibile per anni.

Riferimenti Broadcom: [What’s New in vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html) · [vSAN documentation](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/vsan-deployment-administration-and-monitoring.html)

## vSAN ESA: deduplica e compressione senza sovrastimare il beneficio

![Deduplica e compressione come leve misurabili di efficienza](/img/vcf-vsan-esa-efficiency.png)

VCF 9.1 rende generalmente disponibile la Global Deduplication per vSAN ESA e introduce un nuovo algoritmo di compressione always-on. La deduplica opera come processo post-processing a livello di cluster; la compressione viene applicata ai nuovi dati e, dopo un upgrade, ai blocchi esistenti quando vengono riletti e riscritti. Il risultato dipende dal tipo di dati, dalle policy e dal comportamento dei workload. Dati già compressi, cifrati o poco ripetitivi non offrono lo stesso rapporto di riduzione di template, desktop virtuali o sistemi operativi simili. Nella release 9.1 la Global Deduplication non è supportata nelle topologie stretched cluster e 2-node.

Il capacity planning deve partire dalla capacità raw, sottrarre protezione, slack operativo e spazio necessario alle ricostruzioni, quindi applicare un rapporto di riduzione prudenziale basato su misure reali. Deduplica e compressione possono rinviare un’espansione e ridurre il costo per terabyte utile; non devono eliminare il margine di sicurezza.

Riferimenti Broadcom: [What’s New in vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html) · [vSAN Compression and Global Deduplication in VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/07/vsan-compression-and-global-deduplication-in-vcf-9-1/) · [Using Deduplication and Compression in vSAN](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/vsan-deployment-administration-and-monitoring/administering-vmware-vsan/increasing-space-efficiency-in-a-vsan-cluster/using-deduplication-and-compression-in-vsan-cluster.html)

## Progettare il cluster sul guasto, non sulla normalità

![Capacità calcolata per manutenzione e scenari di guasto](/img/vcf-resilience-capacity.png)

Un cluster non va dimensionato soltanto per il funzionamento ordinario. Deve mantenere il servizio durante la perdita di un host, una manutenzione, un aggiornamento, la ricostruzione dello storage o la perdita di un fault domain prevista dal progetto.

La capacità totale non può quindi essere allocata per intero. Vanno sottratti il margine N+1 o superiore, l’overhead dell’hypervisor e dei servizi di gestione, lo spazio necessario alle manutenzioni, la crescita prevista e la capacità richiesta dalle policy di protezione.

Nei cluster stretched, il witness partecipa al quorum ma non aggiunge capacità eseguibile ai workload. La domanda corretta non è quante VM entrino nel cluster in condizioni normali, ma quante possano restare operative nello scenario di guasto dichiarato dall’architettura.

Riferimento Broadcom: [VMware vSAN Stretched Cluster](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/vsan-deployment-administration-and-monitoring/administering-vmware-vsan/setting-up-a-stretched-cluster.html)

## Consolidare senza creare un nuovo monolite

![Confini di cluster coerenti con hardware e requisiti](/img/vcf-cluster-boundaries.png)
/
Ridurre il numero di cluster limita la frammentazione, aumenta la libertà di posizionamento e semplifica il lifecycle. Consolidare tutto in un unico cluster, però, può creare un monolite difficile da governare.

I confini devono riflettere generazione hardware, sicurezza, licensing applicativo, criticità, profilo prestazionale, finestre di manutenzione, localizzazione e requisiti di disponibilità. Il cluster è sia un’unità tecnica sia un confine operativo: deve essere abbastanza grande da usare bene le risorse e abbastanza coerente da poter essere gestito come un solo oggetto.

Riferimento Broadcom: [VMware Cloud Foundation 9.1 Design](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/design.html)

## Il capacity planning come funzione FinOps

![Capacity management e costo come parte della stessa decisione](/img/vcf-operations-finops.png)

Ogni core inutilizzato, ogni terabyte non governato e ogni VM sovradimensionata produce un effetto economico. Capacity e cost management devono quindi essere letti insieme, includendo costo della capacità inutilizzata, resilienza, licensing, rinnovo hardware e debito tecnico.

Applicare FinOps al private cloud non significa necessariamente fatturare ogni VM a un centro di costo. Significa rendere visibile il legame tra scelta tecnica e conseguenza economica. Una VM sovradimensionata può anticipare un investimento; un cluster frammentato può moltiplicare riserve che non possono essere condivise.

Riferimenti Broadcom: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [Scale, Simplify, and Secure Private Cloud Operations with VCF 9.1](https://blogs.vmware.com/cloud-foundation/2026/05/05/scale-simplify-and-secure-your-private-cloud-operations-with-vcf-9-1/)

## VCF Operations come sistema decisionale

![Operations come ciclo continuo di misura e correzione](/img/vcf-operations-decision-cycle.png)

Un sistema di monitoraggio tradizionale segnala ciò che non funziona. Una piattaforma di operations matura deve indicare anche ciò che sta per esaurirsi, dove esiste capacità recuperabile, quali VM sono sovradimensionate, quali host limitano il lifecycle e quale investimento può essere rinviato.

Fleet Management centralizza la gestione delle istanze e dei componenti registrati nella fleet. Deployment, validazione iniziale e lifecycle appartengono a workflow distinti ma coordinati. Il valore nasce quando le dashboard alimentano un processo: raccolta dei dati, baseline, analisi, azioni correttive, verifica e aggiornamento del piano di capacità.

Riferimenti Broadcom: [Fleet Management](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/overview-of-vmware-cloud-foundation-9/what-is-vmware-cloud-foundation-and-vmware-vsphere-foundation/vcf-operations-overview/fleet-management.html) · [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html)

## Acquistare soltanto dopo aver definito l’architettura target

![Framework architetturale prima della distinta base](/img/vcf-architecture-framework.png)

Prima di ordinare nuovi server è opportuno aver definito Management e Workload Domain, numero e dimensione dei cluster, hardware riutilizzabile, lifecycle, storage, rete, fault domain, capacità N+1 o N+2, crescita, rapporto core/memoria, uso eventuale del Memory Tiering e ipotesi prudenti sulla data reduction di vSAN ESA.

Solo dopo ha senso costruire la distinta base. Questo evita sia l’acquisto di server potenti ma inutilizzati perché il limite è altrove, sia l’acquisto di nodi sbilanciati rispetto al carico e al modello storage. Non serve comprare più velocemente; serve comprare con dati migliori e all’interno di un disegno già definito.

Riferimenti Broadcom: [VMware Cloud Foundation 9.1 Design](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/design.html) · [Broadcom Compatibility Guide](https://compatibilityguide.broadcom.com/)

## Un possibile modello evolutivo

![Percorso progressivo dall’assessment alle operations](/img/vcf-evolution-model.png)

Un percorso pragmatico può articolarsi in cinque fasi. L’assessment inventaria componenti, verifica compatibilità, misura capacità reale, identifica il collo di bottiglia e avvia il rightsizing. Il disegno definisce domini, cluster, storage, rete, resilienza e lifecycle. Il riuso controllato assegna l’hardware ai ruoli compatibili e mantiene cluster omogenei. L’investimento mirato introduce solo ciò che serve a rimuovere un limite misurato. Le operations continue aggiornano capacity plan, soglie, lifecycle e modelli di costo.

Queste fasi non sono una sequenza rigida: alcune attività si sovrappongono e i dati raccolti durante le operations possono richiedere la revisione delle decisioni iniziali. Il punto fermo è la tracciabilità delle scelte.

Riferimenti Broadcom: [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html) · [Lifecycle Management](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-0/lifecycle-management-in-vmware-cloud-foundation.html)

## Le decisioni che fanno davvero risparmiare hardware

![Dalla capacità nascosta a una piattaforma governata](/img/vcf-data-efficiency.png)

Il risparmio raramente dipende da una singola funzione. Deriva dalla combinazione di decisioni coerenti: distinguere capacità fisica e capacità utile, dimensionare sul guasto, identificare il collo di bottiglia, applicare rightsizing, consolidare dove opportuno, separare lifecycle diversi, usare Memory Tiering e data reduction soltanto quando i dati ne dimostrano il beneficio.

L’assessment può confermare la necessità di nuovi server, spostare l’investimento verso storage o rete oppure rinviare l’acquisto. Sono tutti esiti validi. L’obiettivo non è evitare nuovo hardware, ma evitare di comprarlo per compensare un problema non ancora compreso.

Riferimenti Broadcom: [VCF Operations 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vcf-ops.html) · [vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html)

## Conclusioni

![Una piattaforma più efficiente, resiliente e prevedibile](/img/vcf-governed-platform.png)

VMware Cloud Foundation 9.1 può essere il punto di partenza per un private cloud più integrato e governabile, ma il beneficio principale non deriva dall’installazione della piattaforma. Deriva da un diverso modo di prendere le decisioni infrastrutturali.

L’hardware esistente non va mantenuto indiscriminatamente né sostituito per abitudine. Va assegnato al ruolo nel quale può continuare a produrre valore. Il nuovo hardware deve rimuovere un limite misurato, non un’ipotesi. In un private cloud maturo, l’ottimizzazione non è una misura temporanea in attesa del prossimo acquisto: è parte dell’architettura.

Riferimenti finali: [VMware Cloud Foundation 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1.html) · [Memory Tiering best practices](https://techdocs.broadcom.com/us/en/vmware-cis/vsphere/vsphere/9-1/vsphere-resource-management/memory-tiering-over-nvme/memory-tiering-considerations-and-best-practices.html) · [vSAN 9.1](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-service-administration-and-development/9-1/whats-new-vsan.html)
