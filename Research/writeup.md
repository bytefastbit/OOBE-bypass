Scripts getest: methode B werkt in de simulatie (geldige XML, wachtwoord correct versleuteld, lussen en ARM64 kloppen). Nu test ik nog methode A en de korte typ-versie voor als er geen netwerk is.

Ik ben klaar met uitzoeken en testen. De nieuwste bronnen die ik vond zijn van half oktober 2026. De scripts heb ik gecontroleerd op syntaxis en logica; de twee scripts zijn ook meegestuurd als bestand (`oobe-lokaal.ps1` en `oobe-antwoordbestand.ps1`).

## Kort en eerlijk

**Ja, dit kan op 26H2: zonder netwerk, zonder beveiligingsvragen, met een eigen beheerdersaccount.** De route die het betrouwbaarst lijkt (methode A) is het account zelf aanmaken vanuit de Shift+F10-console en OOBE daarna laten afronden. Een blog van half oktober 2026 noemt precies deze aanpak, met losse commando's op de aanmeldpagina, nog steeds heel betrouwbaar op 26H2. Een ElevenForum-gebruiker gebruikte dezelfde commando's op 25H2 (build 26200.8037).

Wat ik niet kan beloven:

- **Jouw exacte build (26300.9974) heb ik niet getest.** Mijn scripts heb ik alleen in een simulatie gecontroleerd (PowerShell 7 op Linux, met nagebootste Windows-commando's), niet in een echte OOBE. Maak dus eerst een VMware-snapshot in OOBE en test daar.
- **Er is geen methode die voor elke build blijft werken.** Microsoft zegt bekende routes naar een lokaal account in setup te verwijderen, onder meer omdat die setup-schermen overslaan. Een overzicht van augustus 2026 concludeert dat geen enkel commando meer op elk Windows 11-image werkt. Daarom krijg je een hoofdmethode, een back-up en een noodroute.
- **Pastebin werkt alleen mét internet.** In je VM zonder netwerkadapter kun je niets downloaden. Daar typ je de korte versie (stap 1).

De drie lagen:

- **A (hoofdmethode, geen netwerk nodig):** account zelf maken, `msoobe` uitvoeren, herstarten.
- **B (back-up, geen USB):** een antwoordbestand naar `C:\Windows\Panther` schrijven en `sysprep /oobe` starten. De maker van een gelijkaardig script meldt dat dit op 5 oktober 2026 nog werkte op 26H2 build 26300.9457, en op 25H2 op 19 september. B is te lang om te typen, dus je hebt netwerk of een ISO nodig.
- **C (noodroute):** een `autounattend.xml` op installatiemedia.

## 1. Nu, in je VM (geen netwerk): typen

1. In het CMD-venster dat al open staat: typ `powershell` en Enter.
2. Typ deze vier regels letterlijk, elke regel met Enter:

```powershell
$n=Read-Host Naam
do{$a=Read-Host Wachtwoord -AsSecureString;$b=Read-Host Herhaal -AsSecureString}until($a.Length -and [Net.NetworkCredential]::new('',$a).Password -ceq [Net.NetworkCredential]::new('',$b).Password)
New-LocalUser $n -Password $a -PasswordNeverExpires|Out-Null
Add-LocalGroupMember -SID S-1-5-32-544 -Member $n
```

3. Typ `exit` (terug naar CMD) en daarna:

```cmd
%windir%\system32\oobe\msoobe.exe && shutdown /r /t 3
```

Wat dit doet:

- **Asterisken:** `Read-Host -AsSecureString` toont een * voor elk teken dat je typt.
- **Herhaling:** je moet het wachtwoord twee keer invoeren. Dat is bewust: dit wordt je enige beheerder, dus een typfout betekent buitengesloten zijn.
- **Beheerdersgroep:** de groep wordt via SID `S-1-5-32-544` gevonden en niet via de naam. Die naam verschilt per taal, bijvoorbeeld in het Frans. `Add-LocalGroupMember` ondersteunt hiervoor de parameter `-SID`.
- **Rechten:** het account is gewoon lid van Administrators, met UAC zoals normaal. Het ingebouwde Administrator-account blijft uit.
- **Geen beveiligingsvragen:** die krijg je alleen als je op de OOBE-accountpagina een wachtwoord instelt. Die pagina komt bij deze route niet voor.

Wat je daarna ziet:

- Bij de herstart verschijnt een foutmelding dat gebruikersnaam of wachtwoord onjuist is. Die mag je negeren.
- Het inlogscherm toont je account én `defaultuser0`. Kies je eigen account.
- Na het inloggen kun je `defaultuser0` uitschakelen om hem van het inlogscherm te halen.

Omdat Microsoft waarschuwt dat dit soort routes schermen overslaat, controleer je na de eerste login even Instellingen → Privacy en Windows Update.

## 2. Mét internet: dezelfde methode via Pastebin

Plak onderstaand script (= `oobe-lokaal.ps1`) in een nieuwe paste:

```powershell
# oobe-lokaal.ps1 - methode A: lokaal beheerdersaccount maken tijdens OOBE
# Houd dit bestand puur ASCII: Windows PowerShell 5.1 leest bestanden zonder BOM als ANSI.
$ErrorActionPreference = 'Stop'

function ConvertTo-Plain([securestring]$s) {
    $b = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($s)
    try { [Runtime.InteropServices.Marshal]::PtrToStringBSTR($b) }
    finally { [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($b) }
}

# Alleen in OOBE doorgaan: msoobe.exe op een werkende pc is niet de bedoeling
$setup = (Get-ItemProperty 'HKLM:\SYSTEM\Setup' -ErrorAction SilentlyContinue).SystemSetupInProgress
if (($env:USERNAME -ne 'defaultuser0') -and ($setup -ne 1)) {
    Write-Host "Dit lijkt geen OOBE (gebruiker: $env:USERNAME)." -ForegroundColor Red
    if ((Read-Host 'Typ CONTINUE (hoofdletters) om toch door te gaan') -cne 'CONTINUE') { return }
}

# 1) Gebruikersnaam
do {
    $naam = (Read-Host 'Gebruikersnaam').Trim()
    $fout = ($naam.Length -lt 1) -or ($naam.Length -gt 20) -or ($naam -match '[\\/\[\]:|<>+=;,?*%@"]') -or ($naam -ieq 'NONE')
    if ($fout) { Write-Host 'Ongeldig: 1-20 tekens en geen \ / [ ] : | < > + = ; , ? * % @ "' -ForegroundColor Yellow }
} while ($fout)

# 2) Wachtwoord, twee keer. Read-Host -AsSecureString toont een * per teken.
do {
    $p1 = Read-Host 'Wachtwoord' -AsSecureString
    $p2 = Read-Host 'Herhaal wachtwoord' -AsSecureString
    $ok = ($p1.Length -gt 0) -and ((ConvertTo-Plain $p1) -ceq (ConvertTo-Plain $p2))
    if (-not $ok) { Write-Host 'Leeg of niet gelijk, probeer opnieuw.' -ForegroundColor Yellow }
} while (-not $ok)
Write-Host ('Wachtwoord van {0} tekens aangenomen.' -f $p1.Length)

# 3) Account maken en lid maken van Administrators (via SID, dus taalonafhankelijk)
if (Get-LocalUser -Name $naam -ErrorAction SilentlyContinue) {
    Set-LocalUser -Name $naam -Password $p1
} else {
    New-LocalUser -Name $naam -Password $p1 -PasswordNeverExpires -AccountNeverExpires | Out-Null
}
try { Add-LocalGroupMember -SID 'S-1-5-32-544' -Member $naam } catch { }

$lid = $null; $kon = $true
try { $lid = Get-LocalGroupMember -SID 'S-1-5-32-544' | Where-Object { $_.Name -like "*\$naam" } } catch { $kon = $false }
if ($kon -and -not $lid) { throw "$naam is geen lid van Administrators geworden. Stop." }
if ($kon) { Write-Host "OK: $naam is beheerder." -ForegroundColor Green }
else { Write-Host 'Lidmaatschap kon niet gecontroleerd worden; controleer het na afloop zelf.' -ForegroundColor Yellow }

# 4) OOBE afronden en herstarten
$msoobe = Join-Path $env:SystemRoot 'System32\oobe\msoobe.exe'
if (-not (Test-Path $msoobe)) { throw 'msoobe.exe ontbreekt: gebruik methode B.' }
Write-Host 'Na de herstart: een foutmelding over gebruikersnaam/wachtwoord mag je negeren. Kies je account.'
Read-Host 'Enter = OOBE afronden en herstarten (Ctrl+C = stoppen)' | Out-Null
Start-Process -FilePath $msoobe -Wait
shutdown.exe /r /t 3
```

Zo gebruik je het:

1. **Paste instellen:** zet de paste op *Unlisted* (niet *Private*) en maak er een korte raw-URL van, `https://pastebin.com/raw/JOUWID`. Zet nooit een wachtwoord in de paste; het script vraagt er zelf om.
2. **Downloaden en starten:** druk in OOBE op Shift+F10 en typ in CMD:

```cmd
curl.exe -L -o %TEMP%\a.ps1 https://pastebin.com/raw/JOUWID
type %TEMP%\a.ps1
powershell -NoProfile -ExecutionPolicy Bypass -File %TEMP%\a.ps1
```

   De `type`-regel laat het script zien, zodat je ziet dat je niet per ongeluk andermans paste hebt gedownload. Je script draait namelijk als beheerder.
3. **Of met Invoke-WebRequest** (jouw voorkeur):

```cmd
powershell -NoProfile -Command "Invoke-WebRequest -UseBasicParsing -Uri https://pastebin.com/raw/JOUWID -OutFile $env:TEMP\a.ps1"
```

   Gebruik `-UseBasicParsing`. Sinds de update van 9 december 2025 vraagt Invoke-WebRequest in Windows PowerShell 5.1 anders een beveiligingsbevestiging. Microsoft raadt `-UseBasicParsing` aan om die prompt te vermijden.
4. **Alleen ASCII:** voeg geen ë, é of slimme aanhalingstekens toe aan het script. Een bestand zonder BOM leest Windows PowerShell 5.1 als ANSI, waardoor zulke tekens kunnen mislukken.
5. **Pastebin blokkeert:** krijg je HTML of een foutcode terug, dan weigert Pastebin het verzoek. Gebruik dan een andere host, zoals een Gist-raw-link.

Per situatie:

- **Geen adapter of niet verbonden:** typ de korte versie uit stap 1. Een VM kan ook een ISO met de scripts als tweede virtuele dvd gebruiken.
- **Kabel met internet:** Shift+F10 op een willekeurige OOBE-pagina en dan Pastebin. Gebruikers melden Shift+F10 bij de regiopagina, bij de Microsoft-aanmeldpagina en bij een latere pagina.
- **Wi-Fi:** verbind eerst via de netwerkpagina van OOBE en start dan Shift+F10, of typ de korte versie.
- **Shift+Fn+F10:** sommige laptops hebben Shift+Fn+F10 nodig.
- **Voor je test:** wil je in je VM de Pastebin-route proberen, voeg dan tijdelijk een NAT-adapter toe. OOBE begint daarna waarschijnlijk weer bij het begin.
- **Eerste deel van Windows Setup:** in het gedeelte met taal, "Nu installeren" en partities bestaat er nog geen Windows en dus ook geen account. Daar heeft dit geen zin; gebruik daarvoor route C.

## 3. Back-up B: antwoordbestand zonder USB

Gebruik dit als A niet meer werkt, bijvoorbeeld omdat `msoobe.exe` ontbreekt of OOBE na de herstart opnieuw begint. Het script staat in `oobe-antwoordbestand.ps1`. Je kunt het op Pastebin zetten en op dezelfde manier downloaden als hierboven.

Wat het doet:

1. Het vraagt naam en wachtwoord (met asterisken, twee keer).
2. Het schrijft een antwoordbestand naar `C:\Windows\Panther\unattend.xml`.
3. Het start `sysprep /oobe /unattend:... /reboot`.
4. OOBE start opnieuw en leest het antwoordbestand.

Het antwoordbestand ziet er zo uit (het script vult naam en wachtwoord in):

```xml
<unattend xmlns="urn:schemas-microsoft-com:unattend" xmlns:wcm="http://schemas.microsoft.com/WMIConfig/2002/State">
  <settings pass="oobeSystem">
    <component name="Microsoft-Windows-Shell-Setup" processorArchitecture="amd64" publicKeyToken="31bf3856ad364e35" language="neutral" versionScope="nonSxS">
      <UserAccounts>
        <LocalAccounts>
          <LocalAccount wcm:action="add">
            <Name>JOUWNAAM</Name>
            <DisplayName>JOUWNAAM</DisplayName>
            <Group>Administrators</Group>
            <Password>
              <Value>BASE64-VAN-WACHTWOORD+Password-IN-UTF16</Value>
              <PlainText>false</PlainText>
            </Password>
          </LocalAccount>
        </LocalAccounts>
      </UserAccounts>
      <OOBE>
        <ProtectYourPC>3</ProtectYourPC>
        <HideEULAPage>true</HideEULAPage>
        <HideWirelessSetupInOOBE>true</HideWirelessSetupInOOBE>
        <HideOnlineAccountScreens>true</HideOnlineAccountScreens>
        <HideLocalAccountScreen>true</HideLocalAccountScreen>
      </OOBE>
      <FirstLogonCommands>
        <SynchronousCommand wcm:action="add">
          <Order>1</Order>
          <Description>Antwoordbestanden verwijderen</Description>
          <CommandLine>cmd.exe /c del /q /f "C:\Windows\Panther\unattend.xml" "C:\Windows\Panther\unattend-original.xml"</CommandLine>
        </SynchronousCommand>
      </FirstLogonCommands>
    </component>
  </settings>
</unattend>
```

Uitleg:

- **Account in het bestand:** door het lokale account hier te definiëren slaat OOBE de accountschermen over, omdat het account al bestaat. Daardoor vraagt OOBE nooit om een Microsoft-account.
- **`Administrators`:** Windows herkent deze Engelse groepsnaam en zet hem om naar de lokale naam, ongeacht de taal van het image.
- **Aanmeldpagina:** `HideOnlineAccountScreens` verbergt de aanmeldpagina.
- **Netwerkpagina:** `HideWirelessSetupInOOBE` verbergt de netwerkpagina.
- **Pagina's die je niet regelt:** OOBE-pagina's die je niet in het antwoordbestand instelt, verschijnen gewoon. Daarom zie je regio en toetsenbord opnieuw.
- **`HideLocalAccountScreen`:** volgens de documentatie geldt dit alleen voor Windows Server. Het staat erin omdat het ook in een als werkend gemeld voorbeeld staat, en het is onschadelijk.
- **Waarom dit werkt:** Windows zoekt bij elke configuratiefase naar een antwoordbestand, onder meer in `%WINDIR%\Panther\Unattend`. Een daar neergezet bestand vervangt aan het begin van elke fase de gecachte kopie.

Let op bij B:

- **Alleen in OOBE:** sysprep op een al ingestelde pc stuurt die terug door de setup. Het script vraagt daarom om `CONTINUE` als het geen OOBE herkent.
- **Wachtwoord in het bestand:** base64 is geen echte beveiliging. Setup haalt gevoelige gegevens uit de gecachte kopie aan het einde van elke fase, en het bestand wordt bij de eerste aanmelding verwijderd. Controleer daarna toch `C:\Windows\Panther\unattend*.xml`.
- **Bestaat het account al:** na methode A verwijdert het script het eerst, en het antwoordbestand maakt het opnieuw aan.
- **ARM64:** het script kiest de juiste architectuur zelf.
- **Als sysprep niet herstart:** de reden staat in `C:\Windows\System32\Sysprep\Panther\setuperr.log`.
- **Wachtwoord nooit laten verlopen:** dat zet alleen methode A. Bij B kun je dat na de eerste login zetten met `Set-LocalUser -Name JOUWNAAM -PasswordNeverExpires $true` in PowerShell als beheerder.

## 4. Noodroute C: antwoordbestand op media

Wil je het vanaf het begin automatisch, of werken A en B niet meer, dan is een `autounattend.xml` op de installatiemedia de officiële weg.

- **Waar Setup zoekt:** in de windowsPE-fase moet het bestand `Autounattend.xml` heten. Setup zoekt in de root van verwisselbare media, ook van read-only media zoals een dvd.
- **Instellingen voor die fase:** een bestand zonder instellingen voor de lopende fase wordt genegeerd. Maak het dus met een generator.
- **Rufus:** Rufus kan een antwoordbestand toevoegen met "Remove requirement for an online Microsoft account" en "Create a local account with username".
- **Unattend-generator:** de unattend.xml-generator van Christoph Schneegans wordt aanbevolen voor eigen antwoordbestanden.
- **VMware:** een ISO met `autounattend.xml` in de root, gekoppeld als tweede virtuele dvd, valt onder die read-only media.

## 5. Wat werkt waar (stand oktober 2026)

- **A (`net user`-aanpak):** op 24H2 meldde een lezer al in februari 2025 dat het nog werkte, met inlogfout daarna. Verder werkt het op 25H2 en 26H2 (bronnen hierboven). Voor 23H2 en ouder vond ik geen recente testmelding.
- **B:** getest door een derde op 25H2 en 26H2, zie hierboven.
- **`OOBE\BYPASSNRO`:** volgens een bron uit augustus 2026 is het script weg, maar de registerwaarde wordt nog geaccepteerd. Een forumgebruiker meldt juist dat het script in de 26H2-ISO terug is. De bronnen spreken elkaar dus tegen. Handmatig:

```cmd
reg add HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\OOBE /v BypassNRO /t REG_DWORD /d 1 /f
shutdown /r /t 0
```

  Na de herstart kies je "I don't have internet", dan "Continue with limited setup" en daarna het account. Laat het wachtwoord leeg om de vragen te vermijden en stel het daarna in.
- **`start ms-cxh:localonly`:** in Insider-build 26120.6772 (oktober 2025) geblokkeerd. Een forumgebruiker maakte er in oktober 2026 nog 26H2-VM's mee. Je krijgt wel de drie vragen zodra je een wachtwoord instelt.
- **"Learn more"-link:** op 26H2 gepatcht. Verspil er geen tijd aan.
- **Registerwaarde `HideOnlineAccountScreens`:** werkte volgens de bron op 25H2-Insider-builds; op 26H2 is dit niet getest.
- **Alleen Pro:** "Set up for work or school" en dan "Domain join instead".
- **Variant van A:** een omslachtigere versie verwijdert ook `defaultuser0` en past OOBE-registerwaarden aan. Die staat beschreven als "Method 7" in aantekeningen van David Bombal. Probeer die alleen als A OOBE niet netjes afrondt.

## 6. Valkuilen

- **Toetsenbordindeling:** kies in OOBE eerst regio en toetsenbord, voordat je het wachtwoord typt. Gebruik bij twijfel een eerste wachtwoord zonder speciale tekens en wijzig het na het inloggen.
- **Geen herstelroute:** zonder Microsoft-account of beveiligingsvragen kun je een vergeten wachtwoord niet zelf herstellen. Bewaar het goed.
- **Console verdwijnt:** na elke herstart (BypassNRO, sysprep) is het CMD-venster weg. Open het opnieuw met Shift+F10.
- **Snapshot:** neem er een in OOBE, dan kun je A, B en C steeds opnieuw proberen.
- **Nieuwe builds:** test elke nieuwe build eerst in een VM. Als A mislukt, ga je naar B. Als dat ook faalt, naar C.

Laat me weten wat er in je VM gebeurt (build, foutmelding, welke stap), dan stem ik de scripts af.
