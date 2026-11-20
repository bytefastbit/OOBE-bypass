**Kort antwoord:** ja, dit kan in één regel, zonder beveiligingsvragen en zonder dat het uitmaakt of er netwerk is. Alleen niet met `start ms-cxh:localonly` of `oobe\bypassnro`. Microsoft heeft die twee commando's uit Insider-build 26220.6772 gehaald. Voor 26H2 zijn de berichten tegenstrijdig: onafhankelijke tests meldden dat ook de BypassNRO-registersleutel en de ms-cxh-commando's niet meer werkten, terwijl een forumgebruiker in een VMware-VM beide commando's nog werkend vond op de eerste officiële 26H2-ISO (build 26300.9457). Voor jou maakt dat weinig uit, want beide eindigen bij de standaardpagina met de beveiligingsvragen.

Wat wél werkt, is de gebruiker zelf aanmaken vanaf de Shift+F10-prompt en OOBE daarna laten afronden met `msoobe`. Een bron over 26H2 noemt dit een zeer betrouwbare methode die nog werkt. Het account wordt rechtstreeks in de lokale accountdatabase gemaakt, zonder OOBE-scherm of snelkoppeling, waardoor het de route is die het best tegen toekomstige patches bestand is.

**Het commando** (typ het precies over in het CMD-venster waarin je nu zit):

```
powershell -c "$u=Read-Host Naam;$p=Read-Host Wachtwoord -AsSecureString;New-LocalUser $u -Password $p -PasswordNeverExpires -EA Stop;Add-LocalGroupMember -SID S-1-5-32-544 -Member $u" && cd oobe && msoobe && shutdown -r
```

- **Wachtwoord met sterretjes:** je typt eerst de naam en daarna het wachtwoord, waarbij je per teken een `*` ziet. PowerShell werkt in die prompt: Microsoft beschrijft zelf dat je vanuit de Shift+F10-prompt in OOBE `powershell.exe` kunt starten. `New-LocalUser` accepteert een wachtwoord dat met `Read-Host -AsSecureString` is ingevoerd.
- **Administrator-rechten, in elke taal:** `S-1-5-32-544` is de vaste identiteit van de groep Administrators en is op elk Windows-systeem hetzelfde, ongeacht de taal. Dat is nodig omdat de groepsnaam in vertaalde Windows-versies kan verschillen, dus `net localgroup administrators ...` kan falen. `Add-LocalGroupMember` kan de groep via SID aanspreken. Het wordt jouw eigen gebruiker, niet het ingebouwde Administrator-account.
- **`-PasswordNeverExpires`:** dit is mijn toevoeging. Zonder geldt het standaardbeleid van 42 dagen. Haal het weg als je dat wilt.
- **`&&`:** de volgende stap start alleen als de vorige slaagde. Mislukt het aanmaken, dan wordt OOBE niet afgesloten en kun je het opnieuw proberen.
- **`cd oobe && msoobe && shutdown -r`:** dit rondt OOBE af en herstart na ongeveer 30 seconden.

**Daarna:** na de herstart verschijnt een foutmelding over een onjuiste gebruikersnaam of wachtwoord, die je mag negeren. Kies dan onderaan het account dat je net hebt gemaakt. De eerste aanmelding duurt even. Op het inlogscherm staat ook `defaultuser0`, een restant van het tijdelijke OOBE-account dat zichtbaar blijft totdat je het uitschakelt. Verwijder het in een CMD met beheerdersrechten, en daarna eventueel de map `C:\Users\defaultuser0`:

```
net user defaultuser0 /delete
```

Omdat OOBE wordt afgekapt, vallen ook pagina's als apparaatnaam en privacykeuzes weg. Microsoft waarschuwt dat zulke omwegen kritieke setup-schermen kunnen overslaan, dus controleer die instellingen achteraf. Het blijft een community-methode zonder steun van Microsoft.

**Met of zonder netwerk:** voor deze methode maakt het niet uit, dus je hoeft niets apart te doen.
- **Geen netwerkadapter (jouw VM):** precies zoals hierboven.
- **Adapter aanwezig maar niet verbonden:** identiek, want de netwerkpagina wordt overgeslagen.
- **Wel internet (kabel of wifi):** identiek. Je hoeft de verbinding niet eerst te verbreken.

**Jouw build en latere versies:** ik kan het zelf niet testen en vond geen expliciete bevestiging voor precies build 26300.9974. De methode staat beschreven voor 25H2 en wordt, zoals hierboven, ook voor 26H2 genoemd. Een bovengrens is er niet bekend, maar Microsoft kan `msoobe` of de OOBE-logica altijd aanpassen. Controleer het daarom bij elke nieuwe build in je VM, zoals je nu doet. Als het ooit dichtgaat, is een `autounattend.xml` of een met Rufus gemaakte USB plan B. Die gebruiken officiële functies en blijven volgens de bron voorlopig relevant. Ze moeten wel klaarstaan vóór de installatie.

**Voor echte laptops en desktops:**
- Sommige laptops vragen om Fn+Shift+F10, en op sommige afgeschermde OEM-images staat de sneltoets uit.
- Voer het commando uit na de pagina's voor land en toetsenbord, zoals nu. Anders sla je die keuzes waarschijnlijk ook over.
- In de blauwe Setup-schermen vóór de eerste herstart werkt het niet, want dan is Windows nog niet geïnstalleerd.
- Gebruik voor de zekerheid alleen letters en cijfers in naam en wachtwoord, en pas het wachtwoord na de eerste aanmelding eventueel aan. Dan kan een afwijkende toetsenbordindeling in de prompt geen probleem geven.

Wil je dat het wachtwoord twee keer wordt gevraagd en vergeleken? Dat kan in dezelfde regel, maar hij wordt dan een stuk langer om over te typen.
