## De methode

**Stap 1 – PowerShell starten**
```
powershell
```

**Stap 2 – Wachtwoord typen (met sterretjes)**
```powershell
$ww = Read-Host Wachtwoord -AsSecureString
```
Elk teken dat je typt verschijnt als `*`, en het wachtwoord wordt beveiligd opgeslagen. Wil je een typfout helemaal uitsluiten, typ het dan nog een keer en vergelijk:
```powershell
$ww2 = Read-Host Nogmaals -AsSecureString
[Net.NetworkCredential]::new($null,$ww).Password -ceq [Net.NetworkCredential]::new($null,$ww2).Password
```
`True` betekent dat ze gelijk zijn. Bij `False` doe je stap 2 opnieuw. Met pijltje-omhoog haal je eerdere regels terug.

**Stap 3 – Account aanmaken en beheerder maken**

Vervang `Jan` door je eigen naam:
```powershell
New-LocalUser -Name Jan -Password $ww -PasswordNeverExpires
Add-LocalGroupMember -SID S-1-5-32-544 -Member Jan
Add-LocalGroupMember -SID S-1-5-32-545 -Member Jan
net user Jan
```
- **`S-1-5-32-544`** is de groep Administrators. De groepsnaam verschilt per taal (in het Duits heet hij bijvoorbeeld Administratoren), maar dit nummer (de SID) niet. Via de SID werkt het in elke taal, zelfs als de groep hernoemd is.
- **`S-1-5-32-545`** is de groep Users/Gebruikers, net als bij een normaal account. Zegt Windows dat Jan daar al lid van is, dan is dat prima.
- **`-PasswordNeverExpires`** voorkomt dat Windows je na 42 dagen een nieuw wachtwoord laat kiezen. Die termijn geldt alleen voor lokale accounts zonder "Wachtwoord verloopt nooit".
- **Regels voor de naam:** maximaal 20 tekens, zonder " / \ [ ] : ; | = , + * ? < > @. Het wachtwoord mag maximaal 127 tekens hebben. Kies liefst een naam zonder spaties, dan heb je geen aanhalingstekens nodig.
- **`net user Jan`** is je controle. Administrators moet bij de lokale groepen staan, en het wachtwoord moet nooit verlopen.
- **Lukt PowerShell niet?** Dan kan het ook met `net user Jan * /add` en `net localgroup Administrators Jan /add`. Maar dan zie je bij het wachtwoord helemaal niets (geen sterretjes), en moet je de groepsnaam in de taal van Windows typen.

**Stap 4 – De installatie afronden**
```
exit
cd /d %windir%\System32\oobe
msoobe && shutdown -r
```
- Met `exit` ga je terug naar CMD. Dat is nodig, want `&&` werkt niet in Windows PowerShell.
- Het herstarten kan even duren: `shutdown -r` wacht standaard zo'n 30 seconden.
- Gebeurt er na een paar minuten niets, typ dan `shutdown /r /t 0`.

**Stap 5 – Na de herstart**
- Je krijgt eerst de melding "De gebruikersnaam of het wachtwoord is onjuist". Die kun je negeren. Klik op OK.
- Op het aanmeldscherm staan je nieuwe account en defaultuser0. Kies je eigen account. Staat het er niet bij, kies dan "Andere gebruiker".

## Met netwerkadapter of internet
De commando's blijven precies hetzelfde. De verschillen:
- **Adapter aanwezig, maar nog niet verbonden** (je ziet een Wi-Fi-lijst, of de kabel zit er niet in): maak geen verbinding en volg gewoon de stappen hierboven.
- **Wel internet (kabel of Wi-Fi):** OOBE zoekt dan vaak eerst naar updates en herstart misschien. Daarna kom je op het aanmeldscherm voor het Microsoft-account in plaats van op het netwerkscherm. Wacht tot er een scherm staat dat om invoer vraagt, dus geen draaiend bolletje, en druk dan op Shift+F10. Ook vanaf dat aanmeldscherm werkt de methode.
- **Extra zekerheid (optioneel):** ga eerst offline. PC Games Hardware raadt bij deze methode aan om pas na de herstart weer internet te verbinden. Trek de kabel eruit, zet de vliegtuigstand aan, of zet in VMware het vinkje "Connected" uit. In PowerShell zet je alle adapters tegelijk uit, in elke taal:
  ```powershell
  Get-NetAdapter | Disable-NetAdapter -Confirm:$false
  ```
  Zet ze na je eerste aanmelding weer aan, in Terminal (Beheerder). Vergeet je dit, dan heb je geen netwerk.
  ```powershell
  Get-NetAdapter | Enable-NetAdapter -Confirm:$false
  ```

## Werkt dit op elk scherm?
- **Ja, in het hele OOBE-gedeelte:** vanaf "Is dit het juiste land of de juiste regio?" tot en met het aanmeldscherm van het Microsoft-account.
- **Nee, in het eerste deel:** dat zijn de "Windows Setup"-schermen vanaf USB of ISO (taal, productcode, schijf). Shift+F10 opent daar ook een venster, maar Windows is dan nog niet geïnstalleerd, dus je kunt nog geen gebruiker aanmaken.
- **Tip:** kies eerst regio en toetsenbord, zoals jij al gedaan hebt. Typ je wachtwoord met dezelfde toetsenbordindeling die je straks bij het aanmelden gebruikt. Bij "Verenigde Staten (internationaal)" zijn ' en " dode toetsen: typ er een spatie achter om het teken zelf te krijgen.
- **Uitzondering:** fabrikanten en bedrijven kunnen Shift+F10 bewust uitschakelen met een bestand DisableCMDRequest.TAG. Dan heb je plan B nodig (verderop).

## Na de eerste aanmelding
- **Tijdelijk account verwijderen:** open Terminal (Beheerder) via Win+X en typ `net user defaultuser0 /delete`. Dat account is alleen bedoeld voor de installatie en hoort daarna weg. Anders ruimt een geplande taak het na 10 dagen op.
- **Privacy:** Microsoft waarschuwt dat zulke omwegen setupschermen kunnen overslaan. Kijk daarom zelf even bij Instellingen → Privacy en beveiliging.
- **Pc-naam:** die is nu willekeurig. Wijzigen kan via Instellingen → Systeem → Info.
- **Wachtwoord vergeten:** zonder beveiligingsvragen is er geen herstelroute. Bewaar je wachtwoord goed, of maak een wachtwoordherstelschijf via Configuratiescherm → Gebruikersaccounts.

## Op welke versies werkt dit, en hoe lang nog?
- **Het PowerShell-deel** (`Read-Host`, `New-LocalUser`, `Add-LocalGroupMember`) zit standaard in elke Windows 11-versie, in elke taal, op Home en Pro.
- **Het `msoobe`-deel** is een onofficiële truc. Het pad is wel in elke taal hetzelfde. De methode is ook in september 2026 nog beschreven en in oktober 2026 gemeld op 25H2 (build 26200.8037). 26H2 (build 26300) deelt via een enablement package dezelfde basis met 24H2 en 25H2.
- **Tot welke build dit werkt, kan niemand zeggen.** Microsoft sluit deze routes stap voor stap:
  - In de Insider-builds 26220.6772 en 26120.6772 zijn bypassnro en ms-cxh:localonly al verwijderd.
  - In oktober 2026 is de "Meer informatie"-truc in de nieuwste 26H2-ISO dichtgezet.
  - Op de gewone 26H2 (26300.9457) werkt ms-cxh:localonly nog wel, met de drie beveiligingsvragen. Zulke blokkades komen dus niet meteen in de gewone versie, maar dat kan op elk moment gebeuren. `msoobe` kan de volgende zijn.

## Plan B dat waarschijnlijk blijft werken: een antwoordbestand
Dit is de enige route die Microsoft zelf documenteert:
- Maakt het antwoordbestand minstens één account aan, dan slaat elke Windows-versie de accountstap in OOBE over.
- Groepsnamen zoals Administrators schrijf je in het Engels, en Windows zet ze automatisch om naar de juiste taal.

Bedrijven en fabrikanten zijn hiervan afhankelijk, dus deze route verdwijnt minder snel.

- **Echte pc's:** gebruik Rufus. Vink bij het schrijven naar USB aan dat de eis voor een online Microsoft-account vervalt, en dat er een lokaal account wordt aangemaakt. Rufus maakt dat account met een leeg wachtwoord. Na de eerste herstart vraagt Windows je om een nieuw wachtwoord. In dat scherm zie je een stip per teken. Borncity bevestigde in oktober 2026 dat dit op 26H2 werkt, en raadt aan de pc tijdens het aanmaken van het account offline te houden.
- **VM's:** Windows Setup pakt automatisch een Autounattend.xml op uit de hoofdmap van een cd/dvd of USB-stick. Zet zo'n bestand dus op een tweede virtuele cd. Je kunt het maken met de Schneegans-generator, die een lokaal admin-account instelt en de online-schermen overslaat.

## Het script voor echte pc's
`maak-lokale-admin.ps1` doet stap 2 en 3 in één keer:
- vraagt je naam en controleert die;
- laat je het wachtwoord 2x typen, met sterretjes, en toont daarna het aantal tekens;
- maakt het account aan via de SID's, met "wachtwoord verloopt nooit";
- controleert of het account echt beheerder is.

Zet het op een USB-stick en typ in het Shift+F10-venster:
```
powershell -ExecutionPolicy Bypass -File E:\maak-lokale-admin.ps1
```
Vervang `E:` door de letter van je stick. Weet je die niet, probeer dan `dir D:\`, `dir E:\` enzovoort. Daarna doe je stap 4; het script laat die commando's ook zien.
