## Verkeerd Geconfigureerde SMB-Shares

Windows is wereldwijd het meest gebruikte besturingssysteem. Het is geliefd om zijn gebruiksgemak en toegankelijke bestandsopslag. Bedrijven implementeren Windows-servers en -computers om hun netwerken te ondersteunen. Hoewel de interface eenvoudig is, blijkt het beveiligen van Windows complex. Voor onervaren beheerders is het een uitdaging om kwetsbaarheden te herkennen en te verhelpen.

In dit scenario onderzoeken we een onjuist geconfigureerde SMB-share. Er zijn twee methoden om toegang te krijgen: een eenvoudige en een geavanceerde methode. De eerste is rechtlijnig; de tweede vereist gespecialiseerde software en meer expertise. Aangezien de geavanceerde methode minder vaak voorkomt en complexer is, vereist deze zorgvuldige overweging.

---

**Stap 1: Controleer of de host actief is**

```bash
# Huidige IP-adres controleren
ip a s

# Test of de doelhost actief is
ping -c3 {target ip}
```

### MITRE ATT&CK

- **Tactic**: **Reconnaissance (TA0043)**
- **Technique**: **ICMP Scan (T1005)**

<aside>
💡

Gebruik `ping6` voor IPv6-netwerken.

</aside>

**Ping-proces in het OSI-model (zender naar ontvanger)**

````bash
Applicatielaag (L7) → Netwerklaag (L3) → Datalinklaag (L2) → Fysieke laag (L1)
(Ping Commando)       (ICMP Echo Request)   (Ethernet Frame)    (Signalen via Netwerk)
````

**Ping-proces in het OSI-model (ontvanger naar zender)**

```bash
Fysieke laag (L1) → Datalinklaag (L2) → Netwerklaag (L3) → Datalinklaag (L2) → Fysieke laag (L1)
(Ontvangt Request)   (Verwerkt Frame)     (ICMP Request)        (Echo Reply)      (Signalen Terug)
```

**Ping-resultaat in het OSI-model**

```bash
Fysieke laag (L1) → Datalinklaag (L2) → Netwerklaag (L3) → Applicatielaag (L7)
(Ontvangt Reply)    (Ethernet Frame)    (ICMP Echo Reply)   (Resultaat in Terminal)
```

---

## Enumeration

Een standaard nmap-scan wordt uitgevoerd om de host te identificeren. Deze scan gebruikt verschillende methoden:

- ARP-verzoeken voor lokale netwerken
- ICMP Echo Requests (gewone pings)
- TCP-verbindingspogingen
- UDP-pings

Door deze methoden te combineren, kunnen we actieve hosts detecteren, zelfs als firewalls bepaalde pings blokkeren. Dit helpt ook bij het identificeren van netwerkbeveiligingsmaatregelen die verkeer beperken.

```bash
# Ping de doelhost en controleer op actieve firewalls
nmap -sn {target}
```

### MITRE ATT&CK

- **Tactic**: **Discovery (TA0007)**
- **Technique**: **Network Scanning (T1046)**

<aside>
🔥

De scan toont aan dat er een firewall actief is.

</aside>

Firewalls blokkeren vaak scans en verzoeken van tools zoals nmap. ICMP wordt vaak gebruikt om een host te pingen en wordt ingezet aan het begin van een nmap-scan. De ***-Pn*** parameter zorgt ervoor dat de host discovery-fase wordt overgeslagen en alle hosts als actief worden beschouwd. Vervolgens worden pogingen gedaan om poorten te scannen en versienummers van services te verzamelen.

```bash
-Pn : Sla host-discovery over (geen ICMP pings); beschouw alle hosts als actief.
-sC : Voer alle standaard scripts uit, met --script=default, en enumerate services.

* Zie [default scripts](https://nmap.org/nsedoc/categories/default.html)
```

---

**Uitbreiding van de attack surface zonder ICMP**

```bash
sudo nmap -sC -Pn {target ip}
```

### MITRE ATT&CK

- **Tactic**: **Discovery (TA0007)**
- **Technique**: **Service Scanning (T1046)**

### Uitleg van de -sC optie

- **Service version detection**: Versies van de services op de doelhost.
- **Vulnerability scanning**: Controleer op kwetsbaarheden in services.
- **OS detection**: Bepaal het besturingssysteem van de doelhost.

<aside>
📝

De resultaten tonen dat poort 445 (SMB) openstaat. Documenteer en analyseer open poorten zorgvuldig voor verdere acties.

</aside>

### **Poort 135: RPC (Remote Procedure Call)**

**Functie**: Communicatie tussen Windows-toepassingen. Actief in **svchost.exe**.

**Voordelen**: Essentieel voor Windows-applicaties en COM/DCOM-technologieën.

**Nadelen**: Mogelijk doelwit voor aanvallen; **svchost.exe** mag niet worden beëindigd.

### **Poort 139: NetBIOS**

**Functie**: Communicatie voor bestands- en printerdeling binnen een LAN.

**Voordelen**: Eenvoudig netwerkverkeer. **SMB lokaal mogelijk**.

**Nadelen**: Verouderd, kwetsbaar voor aanvallen, vaak niet nodig in moderne netwerken.



### **Poort 445: SMB (Server Message Block)**

**Functie**: Netwerkbestandsdeling via **TCP/IP**, ook over internet.

**Voordelen**: Ondersteunt bestandsdeling met IP-adressen.

**Nadelen**: Doelwit voor ransomware-aanvallen; netwerkprestaties kunnen lijden.



SMB kan waardevolle informatie onthullen. Dit kan worden onderzocht met de **smbclient** tool, standaard geïnstalleerd op Parrot OS.

Als **smbclient** ontbreekt, installeer met:

```bash
sudo apt install smbclient
```

**Belangrijke smbclient-opties**:

```bash
-L : Lijst beschikbare shares.
-U : Inloggegevens.
```

### MITRE ATT&CK

- **Tactic**: **Discovery (TA0007)**
- **Technique**: **Network Share Discovery (T1135)**

```bash
smbclient -L {target_IP} -U Administrator
```


---

**Beveiligingsadvies**:

- SMB-toegang met gevoelige data moet correct worden beheerd.
- Toegang beperken tot enkel noodzakelijke gebruikers.
- Gebruik groepsbeleidsinstellingen om SMB-authenticatie en toegang te beheersen.

**Het beschermen van je netwerk vereist continu toezicht en het afschermen van kwetsbare diensten.**
