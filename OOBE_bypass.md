Ja, dit kan, en je hebt er geen enkel OOBE-scherm voor nodig. Daardoor is de methode ook minder kwetsbaar dan `ms-cxh:localonly`, want Microsoft haalt vooral die schermen en commando's weg. Ik kan het zelf niet uitvoeren, dus test het eerst in je VM.

## Kort antwoord

- **Aanpak:** je maakt het account zelf aan in het Shift+F10-venster, via PowerShell zodat je sterretjes ziet. Je zet het via de SID in Administrators en laat OOBE daarna zichzelf overslaan. De drie beveiligingsvragen horen bij het OOBE-scherm en verschijnen pas als je daar een wachtwoord instelt. Een account dat je buiten dat scherm aanmaakt, heeft ze niet.
- **Jouw build:** 26300.9974 is de update van 10 november 2026 (KB5132061), en sinds 19 november bestaat er al 26300.10075. Voor 26H2 meldt Tech Based dat het handmatig aanmaken van een account met `net user` gevolgd door `msoobe` nog betrouwbaar werkt. De grondigere variant hieronder (`SkipMachineOOBE`) staat in een GitHub-gist voor 25H2, met bevestigingen van november 2025 en mei 2026. Een test op precies 26300.9974 vond ik niet, maar ook geen melding dat het geblokkeerd is. Zie het dus als "waarschijnlijk".
- **Tot welke versie:** dat kan niemand beloven. Microsoft kondigde in oktober 2025 aan de bekende mechanismen voor lokale accounts uit OOBE te halen. `SkipMachineOOBE` is door Microsoft zelf als verouderd en alleen voor testomgevingen bestempeld, en Microsoft raadt het af voor het automatiseren van OOBE. Volgens WindowsForum staat er in Microsofts OOBE-plannen voor eind 2026 geen lokale-accountoptie. Dit is dus een noodgreep die bij een nieuwe build kan stoppen.

## Stappenplan

**Voorbereiding**
1. Maak in VMware een snapshot.
2. Druk op Shift+F10 (op sommige laptops Shift+Fn+F10) en klik daarna in het venster. Zonder klik gaat je invoer soms naar het scherm erachter.
3. Typ `echo \ $ ' : -` en controleer of je precies die tekens ziet. Het venster gebruikt de toetsenbordindeling van het OOBE-scherm.

**Stap 1:** start PowerShell in het CMD-venster.
```
powershell
```

**Stap 2: account met sterretjes** (typ regel voor regel)
```powershell
$u = Read-Host 'Gebruikersnaam'
$p = Read-Host 'Wachtwoord' -AsSecureString
$q = Read-Host 'Herhaal wachtwoord' -AsSecureString
if ([Net.NetworkCredential]::new('',$p).Password -cne [Net.NetworkCredential]::new('',$q).Password) { throw 'Wachtwoorden verschillen' }
New-LocalUser -Name $u -Password $p -PasswordNeverExpires -AccountNeverExpires
'S-1-5-32-544','S-1-5-32-545' | % { Add-LocalGroupMember -SID $_ -Member $u }
net user $u
```
- `Read-Host -AsSecureString` toont voor elk getypt teken een `*`. `net user NAAM *` toont tijdens het typen niets, vandaar PowerShell.
- Bij "Wachtwoorden verschillen" herhaal je de regels met `$p`, `$q` en de `if`.
- `S-1-5-32-544` is Administrators en `S-1-5-32-545` is Users. De groepsnaam verschilt per taal, de SID niet.
- `net user $u` moet bij de lokale groepslidmaatschappen beide groepen tonen.
- Laat `-PasswordNeverExpires` weg als je wachtwoordverloop wilt.

**Stap 3: OOBE laten overslaan** (zelfde venster)
```powershell
$o = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\OOBE'
Get-ItemProperty $o
$t = (Get-ItemProperty $o).DefaultAccountSAMName
$t
```
Als `$t` `defaultuser0` (of iets met `defaultuser…`) toont, ga je door. Dat tijdelijke account heeft lokale beheerdersrechten en is ook het account van je Shift+F10-venster. Staat er iets anders of niets, stop dan en lees "Als het misgaat".
```powershell
'DefaultAccountAction','DefaultAccountSAMName','DefaultAccountSID','LaunchUserOOBE' | % { Remove-ItemProperty $o -Name $_ -ErrorAction SilentlyContinue }
New-ItemProperty $o -Name SkipMachineOOBE -Value 1 -PropertyType DWord -Force
if ($t -like 'defaultuser*') { Remove-LocalUser -Name $t }
shutdown /r /t 0
```
Dit is het recept uit de gist, maar dan als commando's in plaats van met regedit. De overbodige `/active:yes` en `/expires:never` uit andere handleidingen zijn standaardwaarden.

**Stap 4: na de herstart** zie je geen OOBE-pagina's meer. Eenmalig kan er een melding over een onjuist wachtwoord verschijnen; klik op OK en log in met je eigen account.

*Tip voor echte machines:* zet de regels van stap 2 en 3 in een `.ps1`-bestand op een USB-stick. Start dat vanuit het Shift+F10-venster met `powershell -ExecutionPolicy Bypass -File E:\naam.ps1` (zoek de stationsletter met `dir D:`, `dir E:`).

## Per netwerksituatie

De commando's zijn overal gelijk en je hoeft niets los te koppelen. Bij de oude BypassNRO-route moest dat wel, inclusief `ipconfig /release`. Wat verschilt:

- **Geen adapter of niet verbonden** (je VM, of een laptop zonder verbinding): niets bijzonders.
- **Bekabeld verbonden:** OOBE slaat de netwerkpagina vaak over, maar Shift+F10 werkt dan ook. Microsoft levert OOBE-updates die tijdens OOBE worden geïnstalleerd zodra er internet is. Begin daarom liefst op de allereerste pagina, of haal de kabel eruit tot na de herstart. Dat laatste is geen vereiste, maar het voorkomt dat OOBE zichzelf wijzigt terwijl je bezig bent.
- **Wi-Fi:** de Wi-Fi-pagina's worden overgeslagen. Had je al verbonden, dan blijft het profiel meestal bewaard. Anders verbind je na het inloggen via het netwerkpictogram.
- **Adapter zonder stuurprogramma:** OOBE toont dan alleen een knop om stuurprogramma's te laden. Deze methode omzeilt dat, het stuurprogramma installeer je na het inloggen.

## Elke taal en elk scherm

- Er is niets taalafhankelijks: SID's in plaats van groepsnamen, en de naam van het tijdelijke account komt uit het register. Alleen de toetsenbordindeling kan roet in het eten gooien.
- Het werkt op elke OOBE-pagina, van de regiokeuze tot het laatste scherm. Het werkt niet in de Windows Setup-fase vóór de eerste herstart (taal kiezen, partitie, bestanden kopiëren). Dan is een `autounattend.xml` op de installatiemedia de enige route.

## Als het misgaat

- **Shift+F10 reageert niet:** probeer Shift+Fn+F10 en klik in het venster. Eén bron meldt dat Windows 11 in S-modus het venster blokkeert.
- **Lichtere variant:** doe stap 2 en daarna dit. Zo wordt het voor 26H2 beschreven.
  ```
  exit
  cd /d %windir%\system32\oobe
  msoobe && shutdown -r
  ```
  `defaultuser0` blijft dan op het inlogscherm staan, dus verwijder het na het inloggen.
- **Installatiemedia voorbereiden:** gHacks noemt een Rufus-USB of een autounattend.xml de betrouwbaarste route in 2026. Bij Rufus werkte het alleen als zowel "online account verwijderen" als "lokaal account met naam" aanstond. Het wachtwoord staat in een answer file leesbaar, dus bewaar die niet.
- **Korte omweg:** `start ms-cxh:localonly` met een leeg wachtwoord geeft geen vragen, en je stelt het wachtwoord daarna in. Volgens gHacks is dat commando op nieuwere builds geblokkeerd, bij jou werkt het nog.
- **Mijd** alles-in-één-scripts die bestanden uit `System32\oobe` verwijderen en taken uitschakelen.
- Op een echte machine is opnieuw installeren de terugweg. Test daarom eerst in de VM.

## Na de eerste aanmelding

- Controleer taal, regio en tijdzone, en de privacy-instellingen. Microsoft waarschuwt dat instellingen uit Windows Welcome, zoals taal en tijdzone, bij overslaan niet worden geconfigureerd.
- Kijk met `manage-bde -status` of BitLocker aanstaat. Zo ja, bewaar dan zelf de herstelsleutel (`manage-bde -protectors -get C:`), want er is geen Microsoft-account dat een kopie bewaart.
- De map `C:\Users\defaultuser0` mag weg.

Laat gerust weten wat je in de VM ziet na stap 4, dan stemmen we de varianten daarop af.
