Coding Guidelines
=================

# Namespaces:
**Eigene Apps**: Verwende den Namespace:  `PSG.App.[Name der App]`. Der *Name der App* wird zusammen geschrieben (z.B. PSG.App.MeineNeueApp).

**Kundenprojekte**: Verwende `PSG.Customization.[Name des Kunden]`. Der *Name des Kunden* wird zusammengeschriebenen (z.B. PSG.Customization.MustermannAG).

# Benamung von Objekten (Tabellen, Pages & Co.)

Alle Objekte erhalten den Präfix `PSG_`. Gefolgt von den eigentlichen Namen **ohne** Sonderzeichen.<br>
Beispiele: 
- PSG_MeineNeueTabelle
- PSG_MeinNeuesFeld
- PSG_MeinePageGroup
- PSG_MeinEnumValue
- PSG_MeinePage

> Ausgeschlossen von dieser Regel sind `Prozeduren` & `Event-Subcriber`.

# Benamung von Dateinamen

Die Dateinamen müssen nach folgenden Schema benamt werden: `PSG[Objekt-Name].[Objekt-Typ].al`. Bei **Extensions** wird der Objekt-Name mit **Ext** abgekürzt. Der Unterstrich "**_**" vom Objekt-Namen entfällt dabei.  <br>
Beispiele:
- PSGMeineCodeunit.Codeunit.al
- PSGMeineTableExt.TableExt.al
- PSGMeinePage.Page.al
- PSGMeinEnum.Enum.al


##### Tabellenfelder:
###### Eigene Tabellenfelder: Beginne die ID mit 1.
###### Extension-Tabellenfelder: Nutze den ID-Bereich der jeweiligen App.
###### Tabellenfelder heißen (z.B. PSG_OrderNo). Das Feld wird mit PSG_'Der Name des Feldes zusammen' geschrieben.
###### Der ID-Bereich von 50900 bis 50999 ist für Kunden reserviert, die eigene Entwicklungen vornehmen möchten. In diesem Bereich werden individuelle Anpassungen ermöglicht.
##### Seitenfelder:
###### Das Feld wird auf der Page wieder mit Anführungzeichen geschrieben. (z.B. "Order Quantity"; Rec.PSG_OrderQuantity) 
##### Variablen:
###### Benenne Variablen so, dass sie dem entsprechenden Record entsprechen (z.B. Kunde, Bestellung).
##### Events:
###### Die bestehenden Eventbenennungen bleiben unverändert.
##### Versionsnummerierung
##### Platform &lt;major&gt;.&lt;minor&gt;.&lt;build&gt;.&lt;revision&gt;
##### Major = Version von BC
##### Minor = Interne Versionierung
##### Build = Automatische Nummer beim Builden der App aus GitHub
##### Revision = Hofixes und kleiner Änderungen
###### Entwicklung: Verwende das Format "BCVersion.0.0.1".
###### Produktivstart: Ändere auf "BCVersion.1.0.0".