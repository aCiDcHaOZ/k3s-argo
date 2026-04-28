# Inleiding
In deze markdown file vertellen we over de totstandkoming van het project "orange-kuma" ....

---

# Voorbereiding
Voor de infrastructuur werden we verplicht om een Kubernetes cluster op te zetten.  
Als (virtuele) hardwarekeuze om de boel te hosten, viel de keuze op de Proxmox omgeving die ter beschikking is gesteld door de Hanze Hogeschool.

---

# Gitea actions: Uptime kuma klonen
Het klonen van de playbook duurde erg lang omdat met elke action de image van `ubuntu:latest` gepulled moet worden van docker.io.  

Om dit sneller te laten verlopen hebben we een `ubuntu:latest` image in onze eigen repo gezet. Dit is gedaan met een tijdelijke Skopeo pod. Deze pod had de taak om de image van docker.io te pushen naar onze eigen repository.

---

# Hostname resolving
Dit was een hoofdpijn dossier omdat containers van andere namespaces niet het IP van `gitea.local` konden resolven.  

Dit is uiteindelijk gemaakt door een override te maken in de CoreDNS configuratie van k3s.

Daarna kwam het probleem dat Gitea buiten het cluster HTTPS gebruikte en binnen het netwerk HTTP. Dit komt omdat er gewerkt wordt met een split horizon DNS:

- Buiten het netwerk verwijst `gitea.local` naar `10.24.43.100` ← adres van de k3s server  
- Binnen het netwerk verwijst `gitea.local` naar `10.43.109.173` ← Intern adres van k3s  

De ingress van Traefik zet http om naar https. Omdat ik in ieder geval binnen k3s native http wilde gebruiken om certificaatfouten tegen te gaan, wilde de gitea runners weer niet werken met http tenzij ze daarvoor expliciet geconfigureerd werden.  

Na lang troubleshooten kwam ik er achter wat er mis ging:

De gita-runner applicatie en gitea konden niet met elkaar communiceren doordat de runner eigenlijk Docker-in-docker (dind) is. Door de extra abstractielaag verwees localhost naar een intern IP-adres dat door Docker network is uitgegeven en niet het IP-adres wat de pod heeft.

**Foutmelding: Cannot connect to the Docker daemon at tcp://localhost:2375**

---

# 'Eenvoudige' Kloon maken van Uptime kuma
In eerste instantie wilde ik hier een Gitea workflow voor maken, Echter zijn we verplicht met Semaphore te werken. Semaphore is hier ook een wat elegantere oplossing voor in plaats van hacky een workflow laten runnen.

---

# Werken met Semaphore en Skopeo
Semaphore is out-of-the-box niet ingericht om docker images te kopieren of te plakken van verschillende repositories.  

Hier is een andere tool voor: Skopeo (niet te verwarren met scapino).

Om skopeo te draaien is daar een nieuwe job-runner manifest voor aangemaakt in Github die opgehaald wordt door ArgoCD.  

Het idee is dat Semaphore opdracht geeft aan skopeo om uptime kuma te pullen van docker.io en daarna te pushen naar onze eigen Gitea repo.

---

# Custom image voor Semaphore
Hiervoor is de kubernetes python library nodig die natuurlijk niet standaard in Semaphore zit.  

Om dat probleem te verhelpen moest er een custom image gebouwd worden. Dit gebeurt in de repo semaphore-plus:

- De semaphore image wordt gepulled  
- Met pip wordt de kubernetes module toegevoegd
- Het manifest van Semaphore wordt aangepast: Pull nu onze eigen image
- ArgoCD regelt de redeployment van Semaphore

Ook hier liep ik tegen DNS problemen aan. De vorige keer hebben we de DNS problemen verholpen die binnen een pod kunnen voorkomen en die op de windows pc voorkomen. Na een DNS override op de k3s servers zelf was de service nog steeds niet benaderbaar vanaf de k3s server. Het volgende plan was om een nieuwe service te maken die port 30080 vanaf het IP-adres van de k3s servers doorstuurt naar gitea.local:3000. Een laatste aanpassing aan het manifest van gitea; de ROOT_URL moest aangepast worden naar http://gitea.local:30080 en voila! Semaphore draait onze zelfgemaakte image. een uur minder maar een deployment rijker.

---

# Uptime kuma klonen naar eigen repo

Nu het kopieren van de uptime kuma image naar onze eigen gitea is gelukt, willen de hem graag koppelen aan een repo. Daar is een API call voor die ik in eerste instantie wilde aanspreken met curl maar curl is niet geinstalleerd op de skopeo image. Een uitwijking naar wget hielp ook niet. Ik ging het dichter bij de bron zoeken door een wget uit te voeren binnen de pod. Daar bleek dat gitea.local weer niet geresolved werd. Dit is inmiddels de derde keer dat ik me aan deze steen stoot. In het vervolg moet ik maar de defult entry gitea.gitea.svc.cluster.local gebruiken binnen het cluster. Nadat de problemen met resolve geresolved waren liep ik tegen een nieuwe 'uitdaging'. De API-call die ik nodig heb om de package te koppelen aan een repo, komt niet voor in deze gitea versie.

---

# Gitea updaten

Blijkbaar heb ik een pre-historische versie van Gitea geinstalleerd: Ik draai nu op 1.21 (april 2024 trouwens). De API-call die ik nodig heb zit in versie 1.24 en de laatste gitea versie is 1.26.1. Ik voelde me moedig en wilde meteen migreren naar 1.26.1. Niet zo moedig dat ik dit zonder een snapshot wilde proberen. Snel de foto gemaakt en het versienummer gespecificeerd in het manifest. Na een refresh en een nieuwe pod kon ik niet meer inloggen. Misschien moet ik het wat subtieler doen en netjes het upgrade path volgen: 1.21 -> 1.22.6 -> 1.23.8 -> 1.24.7 en misschien daarna 1.24.7 -> 1.25.5 -> 1.26.1. Tijdens het maken van de snapshot viel het me op dat longhorn ook drie major versies achterloopt. Ik besluit dat te laten voor wat het is.

---

# De eerste deployment

Nu we een werkende Orange Kuma (OK) image hebben, willen we nadenken over hoe we die precies willen gaan deployen. De image moet meerdere keren gedeployed kunnen worden, daarom is een unieke naam noodzakelijk. Ik kies ervoor om de fictieve klantnummer deel uit te laten maken van de deployment. Omdat ik nu nog geen willekeurige klantnummers heb, besluit ik een hard-coded nummer te gebruiken. Deployment wil ik laten doen door middel van een playbook die een manifest pusht naar de zelf-gehoste Gitea repository. ArgoCD neemt het vanaf daar over. Na een succesvolle deploy moet de pod nog bereikbaar worden buiten de kubernetes cluster. Hier zijn twee methodes voor ingress of nodeport. Omdat voor elke nieuwe ingress het lokale hosts bestand gewijzigd moet worden van de computer waarop de pod bereikt moet worden, gaan we dat niet doen. Met nodeport forwarden we een port vanaf het publieke IP-adres van een k3s server naar de pod.

De logische volgende stap was om een playbook te maken die een klantnummer kan meegeven aan de deployment. Ik heb gekozen voor een opmaak als KN-XXX waar XXX drie getallen zijn. Het klantnummer wordt, bij gebrek aan beter ook gebruikt als portnummer met de som (KN / 100) + 31000. Zo krijgt de pod met klantnummer 123 port 31023 toegewezen. Er is een redelijke kans op collisions en het klantnummer mag niet hoger zijn dan 2767 omdat er geen portnummers hoger zijn dan 32767. Door gebruik te maken van klantnummer is het eenvoudig om hetzelfde nummer te gebruiken als namespace.
Met een werkende deploy playbook op klantnummer is een playbook die de boel opruimt snel gemaakt. Het verwijderen van een deployment was wel een delicaat werkje omdat ArgoCD zo aggressief redeployments doet.

---

# orange-kuma-manager

Omdat ik semaphore maar een saaie manier van deployen vind en lang niet alle informatie weergeeft wat de 'afdeling verkoop' zou willen zien, leek het me een goed idee om de opdracht nog moeilijker te maken. Dit doe ik door een website te maken waar alle deployments te managen zijn. De website start een playbook met de Ansible API zodat de basis nog steeds gedaan wordt door middel van playbooks.

