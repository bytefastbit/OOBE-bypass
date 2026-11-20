Ja, dit kan, en zonder `start ms-cxh:localonly`. Maak het account zelf aan vanuit de Shift+F10-console en rond OOBE daarna af. Dan komen de netwerkpagina, de Microsoft-accountpagina en de beveiligingsvragen niet meer langs. Tech Based meldt in oktober 2026 dat deze aanpak op 26H2 nog werkt. Admin365 noemt het de duurzaamste methode, omdat de lokale accountdatabase direct wordt aangepast in plaats van een OOBE-snelkoppeling te gebruiken.

Ik kan dit zelf niet in een VM uitproberen. De bronnen gebruiken `net user`, maar dat toont tijdens het typen helemaal niets, dus ook geen sterretjes. Daarom gebruik ik PowerShell: `Read-Host -AsSecureString` toont voor elk getypt teken een *. Winaero gebruikt PowerShell ook in deze OOBE-console om het account aan te maken. De combinatie hieronder is mijn eigen samenstelling op basis daarvan.

## Stappen (je zit al op "Let's connect you to a network")

1. Maak eventueel eerst een VMware-snapshot.
2. Typ in je CMD-venster `powershell` en druk op Enter.
3. Typ dit, elke regel met Enter:

```powershell
$n = Read-Host 'Gebruikersnaam'
do {
  $a = Read-Host 'Wachtwoord' -AsSecureString
  $b = Read-Host 'Herhaal wachtwoord' -AsSecureString
  $x = [pscredential]::new('x', $a).GetNetworkCredential().Password
  $y = [pscredential]::new('x', $b).GetNetworkCredential().Password
} until ($x.Length -gt 0 -and $x -ceq $y)
$u = New-LocalUser -Name $n -Password $a -PasswordNeverExpires -AccountNeverExpires
Add-LocalGroupMember -SID S-1-5-32-544 -Member $u
Get-LocalGroupMember -SID S-1-5-32-544
exit
```

   - Je ziet per teken een `*`. Je typt het wachtwoord twee keer en de lus herhaalt tot beide gelijk en niet leeg zijn.
   - `S-1-5-32-544` is in elke taal de groep Administrators. De naam van die groep verschilt per taal, dus een SID is veiliger dan "Administrators" intypen.
   - `-PasswordNeverExpires` voorkomt dat Windows na 42 dagen (het standaardbeleid) om een nieuw wachtwoord vraagt. Laat het weg als je dat wél wilt.
   - De laatste regel toont de beheerders, waar jouw naam bij moet staan. Bij een rode foutmelding (bijvoorbeeld "naam bestaat al") ga je niet verder. Verwijder het account dan met `Remove-LocalUser -Name <naam>` en begin opnieuw.

4. Je zit weer in CMD. Typ:

```
cd /d %windir%\System32\oobe
msoobe && shutdown -r
```

   Deze afsluiting met msoobe en herstart staat ook bij Winaero. De pc herstart na ongeveer 30 seconden.

5. Na de herstart: je krijgt een melding dat gebruikersnaam of wachtwoord onjuist is, en het inlogscherm toont eerst defaultuser0. Dat hoort zo. Klik OK, kies linksonder jouw account en log in. Volgens Winaero verdwijnt defaultuser0 na je eerste aanmelding. Blijft hij zichtbaar, dan schakel je hem uit met `net user defaultuser0 /active:no` in een terminal als beheerder.

## Wat je hiermee wel en niet krijgt

- Jouw account zit in de groep Administrators en het ingebouwde Administrator-account blijft uitgeschakeld. Je krijgt wel de gewone UAC-meldingen, zoals bij elk beheerdersaccount.
- Er zijn geen beveiligingsvragen, dus ook geen wachtwoordherstel via het inlogscherm als je het vergeet.
- De rest van OOBE wordt overgeslagen, zoals privacykeuzes en de apparaatnaam. Microsoft noemt precies dat als reden om dit soort omwegen te blokkeren. Loop na de eerste aanmelding even Instellingen → Privacy en beveiliging na.

## Netwerk: geen aparte methode nodig

De stappen raken de netwerkpagina niet, dus ze zijn identiek voor drie situaties:

- Er is geen adapter, zoals in jouw VM.
- Er is wel een adapter maar nog geen verbinding.
- Je bent al verbonden via kabel of Wi-Fi.

Het enige verschil is wat er daarna gebeurt. Met een actieve verbinding gaat Windows na de herstart meteen online, dus ontkoppel vooraf als je dat niet wilt. Anders verbind je later op het bureaublad.

## Welke versies, en wat als het ooit niet meer werkt?

De aanpak is gedocumenteerd voor 24H2 (PCWorld), 25H2 (een GitHub-gist, WindowsOSHub) en 26H2 (Tech Based). In een ElevenForum-thread over 26H2-builds, ook jouw 26300.9974, staan dezelfde commando's, al werd daar vooral `start ms-cxh:localonly` bevestigd.

Ik vond geen melding dat Microsoft deze methode blokkeert. Windows Latest zag in een Insider-build van oktober 2025 wél de BypassNRO-registerwaarde, ms-cxh:localonly, setaddlocalonly, nepaccount-trucs en oobenetworkconnectionflow geblokkeerd, maar niet deze methode. Ook de "Learn more"-link is in oktober 2026 server-side dichtgezet.

Voor toekomstige builds kan niemand iets garanderen. Microsoft kondigde in oktober 2025 aan bekende mechanismen voor lokale accounts uit OOBE te halen, omdat ze kritieke setup-schermen overslaan, en deze methode doet dat ook. Test daarom elke nieuwe build eerst in een VM met snapshot.

Valt de methode weg, dan zijn dit de officiële en duurzamere routes:

- Rufus met de optie om de online-accounteis te verwijderen en een lokaal account aan te maken.
- Een autounattend.xml-antwoordbestand.
- Bij Windows 11 Pro: "Set up for work or school" en dan "Domain join instead".

gHacks beschrijft deze drie. Een antwoordbestand kun je ook achteraf vanuit Shift+F10 toepassen met sysprep /oobe /unattend. Nadeel: het wachtwoord staat dan in dat bestand en je typt het niet interactief met sterretjes.

## Talen, schermen en echte pc's

- **Talen:** de SID en de PowerShell-cmdlets zijn taalonafhankelijk, en de gebruikersnaam mag in elke taal.
- **Toetsenbord:** doe dit bij voorkeur ná de toetsenbordkeuze, zoals nu, zodat de console dezelfde indeling gebruikt als het inlogscherm. Klik bij de eerste aanmelding vóór Enter op het oogje om je wachtwoord te controleren.
- **Schermen:** de bronnen starten bij de landselectie (het eerste OOBE-scherm) en het werkt ook op de Microsoft-accountpagina. Alle OOBE-schermen daartussen zouden dus werken.
- **Setup-fase:** voor de eerste herstart werkt het niet, bijvoorbeeld bij de schijfkeuze. Daar opent Shift+F10 een tijdelijke omgeving en komt een account niet in je geïnstalleerde Windows terecht. In die fase heb je alleen een antwoordbestand of Rufus.
- **Echte pc's:** op sommige laptops is het Shift+Fn+F10 (volgens WindowsOSHub), en sommige OEM-images blokkeren de sneltoets.

Als een stap iets anders doet dan ik beschrijf, laat dan weten welke melding of welk scherm je ziet, dan denk ik mee.
