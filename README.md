# Penetration Testing & Vulnerability Assessment: Athena

Repository contenente la documentazione e i risultati dell'attività di Vulnerability Assessment e Penetration Testing (VAPT) condotta sul target "Athena", svolta in ambito accademico (Università di Catania) seguendo lo standard metodologico PTES (Penetration Testing Execution Standard).

Il report tecnico completo è consultabile nel file [Report_1000044165_Petronio_Petronio_Davide.pdf](./Report_1000044165_Petronio_Petronio_Davide.pdf).

---

## Panoramica dell'Attività

L'obiettivo dell'attività è stato valutare il livello di sicurezza complessivo della macchina target, simulando le azioni di un attaccante esterno per identificare vulnerabilità nei servizi esposti, concatenarle in una catena di exploit (vulnerability chain) e verificare la possibilità di compromissione totale del sistema fino all'ottenimento dei massimi privilegi di amministratore (`root`).

L'analisi ha coperto tutte le fasi canoniche del processo di sicurezza offensiva:
- Ricognizione e scansione dei servizi di rete
- Analisi delle vulnerabilità note e misconfigurazioni
- Sfruttamento (Exploitation) e acquisizione dell'accesso iniziale
- Privilege escalation orizzontale e verticale
- Analisi statica e reverse engineering di un modulo kernel (rootkit)
- Post-exploitation e ripristino dell'integrità del sistema (clean-up)
- Valutazione dell'impatto con scoring CVSS, mappatura CWE e linee guida di mitigazione

---

## Metodologia e Standard

L'attività è stata strutturata in conformità con il **Penetration Testing Execution Standard (PTES)**, articolandosi nelle seguenti fasi:

1. **Pre-engagement Interactions**: Definizione dello scope, degli obiettivi e delle regole di ingaggio.
2. **Vulnerability Analysis**: Scansione automatizzata e manuale dei servizi esposti per l'individuazione di debolezze e punti di ingresso.
3. **Exploitation**: Esecuzione mirata di exploit per validare l'effettiva sfruttabilità delle falle riscontrate.
4. **Post-Exploitation**: Verifica dell'impatto sul target, acquisizione delle flag di prova (`user.txt` e `root.txt`) e rimozione completa degli artefatti generati durante i test.
5. **Reporting**: Redazione di un documento dettagliato con analisi del rischio, scoring CVSS e raccomandazioni di remediation applicabili.

---

## Tool e Competenze Tecniche

- **Network Scanning & Enumeration**: Nmap, OpenVAS, smbclient, dirsearch
- **Web Application Security**: Burp Suite, cURL, Reverse Shell Bash/BusyBox
- **Privilege Escalation & Process Monitoring**: pspy64, Linux Privilege Escalation methodologies
- **Reverse Engineering**: Ghidra (disassembly e decompilazione C di moduli kernel `.ko`)
- **Framework di Valutazione**: CVSS v4.0 / v2.0, standard CWE

---

## Catena di Attacco (Vulnerability Chain - CVSS 9.3 Critical)

La compromissione totale della macchina è stata ottenuta concatenando diverse vulnerabilità:

1. **Information Gathering & Enumerazione SMB**: La scansione di rete ha identificato una condivisione Samba aperta senza autenticazione (`public`), contenente una nota per gli amministratori che rivelava un percorso interno non indicizzato del server web (`/myrouterpanel`).
2. **Bypass dei Filtri e Command Injection**: L'endpoint `/myrouterpanel` ospitava un tool di diagnostica di rete privo di autenticazione. Sebbene filtrasse caratteri di chaining come `;`, `&` e `|`, la mancata neutralizzazione del carattere di fine riga ha permesso di aggirare il filtro con un payload con newline URL-encoded (`%0A`), stabilendo una reverse shell con i privilegi dell'utente di servizio `www-data`.
3. **Privilege Escalation Locale (utente `athena`)**: Tramite monitoraggio con `pspy64`, è stata individuata un'attività schedulata (cron) eseguita periodicamente dall'utente `athena` tramite lo script `/usr/share/backup/backup.sh`. A causa di permessi errati sul file (CWE-276, file scrivibile da `www-data`), è stato possibile inserire codice arbitrario ed elevare i privilegi all'utente `athena`, recuperando la prima flag.
4. **Analisi del Rootkit ed Escalation a `root`**: Dall'ispezione della configurazione `sudo -l`, l'utente `athena` disponeva del permesso di caricare un modulo kernel (`insmod venom.ko`) senza password. La decompilazione del modulo con Ghidra ha evidenziato una tecnica di hooking sulla syscall `sys_kill`: alla ricezione del segnale 57 (`kill -57 <PID>`), veniva invocata la funzione interna `give_root()`. L'invio del segnale ha consentito di acquisire una shell interattiva con massimi privilegi (`root`), con conseguente recupero della flag finale.

---

## Riepilogo delle Vulnerabilità Rilevate

| Vulnerabilità | Gravità | Punteggio CVSS | Classificazione CWE |
| :--- | :--- | :--- | :--- |
| **Vulnerability Chain Completa** | Critica | **9.3** | - |
| Modulo rootkit non sicuro eseguibile da Athena (`venom.ko`) | Alta | 8.5 | CWE-912: Hidden Functionality |
| Condivisione SMB accessibile senza autenticazione | Media | 6.9 | CWE-862: Missing Authorization |
| Accesso non autenticato alla pagina `/myrouterpanel` | Media | 6.9 | CWE-862: Missing Authorization |
| OS Command Injection su pannello diagnostico | Media | 6.3 | CWE-78: OS Command Injection |
| Permessi di scrittura errati su `/usr/share/backup/backup.sh` | Media | 4.8 | CWE-276: Incorrect Default Permissions |
| Esposizione dell'Uptime tramite TCP Timestamps | Bassa | 2.6 | CWE-200: Exposure of Sensitive Information |
| Algoritmi MAC deboli supportati su servizio OpenSSH | Bassa | 2.6 | CWE-327: Risky Cryptographic Algorithm |
| Risposte a richieste ICMP Timestamp Reply | Bassa | 2.1 | CWE-200: Exposure of Sensitive Information |

---

## Misure di Mitigazione e Hardening Proposte

Per la messa in sicurezza dell'ambiente sono state formulate raccomandazioni sia applicative che infrastrutturali:

- **Autenticazione e Controllo degli Accessi**: Configurare credenziali sicure su tutte le share Samba ed escludere l'accesso anonimo; proteggere gli endpoint amministrativi web mediante autenticazione forte e restrizioni sul web server (es. direttive `.htaccess` o reverse proxy dedicati).
- **Validazione Rigorosa degli Input**: Sostituire le logiche di blacklist con whitelist stringenti (es. espressioni regolari per indirizzi IPv4/IPv6 validi) ed evitare la concatenazione di comandi a livello di sistema operativo, adottando le funzioni di escaping fornite dal linguaggio o chiamate API sicure.
- **Principio del Minimo Privilegio (Least Privilege)**: Correggere i permessi POSIX sui file di sistema e di backup, eliminando i permessi di scrittura per utenti non privilegiati (`www-data`), e implementare controlli di integrità sugli script eseguiti periodicamente.
- **Hardening del Kernel e Restrizioni `sudo`**: Rimuovere moduli kernel malevoli o non approvati, revocare l'esecuzione di comandi critici come `insmod` agli account utente non amministrativi e verificare periodicamente i binari con permessi elevati.
- **Hardening di Rete e Crittografia**: Disabilitare risposte a richieste ICMP e TCP Timestamp agendo sui parametri di kernel (`sysctl`), e aggiornare la configurazione di OpenSSH eliminando algoritmi MAC non sicuri a favore di varianti robuste come SHA-256 e SHA-512.

---

## Struttura della Repository

```text
.
├── Report_1000044165_Petronio_Petronio_Davide.pdf   # Report tecnico dettagliato di VAPT
└── README.md                                       # Sintesi esecutiva e tecnica del progetto
```