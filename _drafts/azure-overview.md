---
layout: post
title:  "Eine Übersicht zu den Microsoft Azure Diensten"
date:   2021-05-12 11:31:00 +0200
categories: [programming, cloud, azure]
---

## Einleitung

Ich habe selbst schon öfters mit der Azure Cloud gearbeitet. Dabei habe ich
hauptsächlich die "Infrastructure as a Service" (kurz _IaaS_) Dienste, wie
Virtuelle Maschinen und Netzwerke genutzt. Von den "Software as a Service"
(_SaaS_) Diensten beschränkte sich mein Erfahrungshorizont auf den MS SQL
Server. Ich wusste, dass da mehr ist, und hatte auch im Groben einen Überblick
darüber, was sonst noch alles angeboten wird. Aber die Details waren verwirrend,
schon nur weil viele der Namen erstens nicht sehr aussagekräftig sind und
zweitens erhebliche Überlappungen und Verwechslungspotential bieten.

In Diskussionen mit Arbeitskolleg*innen, insbesondere aus dem nicht-technischen,
eher kommerziellen Umfeld, stellte ich fest, dass ich nicht der einzige bin, dem
es so geht. In einer dieser Diskussionen kam dann die Idee auf, eine Übersicht
mit Kurzzusammenfassungen zu allen angebotenen Azure Diensten zu erstellen.
Diese sollte für IT-affine Nicht-Spezialisten, und vor allem
Nicht-Programmierer, zugänglich sein, und sich auf das _WAS_ statt das _WIE_
fokussieren.

Also setzte ich mich hin und wühlte mich durch Unmengen and Marketing-Material
voll von Worthülsen und notierte mir zu jedem der Dienste ein paar Sätze. Ich
orientierte mich dabei an den Gruppierungen der Dienste, wie sie auch von
Microsoft verwendet wird. Allerdings tauchen viele Dienste in den Dokumenten von
Microsoft unter mehren Gruppen auf. Einerseits macht das Sinn, weil die Dienste
in verschiedenen Kontexten genutzt werden können. Aber gleichzeitig leidet
darunter die Übersichtlichkeit. Daher wählte ich die Gruppe, mit der ein Dienst
am stärksten verbunden ist, und beschrieb in nur dort. Für die meisten Leser es
eigentlich nach der Lektüre offensichtlich sein, in welchen anderen
Zusammenhängen ein Dienst auch noch sinnvoll eingesetzt werden kann. Wo
nötig, versuchte ich diese zu ähnlichen Diensten abzugrenzen. Aus dieser Arbeit
entstand dann https://wildmichael.github.io/azure-overview.

In diesem Artikel möchte ich diese Übersicht nochmals konzentrierter vorstellen.
Ich werde aber den Schnitt etwas anders anlegen, indem ich beschreibe, _WAS_ die
Azure Cloud kann. Dazu erwähne ich dann jeweils, welche Dienste dafür genutzt
werden können und gebe Hinweise auf deren Charakteristika. Auch werde ich sehr
exotische Dienste oder solche, die bald eingestellt werden, weglassen.

## Analysen

### Big Data Analysen

Microsoft Azure bietet mehrere Dienste zur Verarbeitung von grossen Datenmengen,
welche z.B. aus SQL Datenbanken, aber auch unstrukturierten Dateien bestehen
können. Microsoft setzt hier auf bekannte Open-Source Technologien wie Apache
Spark, Apache Hadoop, Apache HBase, R und Python.

* Azure Daten-Explorer
* Azure Databricks
* Azure Synapse Analytics
* Data Factory
* Data Lake Analytics
* HDInsight
* R-Server for HDInsight
* Microsoft Graph Data Connect

### Daten-Governance und Verwaltung

Grosse Firmen müssen ihre Daten einerseits gut auffindbar machen, andererseits
diese aber auch vor unerlaubten Zugriffen schützen.

* Azure Pureview
* Azure Data Catalog

### Datenstrom-Verarbeitung

Mit diesen Diensten können Daten _on-the-fly_ transformiert und analysiert
werden. Beispiele dafür könnten prädiktive Wartungstools sein, welche
Maschinendaten laufend analysieren um z.B. Anomalien zu entdecken. Diese
Analysen können auch direkt auf Geräten ausserhalb der Cloud, z.B. auf
Maschinensteuerungen, ausgeführt werden.

* Azure Stream Analytics

### Log-Daten Analyse

Grosse Applikationen generieren grosse Mengen an Log-Daten. Diese Logs können
unmöglich mehr einzeln gelesen werden, daher müssen sie mit spezialisierten
Tools analysiert werden.

* Log Analytics

### Business Intelligence

Dienste und Werkzeuge um die eigenen Geschäftsdaten zu analysieren, aber auch,
um diese unter eigenem Namen in Produkte einzubauen.

* Azure Analysis Services
* Power BI Embedded

## Rechnerinfrastruktur

Wie in fast jeder anderen Cloud, können auch in Azure Rechner/Server als Dienst
gemietet werden. Dies kann notwendig oder von Vorteil sein, wenn sonst kein
Angebot die funktionalen Anforderungen, oder aber auch die
Leistungsanforderungen, nicht erfüllt. Oder z.B. aber auch, wenn
Legacy-Anwendungen in die Cloud migriert werden sollen. Mit all diesen Diensten
übernimmt der Kunde die Verantwortung für Updates und Wartung des
Betriebssystems. 

* Azure CycleCloud
* VMWare-Lösungen in Azure
* Virtuelle Computer
* Microsoft Azure Virtual Machine Scale Sets
* Azure Dedicated Host
* Windows Virtual Desktop
* VMware Horizon Cloud in Microsoft Azure
* Virtuelle Citrix-Apps und -Desktops für Azure

## Betrieb von Anwendungen

### Hosting von Applikationen

Diese Dienste erlauben dem Kunden das Betreiben von Anwendungen auf von
Microsoft verwalteter Infrastruktur. D.h. der Kunde ist im Gegensatz zu den
zuvor erwähnten Diensten, nicht für die Updates und Wartung des Betriebssystems
verantwortlich.

* Azure Spring Cloud
* Cloud Services
* SQL Server auf VM
* Mobile Apps

### Microservices und Container

* Service Fabric
* Azure Kubernetes Service
* Azure Red Hat OpenShift
* Azure Container Instances
* Web-App für Container
* Container Registry

### Serverlose Anwendungen

* Azure Functions

### API's

Moderne Applikations-Architekturen und Designs beherzigen oft den API-first
Ansatz. Dies erlaubt es auch viel einfacher, Dienste miteinander zu verknüpfen
und Automatisierungen zu bauen. So können auch Frontends von dem Backend
getrennt werden, und das Frontend z.B. nur noch als statische Webseite
veröffentlicht werden, welche dann auf per REST API auf eine separate API als
Backend zurückgreift. Sogenannte Microservice- und Service-Mesh-Architekturen,
obwohl nicht gleich, benötigen eine oder mehrere Gateways. Bidirektionale
Kommunikation wird auch immer mehr, z.B. für Chats und Benachrichtigungen, zur
Anzeige von Live-Daten, etc. benötigt. Für all diese Bedürfnisse bietet
Microsoft in der Azure Cloud Dienste.

* API-Apps
* API Management
* App Service
* Azure SignalR Service
* Statische Web-Apps

## Speicher

### Datenbanken

Eine der wohl wichtigsten Komponenten für eine Vielzahl von Applikationen ist
die Datenbank. Entsprechend gross ist das Angebot, welches verschiedene Formen
von dem Microsoft-eigenen SQL Server, aber auch verschiedene open-source SQL
Datenbanken und sogenannte No-SQL Datenbanken, wie Key-Value-Stores, Dokumenten-
und Graphendatenbanken umfasst.

* Azure API für FHIR (Gesundheitsdaten)
* Azure Cache for Redis
* Azure Cosmos DB
* Azure Database for MariaDB
* Azure Database for MySQL
* Azure Database for PostgreSQL
* Azure SQL Datenbank
* Azure SQL Edge
* Azure SQL Managed Instance
* Table Storage
* Azure Managed Storage for Apache Cassandra

### Andere Speicher

Um Daten in unstrukturierten, z.B. in Dateisystem ähnlichen Strukturen, zu
speichern gibt es für unterschiedliche Anwendungsszenarien spezialisierte
Dienste. Es können z.B. traditionelle Netzwerklaufwerke in die Cloud migriert
werden. In sogenannten Blob-Speichern können Anwendungen Daten ablegen, für
welche sich eine Datenbank schlecht eignet (z.B. Benutzerbilder oder
Dokumentenuploads). Es können aber auch Archive und Backups gespeichert werden.
Für wissenschaftliche Simulationen stehen speziell schnelle Speicherdienste
bereit.

Falls sehr grosse Datenmengen in die Cloud migriert werden sollen und dies über
das Netzwerk zu lange dauern würde, stellt Microsoft auch physische Speicher
verschiedener Grösse zur Verfügung, mit welchen die Daten in das
Cloud-Datencenter transportiert werden können.

Sollen Daten sicher mit Drittparteien geteilt werden, stellt auch Microsoft hier
einen Dienst bereit.

* Archive Storage
* Avere vFXT für Azure
* Azure Backup
* Azure Data Lake Storage
* Azure Data Share
* Azure Files
* Azure FXT Edge Filer
* Azure HPC Cache
* Azure NetApp Files
* Azure Blob Storage
* Azure Data Box
* Azure Disk Storage
* Queue Storage
* Speicher Explorer
* StorSimple

## Stapelverarbeitung

Viele Applikationen laufen als Stapelverarbeitung. Wie z.B. Lohnabrechnungen,
wissenschaftliche Simulationen oder Berechnen von Portfoliorisiken. Oftmals
benötigen diese Anwendungen viel Rechnerleistung, welche dann zwischen den
Durchläufen brach liegt und Kosten verursacht. Mit diesem Dienst zur
Stapelverarbeitung können die Ressourcen automatisch skaliert werden.

* Batch

## Entwicklerwerkzeuge und DevOps

Für Entwickler bietet Microsoft in Azure einige Dienste an. Einerseits,
für das agile Projektmanagement, Versionskontrolle und Build Server für
Continuous Integration und Deployment, aber auch Dienste, um Softwarepakete
für .NET, NPM, und Python zu veröffentlichen oder im Team zu teilen. Weiter
gibt es Dienste zur Bereitstellung von standardisierten Entwicklungs- und
Testumgebungen. Manuelle und explorative Tests lassen sich auch planen und
protokollieren. All diese Dienste werden in dem Azure DevOps Portal
zusammengefasst.

Für Schulungen, Kurse, Hackathons, etc. lassen sich bedarfsgesteuert
vorkonfigurierte virtuelle Maschinen bereitstellen.

Um Applikationen in Azure Dienste einzubinden und um die Azure Ressourcen
programmgesteuert zu verwalten bietet Microsoft
SDK's für eine grosse Anzahl Programmiersprachen an und stellt
Kommandozeilentools für Scripting bereit.

Um die Anwendungskonfiguration zu vereinfachen, bietet Azure einen Dienst, um
die Parameter zentral zu verwalten.

* Azure Artifacts
* Azure Boards
* Azure DevTest Labs
* Azure Pipelines
* Azure Repos
* Azure Test Plans
* Azure Lab Services
* Azure SDK's
* App Configuration

## Hybrid und Multicloud

Viele Anwendungsszenarien verlangen, dass Azure Ressourcen mit Computer,
Programmen oder Geräten ausserhalb der Cloud zusammenarbeiten. Z.B. können das
IoT Geräte wie Anlagensteuerungen, Kassensystem, noch nicht in die Cloud
migrierte on-premise Dienste, etc. pp. sein. Oder aber eine Firma will sich
nicht auf einen Cloud-Anbieter festlegen, möglicherweise werden aus
Verfügbarkeitsgründen sogar die gleichen Applikationen in mehreren Clouds
redundant betrieben. All diese Szenarien stellen besondere Herausforderungen an
das Verwalten der Dienste, die Überprüfung von Policies und Compliance
Richtlinien und die Sicherheit. Mit den von Microsoft angebotenen Diensten kann
man diese Aufgaben unter einer vereinheitlichten, zentralen Stelle aus erfüllen.
Andere Dienste von Microsoft in diesem Themenfeld ermöglichen es, Azure Dienste
auch ausserhalb der Cloud zu nutzen, z.B. auf Edge-Geräten, im eigenen Datacenter
oder an Orten ohne Netzwerkanbindung.

* Azure Arc
* Azure ExpressRoute
* Azure Stack
* Azure Stack Edge
* Azure Stack HCI
* Azure Stack Hub
* Azure Modular Datacenter

## Identität

Mit Azure Active Directory bietet Microsoft einen sehr umfangreichen Dienst zur
Authentifizierung und Autorisierung mit modernsten Protokollen wie OpenID
Connect und Multifaktor-Authentifizierung (MFA). Weiter bietet Microsoft
Domänenkontroller zur Verwaltung von virtuellen Maschinen in der Azure Cloud
bereit. Auch können Zugriffe über Organisationsgrenzen hinweg an externe
Personen, z.B. Kunden oder Partnern gewährt und verwaltet werden.

* Azure Active Directory
* Azure Active Directory Domain Services
* Externe Azure Active Directory Identitäten

## Integration, Messaging, Events

Microsoft bietet mehrere Dienste an, mit denen Applikationen untereinander
mittels Ereignissen und Nachrichten kommunizieren können. Je nach Charakteristik
eignen sich diese eher für wenige grosse, komplexe Datenpakete, oder aber für
kleine Ereignisse in sehr grosser Zahl. Diese können genutzt werden, um
Applikationen in kleine, lose gekoppelte Teile zu zerlegen oder bestehende
Applikationen miteinander zu integrieren. Die Ereignisse oder Nachrichten können
aber z.B. auch von IoT Geräten erzeugt werden.

* Service Bus
* Event Grid
* Logic Apps
* Event Hubs
* Queue Storage

## Internet of Things (IoT)

Unter der Flagge _IoT_ bietet Microsoft sehr unterschiedliche
Dienste an. Diese reichen von der Erstellung, Auslieferung und Wartung der
Software auf IoT Geräten über die Verwaltung ebendieser Geräte bis hin zur
Datenverarbeitung der durch die Geräte erzeugten Daten. Microsoft bietet sogar
ein eigenes real-time Betriebssystem für Mikrokontroller an. Natürlich können
die Geräte auch überwacht werden. Um die Geräte und die Kommunikation
abzusichern bietet Microsoft auch mehrere Dienstleistungen an. Weiter können
sogenannte Digital Twins erstellt werden, mit denen ganze Systeme in Echtzeit
simuliert werden können, um z.B. sonst nicht messbare Grössen zu erfassen, die
Effizienz von Anlagen zu maximieren oder deren Wartung zu optimieren. Letztlich
können auch AI/Machine-Learning-Anwendungen auf IoT Geräten eingesetzt und mit
den AI/Machine-Learning Diensten in der Cloud integriert werden.

* Azure Digital Twins
* Azure IoT Central
* Azure IoT Edge
* Azure IoT Hub
* Azure IoT Solution Accelerators
* Azure RTOS
* Azure Sphere
* Azure Time Series Insight
* Windows 10 IoT Services
* Azure Defender for IoT
* Azure Percept

## AI und Machine-Learning

In den letzten Jahren ist das Feld von AI und maschinellem Lernen förmlich
explodiert. Und so ist hat auch Microsoft auf der Azure Cloud ein sehr grosses
Angebot an Diensten, die sich im weitesten Sinne mit künstlicher Intelligenz und
maschinellem Lernen beschäftigen. Diese reichen von der Bereitstellung von
Infrastruktur für das Trainieren eigener Modelle bis hin zu deren produktivem
Einsatz, bis hin zu fix fertigen Lösungen, z.B. zur Verarbeitung und Analyse von
Text, Sprache, Bildern und Videos. Es können auch relativ einfach eigene Bots,
z.B. für Chats, oder Modelle zur Anomalie-Erkennung, Inhaltsmoderation oder
KI-gestützten Suche implementiert werden. Microsoft bietet auch eine kuratierte
Sammlung öffentlich zugänglicher Datensätze an. Viele der Dienste können in der
Cloud, im eigenen Datacenter oder auf Edge-Geräten eingesetzt werden.

* Azure Bot Services
* Azure Cognitive Search
* Azure Machine Learning
* Azure Open Datasets
* Azure Cognitive Services
* Anomalieerkennung
* Content Moderator
* Metrics Advisor
* Personalisierung
* Language Understanding
* Plastischer Reader
* QnA Maker
* Textanalysen
* Übersetzer
* Spracherkennung
* Text-to-Speech
* Sprachübersetzung
* Sprechererkennung
* Custom Vision
* Formularerkennung
* Gesichtserkennung
* Maschinelles Sehen
* Videoindizierung
* Bing Websuche
* Virtuelle Data-Science Computer
* Microsoft Genomics
* Project Bonsai
* Health Bot

## Multimedia

In der Azure Cloud werden Dienste zur Verarbeitung, Analyse, Verbreitung und
Wiedergabe von Multimedia-Inhalten angeboten, welche z.B. auch schon für die
Übertragung von den Olympischen Spielen genutzt wurden. Weiter können diese
Inhalte mit DRM vor Diebstahl geschützt werden.

* Azure Media Player
* Content Delivery Network
* Content Protection
* Codierung
* Live- und On-Demand Streaming
* Live Video Analytics

## Migration

Um grossen Organisationen die komplexe und mit erheblichen Risiken verbundene
Migration in die Cloud zu erleichtern, bietet Microsoft sehr viele Dienste an,
welche z.B. den Betrieb von Legacy-Anwendungen z.B. mittels Lift-and-Shift
ermöglichen. Um den Gesamtüberblick über das Vorhaben _Migration in die Cloud_
zu behalten, bietet Microsoft aber auch spezialisierte Dienste an, mit deren
Hilfe sich das Unterfangen Planen lässt. Workloads können ermittelt bewertet,
geplant und verfolgt werden. Sämtliche Aktionen werden rückverfolgbar
aufgezeichnet und können so für Audits verwendet werden.

* Azure Migrate

## Mixed Reality

Virtual Reality und Augmented Reality, unter dem neuen Begriff _Mixed Reality_
zusammengefasst, sind mittlerweile in Freizeit und Beruf angekommen. Spiele wie
_Pokémon Go_, Navigationssysteme aber auch ganze Produktionsanlagen, nutzen
mittlerweile die Möglichkeiten, welche VR- und AR-Brillen, Mobiltelefone und
Tablets bieten. Mit den von Microsoft angebotenen Diensten lassen sich virtuelle
Inhalte z.B. an physische Orte oder Objekte binden. Mittels Analyse von Sprache
und maschinellem Sehen lassen sich auch Eingaben Erfassen. Das Rendering für
z.B. komplexe 3D-Modelle kann remote in der Cloud erfolgen und die so
entstandenen Bilder auf die Datenbrillen übertragen werden.

* Kinect DK
* Azure Remote Rendering
* Spatial Anchors
* Object Anchors

## Mobile

Die meisten in der Azure Cloud angebotenen Dienste lassen sich
selbstverständlich auch im Kontext von Mobile-Apps nutzen, z.B. um deren Backend
zu erstellen. Dennoch bietet Microsoft einige, speziell auf die Erstellung von
Mobile-Applikationen ausgerichtete Dienste an. Diese reichen von Karten- und
Navigationsdiensten über Programmier-Bibliotheken und Werkzeugen und
Kommunikationsplattformen, welche es dem Entwickler ermöglichen, Video, Chat,
SMS und Telefonie in der App zu nutzen.

* Azure Maps
* Visual Studio App Center
* Xamarin
* Azure Communications Services

## Netzwerk

Auch dieser Bereich ist aufgrund seiner Diversität und Komplexität sehr gross
und bietet sehr viele Dienste. Da gibt es Dienste zur Erstellung und Verwaltung
von virtuellen Netzwerken zwischen Cloud-Ressourcen wie virtuellen Maschinen,
Applikationen, Containern, etc. Für Webapplikationen gibt es Reverse Proxies,
Firewalls und Load-Balancer. Ein Dienst kombiniert das Content Distribution
Network (CDN) von Azure mit Sicherheitsdiensten wie Web-Application Firewall,
SSL-Offloading und Schutz vor DDoS Angriffen. Es können private Tunnel
eingerichtet werden, um on-premise Netzwerke und Ressourcen mit denen in der
Cloud zu verbinden. Um z.B.  Filialen miteinander zu verbinden können virtuelle
WANs angelegt werden. Der Netzwerkverkehr kann überwacht und gesteuert, die
verwendeten Ressourcen bedarfsgesteuert skaliert werden. Routinginformation kann
automatisch zwischen on-premise und Cloud-Netzwerken ausgetauscht werden.
Entlegene Orte ohne Netzwerkinfrastruktur können mittels Satellitenverbindung in
ein Netzwerk eingebunden werden.  Betreiber von eigenen Kommunikationssatelliten
können deren Einsatz, die Erdstationen etc. planen und als eigene Dienste auf
Azure anbieten.

* Application Gateway
* Azure Bastion
* Azure DNS
* Azure Firewall
* Lastenausgleich
* Azure Firewall Manager
* Azure Front Door
* Azure Internet Analyzer
* Azure Private Link
* Network Watcher
* Traffic Manager
* Virtual Network
* Virtual WAN
* VPN Gateway
* Web Application Firewall
* Azure Orbital
* Azure Route Server
* Azure Load Balancer

## Sicherheit

Hier bietet Microsoft eine Reihe von Diensten an welche von sehr spezialisiert
bis hin zur eierlegenden Wollmilchsau gehen. Es gibt Abwehrmechanismen gegen
DDoS Attacken, Hardware Sicherheitsmodule, Dienste zum Schutz von Informationen,
Speicher für Anwendungspasswörter, Zertifikate und ähnliches, aber auch
KI-gestützte Analysewerkzeuge. Schliesslich gibt es eine zentrale Übersicht über
die Sicherheit von Azure Diensten und hybriden Ressourcen, welche auch Berichte
für Compliance-Nachweise erstellt. Microsoft bietet auch einen Dienst zur
Sicherstellung der Integrität und Vertrauenswürdigkeit von Ressourcen ausserhalb
der Cloud an, zum Beispiel von Edge Geräten.

* Azure Defender
* Azure DDoS Protection
* Azure Dedicated HSM
* Azure Information Protection
* Azure Sentinel
* Key Vault
* Security Center
* Microsoft Azure Attestation

## Verwaltung, Automatisierung und Governance

### Automatisierung

In der Azure Cloud lassen sich viele Aufgaben und Dienste automatisieren und
mittels Vorlagen und Schablonen konsistent und vereinfacht erstellen.

* Automation
* Azure Blueprint
* Vorlagen für Azure Resource Manager
* Cloud Shell
* Azure Automanage
* Azure Resource Mover

### Governance

Microsoft unterstützt seine Kunden bei der Erstellung, Erzwingung und
Überwachung von Richtlinien in der Cloud und für on-premise Ressourcen. Ein
Ratgeber erteilt automatisch Vorschläge zur Verbesserung der Zuverlässigkeit,
Sicherheit, Leistung und Senkung von Kosten. Die Zugriffsrechte auf Daten kann für
Personen und Systeme innerhalb und ausserhalb der eigenen Organisation gesteuert
und überwacht werden.

* Azure Advisor
* Azure Lighthouse
* Azure Policy

### Verwaltung und Überwachung

Für den Betrieb von Anwendungen und Ressourcen in der Cloud bietet Microsoft
Dienste zur Überwachung, Kostenverrechnung und Wiederherstellung. Mit der
Mobile-App können viele dieser Aufgaben auch unterwegs erledigt werden.

* Azure Mobile App
* Azure Monitor
* Azure Service Health
* Azure Site Recovery
* Azure Cost Management und Abrechnung

## Anbieten eigener Dienste

Microsoft hat das Azure-Ökosystem auch für Drittanbieter geöffnet, die selbst
eigene Dienste und Ressourcen anbieten können.

* Azure Managed Applications

## Abschliessende Gedanken

Wie ersichtlich, ist die Anzahl der Dienste und die von ihnen abgedeckten
Themenfelder sehr gross. Für mich die einfacheren sind diejenigen, welche sich
auf die "Lift-and-Shift" Szenarien beziehen. D.h. bestehende Anwendungen und
Ressourcen werden mit möglichst wenig Modifikation und Aufwand in die Cloud
migriert. Allerdings kann dies nur der erste Schritt sein. Man spart sich
dadurch bestenfalls die Investitionskosten in eigene Hardware und deren Wartung,
aber dadurch verbessert sich weder die Unternehmens- noch die
Applikationsarchitektur und eine Integration in andere Dienste ist immer noch
herausfordernd. Eine echte Cloud-Strategie sollte diese Dienste nur für eine
Übergangsphase als tragend betrachten.

Die Herausforderung liegt dann darin, die Anwendungen "neu zu denken" und die
bekannten Trampelpfade zu verlassen. Die vielen "Legobausteine" wollen geschickt
kombiniert werden, um viel rascher und mit sehr wenig Code zu sehr ausgereiften
und robusten Lösungen zu kommen. Die Gefahr liegt darin, dass man dazu tendiert,
vertraute Lösungsansätze zu bevorzugen und so den Lösungsraum unnötig und
ungünstig einschränkt. Der sprichwörtliche Junge mit einem Hammer, für den jeder
Gegenstand wie ein Nagel ausschaut. Hier lohnt es sich den Spieltrieb
auszuleben, die neuen Möglichkeiten in kleinen Prototypen und "Wegwerfprojekten"
auszuprobieren. Aus dem einfachen Grund, dass man das, was man nicht kennt, auch
nicht sinnvoll einsetzen kann.