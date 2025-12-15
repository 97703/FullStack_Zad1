<img src="https://github.com/97703/DockerLab3/blob/main/rysunki/loga_weii.png?raw=true" style="width: 40%; height: 40%" />

> **Programowanie Full-Stack w Chmurze Obliczeniowej**

      dr inż. Sławomir Wojciech Przyłucki

<br>
Termin zajęć:

      środa, godz. 11:30,

Imię i nazwisko:

      Paweł Pieczykolan,
      II rok studiów magisterskich, WOiSI 2.3.

# Wstęp
<p align="justify">Celem było przygotowanie kompletnego klastra Kubernetes w oparciu o Minikube, który miał składać się z jednego węzła głównego oraz trzech węzłów roboczych oznaczonych jako A, B i C. Klaster miał zostać uruchomiony z wykorzystaniem sterownika Docker oraz pluginu sieciowego CNI Calico, co zapewniało możliwość późniejszej konfiguracji polityk sieciowych.</p>

<p align="justify">Pierwszym krokiem było utworzenie zestawu plików manifestów YAML, które opisywały obiekty środowiska Kubernetes zgodnie z przyjętymi założeniami. W przestrzeni nazw frontend należało przygotować Deployment o nazwie frontend, bazujący na obrazie nginx i posiadający trzy repliki. Pod-y tego Deploymentu miały być uruchamiane na dowolnym węźle, z wyjątkiem węzłów, na których działały komponenty backend oraz baza danych MySQL. W przestrzeni nazw backend należało przygotować Deployment o nazwie backend, również oparty na obrazie nginx, ale posiadający tylko jedną replikę.</p>

<p align="justify">Kolejnym wymaganiem było utworzenie Poda my-sql w przestrzeni nazw backend, korzystającego z obrazu MySQL. Dla każdego z komponentów należało przygotować odpowiednie obiekty Service: dla frontend typu NodePort, a dla backendu i MySQL typu ClusterIP. Dzięki temu możliwe było zapewnienie właściwej komunikacji pomiędzy warstwami aplikacji oraz wystawienie frontendu na zewnątrz klastra.</p>

<p align="justify">Istotnym elementem zadania było opracowanie polityki sieciowej (NetworkPolicy), która miała ograniczyć komunikację pomiędzy komponentami. Zgodnie z założeniami, z Podów frontendu można było łączyć się wyłącznie z backendem na porcie 80, natomiast backend miał możliwość komunikacji z bazą danych MySQL, ale tylko na porcie 3306. W ten sposób zapewniono kontrolę nad przepływem danych i bezpieczeństwo aplikacji. Dodatkowo należało wprowadzić ograniczenia zasobów dla przestrzeni nazw – w frontend maksymalnie 10 Podów, 1 CPU i 1,5 Gi pamięci RAM, a w backend maksymalnie 3 Pody, 1 CPU i 1,0 Gi pamięci RAM.</p>

<p align="justify">Ostatnim wymaganiem było utworzenie autoskalera HPA dla Deploymentu frontend, który miał umożliwiać skalowanie liczby replik w zależności od obciążenia, ale jednocześnie nie przekraczać limitów zasobów przypisanych do przestrzeni nazw. Wszystkie parametry, które nie zostały jednoznacznie określone w treści zadania, należało dobrać samodzielnie i uzasadnić ich wybór. Na końcu należało zaproponować test, który potwierdzi poprawność konfiguracji przy obciążeniu Deploymentu frontend, a wyniki działania poleceń i konfiguracji przedstawić w sprawozdaniu.</p>

<p align="justify">Zadanie zostało zrealizowane w środowisku Windows 11 PRO z wykorzystaniem WSL. W ramach WSL uruchomiono system Ubuntu 24.04.1 LTS, który stanowił bazę do instalacji i konfiguracji Minikube, klastra Kubernetes oraz wszystkich komponentów opisanych w zadaniu.</p>

# 1. Utworzenie klastra Minikube z czterema węzłami
<p align="justify">Pierwszym krokiem w realizacji zadania było całkowite usunięcie poprzedniego klastra Minikube. W terminalu uruchomiono polecenie minikube delete, które usunęło kontener klastra, pliki konfiguracyjne oraz wpisy z Docker Engine. Dzięki temu możliwe było rozpoczęcie konfiguracji od zera, bez ryzyka konfliktów z wcześniejszymi ustawieniami.</p>


<p align="justify">Następnie uruchomiono nowy klaster Minikube z czterema węzłami, wykorzystując sterownik Docker oraz plugin sieciowy Calico. Polecenie:</p>

      minikube start --nodes=4 --cni=calico --driver=docker
  
<p align="justify">zainicjowało proces tworzenia klastra. W pierwszej kolejności utworzony został węzeł główny, a następnie trzy węzły robocze: <code>minikube-m02</code>, <code>minikube-m03</code> i <code>minikube-m04</code>. Każdy z nich został skonfigurowany z 2 CPU i 3072 MB pamięci RAM.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys1.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 1. Usunięcie poprzedniego klastra i utworzenie nowego</i>
</p>

<p align="justify">Po zakończeniu procesu tworzenia kontenerów, Minikube automatycznie skonfigurował komponenty Kubernetes w wersji 1.34.0. Polecenie
  
    kubectl get nodes

potwierdziło, że wszystkie cztery węzły są aktywne i mają status <code>Ready</code>.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys2.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 2. Weryfikacja aktywności węzłów</i>
</p>

<p align="justify">W kolejnym kroku nadano aliasy węzłom roboczym, aby ułatwić przypisywanie zasobów. Węzłowi <code>minikube-m02</code> przypisano etykietę <code>node=A</code>, <code>minikube-m03</code> otrzymał <code>node=B</code>, a <code>minikube-m04</code> <code>node=C</code>. Dzięki temu możliwe było precyzyjne sterowanie rozmieszczeniem Podów w dalszych etapach konfiguracji. Dodatkowo utworzono dwie przestrzenie nazw: frontend i backend, które odpowiadały za logiczne rozdzielenie komponentów aplikacji.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys3.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 3. Nadanie aliasów węzłom oraz utworzenie wymaganych przestrzeni nazw</i>
</p>

# 2. Konfiguracja limitów zasobów

<p align="justify">W drugim etapie zadania skonfigurowano ResourceQuota dla przestrzeni nazw frontend oraz backend. Celem było ograniczenie liczby Podów oraz dostępnych zasobów CPU i pamięci RAM, aby zapewnić kontrolę nad wykorzystaniem klastra i uniknąć sytuacji, w której jeden komponent aplikacji zużywałby wszystkie dostępne zasoby. Dzięki temu możliwe było zachowanie równowagi pomiędzy warstwami aplikacji i zagwarantowanie stabilności działania całego środowiska.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys4.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 4. Zawartość pliku front-quota.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: frontend-quota</code> – nazwa zasobu dla przestrzeni frontend.

<code>namespace: frontend</code> – wskazuje, że quota dotyczy przestrzeni nazw frontend.

<code>hard:</code> – oznacza twarde limity, których nie można przekroczyć.

<code>pods: "10"</code> – maksymalna liczba Podów w przestrzeni frontend to 10.

<code>requests.cpu: "1000m"</code> – łączna ilość CPU dostępna dla Podów to 1 pełny rdzeń.

<code>requests.memory: 1.5Gi</code> – maksymalna ilość pamięci RAM dostępna dla Podów to 1,5 GiB.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys5.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 5. Zawartość pliku back-quota.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: backend-quota</code> – nazwa zasobu dla przestrzeni backend.

<code>namespace: backend</code> – quota dotyczy przestrzeni nazw backend.

<code>hard:</code> – twarde limity zasobów.

<code>pods: "3"</code> – maksymalna liczba Podów w przestrzeni backend to 3.

<code>requests.cpu: "1000m"</code> – łączna ilość CPU dostępna dla Podów to 1 rdzeń.

<code>requests.memory: 1Gi</code> – maksymalna ilość pamięci RAM to 1 GiB.

<p align="justify">W konfiguracji użyto sekcji hard, ponieważ w tym zadaniu celem było ustalenie nieprzekraczalnych granic dla liczby Podów oraz zasobów CPU i pamięci. Twarde limity są najprostszym i najbardziej skutecznym sposobem kontroli – jeśli aplikacja spróbuje uruchomić więcej Podów lub zażąda większej ilości zasobów niż przewidziano, Kubernetes odrzuci takie żądanie. Dzięki temu administrator ma pewność, że przestrzeń nazw nie przekroczy przydzielonych zasobów i nie wpłynie negatywnie na inne komponenty klastra.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys6.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 6. Utworzenie obiektów typu ResourceQuota</i>
</p>

<p align="justify">Dodatkowo, w przestrzeni nazw backend utworzono obiekt typu LimitRange, który definiuje domyślne wartości zasobów dla kontenerów uruchamianych w tej przestrzeni. Celem było zapewnienie, że każdy nowy kontener będzie miał przypisane minimalne wartości CPU i pamięci, nawet jeśli nie zostały one jawnie określone w jego definicji. Dzięki temu uniknięto sytuacji, w której kontenery uruchamiane byłyby bez żadnych ograniczeń, co mogłoby prowadzić do niekontrolowanego zużycia zasobów.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys7.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 7. Zawartość pliku limit.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: backend-limits</code> – nazwa obiektu LimitRange.

<code>namespace: backend</code> – przestrzeń nazw, której dotyczy limit.

<code>limits:</code> – lista limitów zasobów.

<code>type: Container</code> – limit dotyczy pojedynczych kontenerów.

<code>defaultRequest:</code> – domyślne wartości zasobów, które zostaną przypisane kontenerowi, jeśli nie określono ich w jego definicji.

<code>cpu: 100m</code> – domyślny przydział CPU to 0.1 rdzenia.

<code>memory: 128Mi</code> – domyślny przydział pamięci RAM to 128 MiB.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys8.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 8. Utworzenie obiektu LimitRange</i>
</p>

<p align="justify">Dzięki zastosowaniu LimitRange, każdy kontener w przestrzeni backend ma zagwarantowane minimalne zasoby, co pozwala lepiej planować wykorzystanie klastra i zapobiegać przeciążeniom. LimitRange działa komplementarnie z ResourceQuota – quota ogranicza sumaryczne zużycie, a limit ustala wartości domyślne dla pojedynczych kontenerów.</p>

# 3. Utworzenie Poda MySQL oraz deploymentów backend i frontend

<p align="justify">Na tym etapie zbudowano trzy kluczowe elementy aplikacji: Pod bazy danych MySQL (w przestrzeni nazw backend), Deployment backend (1 replika na węźle B) oraz Deployment frontend (3 repliki na węźle A). Celem było rozdzielenie warstw aplikacji na różne węzły, utrzymanie spójnej komunikacji i zgodności z ograniczeniami zasobów oraz politykami sieciowymi. Rozmieszczenie Podów wymuszono etykietami węzłów (A, B, C) i nodeSelector, a dostępność sprawdzono poprzez proste healthchecki na głównym endpointcie serwera.</p>

<p align="justify">Wdrożenie Poda MySQL w backendzie pozwoliło zrealizować warstwę danych, a wybór Deploymentów dla frontend i backend zapewnił deklaratywne zarządzanie replikami i aktualizacjami. Readiness i liveness na ścieżce “/”: w tym scenariuszu wystarczało potwierdzenie odpowiedzi serwera HTTP. Wymagane requests CPU/RAM wpisują się w zdefiniowane ResourceQuota i LimitRange, gwarantując poprawne przyjęcie obiektów przez API oraz przewidywalne planowanie.</p>

## 3.1. Pod MySQL (plik mysql.yaml)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys9.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 9. Zawartość pliku mysql.yaml</i>
</p

<p align="justify">Objaśnienie:</p>

<code>name: my-sql</code> – nazwa Poda, używana do identyfikacji.

<code>namespace: backend</code> – przestrzeń nazw, w której Pod ma zostać utworzony.

<code>labels:</code> – etykiety służące do selekcji i grupowania obiektów.

  <code>app: my-sql</code> – etykieta logicznie identyfikująca aplikację (warstwę danych).

<code>nodeSelector:</code> – wymuszenie umieszczenia Poda na węźle o wskazanej etykiecie.

<code>node: C</code> – węzeł klastra oznaczony etykietą C, na którym Pod ma działać.

<code>name: mysql</code> – nazwa kontenera wewnątrz Poda.

<code>image: mysql:8.0</code> – obraz kontenera MySQL w wersji 8.0.

<code>env:</code> – zmienne środowiskowe przekazywane do kontenera.

  <code>name: MYSQL_ROOT_PASSWORD</code> – nazwa zmiennej definiującej hasło użytkownika root.

  <code>value: root</code> – wartość hasła dla użytkownika root (na potrzeby testów).

<code>containerPort: 3306</code> – port MySQL dostępny w kontenerze.

<code>resources:</code> – deklaracja zapotrzebowania na zasoby przez kontener.

<code>requests:</code> – minimalne gwarantowane zasoby, wymagane przez ResourceQuota.

<code>cpu: 200m</code> – minimalny przydział CPU to 0.2 rdzenia.

<code>memory: 256Mi</code> – minimalny przydział pamięci RAM to 256 MiB.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys10.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 10. Utworzenie Poda my-sql w przestrzeni nazw Backend</i>
</p

<p align="justify">W Podzie <code>MySQL</code> dodano zmienną środowiskową <code>MYSQL_ROOT_PASSWORD</code> z wartością root, aby umożliwić uruchomienie serwera bazy danych z podstawową konfiguracją. Obraz <code>MySQL</code>code> wymaga ustawienia hasła dla użytkownika administracyjnego, w przeciwnym razie kontener nie wystartuje. Wprowadzenie tej zmiennej pozwoliło na szybkie uruchomienie bazy w środowisku testowym i sprawdzenie poprawności komunikacji między komponentami. Wartość została dobrana prosto i czytelnie, ponieważ celem było testowanie polityk sieciowych, a nie produkcyjne zabezpieczenie danych.</p>

## 3.2. Deployment backend (plik backend-dep.yaml)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys11.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 11. Zawartość pliku backend-dep.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: backend</code> – nazwa Deploymentu.

<code>namespace: backend</code> – przestrzeń nazw dla tego Deploymentu.

<code>replicas: 1</code> – liczba replik (Podów) uruchamianych przez Deployment.

<code>selector:</code> – reguły selekcji Podów zarządzanych przez ten Deployment.

<code>matchLabels:</code> – etykiety, które muszą mieć Pody, aby były zarządzane.

  <code>app:</code> backend – wartość etykiety dopasowującej Pody.

<code>labels:</code> – etykiety nadawane Podom z tego szablonu.

  <code>app: backend</code> – etykieta Podów zgodna z selektorem.

<code>nodeSelector:</code> – wymuszenie umieszczania Podów na wybranym węźle.

<code>node: B</code> – węzeł klastra oznaczony etykietą B dla backendu.

<code>name: nginx</code> – nazwa kontenera aplikacyjnego.

<code>image: nginx</code> – obraz kontenera z serwerem Nginx.

<code>containerPort: 80</code> – port HTTP serwera backendu.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys12.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 12. Utworzenie obiektu Deployment o nazwie Backend w przestrzeni nazw Backend</i>
</p>

## 3.3. Deployment frontend (plik front-dep.yaml)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys13.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 13. Zawartość pliku front-dep.yaml</i>
</p

<p align="justify">Objaśnienie:</p>

<code>name: frontend</code> – nazwa Deploymentu frontendu.

<code>namespace: frontend</code> – przestrzeń nazw dla frontendu.

<code>replicas: 3</code> – liczba replik Podów frontendu.

<code>selector:</code> – reguły selekcyjne dla Podów zarządzanych przez Deployment.

<code>matchLabels:</code> – etykiety, które muszą mieć Pody.

  <code>app: frontend</code> – wartość etykiety dopasowującej Pody.

<code>labels:</code> – etykiety przypisane Podom z szablonu.

  <code>app: frontend</code> – etykieta Podów zgodna z selektorem.

<code>nodeSelector:</code> – wymuszenie umieszczania Podów frontendu na wskazanym węźle.

<code>node: A</code> – węzeł klastra oznaczony etykietą A dla frontendu.

<code>affinity:</code> – reguły rozmieszczania Podów względem innych Podów.

  <code>podAntiAffinity:</code> – zakaz współlokacji z wybranymi Podami na tym samym hoście.

  <code>requiredDuringSchedulingIgnoredDuringExecution:</code> – twardy zakaz przy planowaniu (ignorowany po uruchomieniu).

  <code>labelSelector:</code> – selektor wskazujący Pody, względem których obowiązuje antyafinity.

<code>matchLabels:</code> – etykiety definiujące grupę docelowych Podów.

<code>app: backend</code> – antyafinity dotyczy Podów oznaczonych jako backend.

<code>topologyKey: kubernetes.io/hostname</code> – klucz topologii: odseparuj na poziomie hosta.

<code>name: nginx</code> – nazwa kontenera serwującego frontend.

<code>image: nginx</code> – obraz kontenera Nginx.

<code>ports:</code> – lista portów wystawianych przez kontener.

<code>containerPort: 80</code> – port HTTP frontendu.

<code>requests:</code> – minimalne gwarantowane zasoby wymagane przez Quota.

  <code>cpu: 100m</code> – minimalny przydział CPU to 0.1 rdzenia.

  <code>memory: 128Mi</code> – minimalny przydział pamięci RAM to 128 MiB.

<code>readinessProbe:</code> – sonda gotowości (czy Pod może przyjmować ruch).

<code>httpGet:</code> – sprawdzenie przez żądanie HTTP GET.

<code>path: /</code> – ścieżka sprawdzająca stronę główną serwera.

<code>port: 80</code> – port HTTP, na który kierowane jest sprawdzenie.

<code>livenessProbe:</code> – sonda żywotności (czy proces żyje).

<code>httpGet:</code> – sprawdzenie przez żądanie HTTP GET.

<code>path: /</code> – ścieżka główna serwera.

<code>port: 80</code> – port HTTP sondy żywotności.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys14.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 14. Utworzenie obiektu Deployment o nazwie Frontend w przestrzeni nazw Frontend</i>
</p>

<p align="justify">Reguły <code>affinity</code> zostały wprowadzone, aby uniknąć pomyłki przy rozmieszczaniu Podów w klastrze i zapewnić separację warstw aplikacji. W szczególności zastosowano <code>podAntiAffinity</code>, które wymusza, aby Pody frontendu nie były uruchamiane na tym samym hoście co Pody backendu. Dzięki temu uniknięto sytuacji, w której oba komponenty działałyby na jednym węźle, co mogłoby prowadzić do przeciążenia zasobów i utrudniać testowanie polityk sieciowych. Rozdzielenie warstw zwiększa stabilność systemu i ułatwia zarządzanie ruchem w aplikacji.</p>

## 3.4. Uzupełnione parametry probe (plik front-dep.yaml)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys15.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 15. Healthcheck obiektu Frontend typu Deployment</i>
</p

<p align="justify">Objaśnienie:</p>

<code>readinessProbe.httpGet.scheme: HTTP</code> – protokół sondy gotowości.

<code>readinessProbe.initialDelaySeconds: 5</code> – opóźnienie startu sondy gotowości.

<code>readinessProbe.periodSeconds: 10</code> – częstotliwość sprawdzania gotowości.

<code>readinessProbe.timeoutSeconds: 2</code> – maksymalny czas oczekiwania na odpowiedź.

<code>readinessProbe.successThreshold: 1</code> – liczba wymaganych sukcesów pod rząd.

<code>readinessProbe.failureThreshold: 3</code> – liczba tolerowanych porażek zanim uzna NotReady.

<code>livenessProbe.httpGet.scheme: HTTP</code> – protokół sondy żywotności.

<code>livenessProbe.initialDelaySeconds: 15</code> – opóźnienie startu sondy żywotności.

<code>livenessProbe.periodSeconds: 20</code> – częstotliwość sprawdzania żywotności.

<code>livenessProbe.timeoutSeconds: 2</code> – maksymalny czas oczekiwania na odpowiedź.

<code>livenessProbe.successThreshold: 1</code> – liczba wymaganych sukcesów.

<code>livenessProbe.failureThreshold: 3</code> – liczba porażek skutkująca restartem kontenera.

<p align="justify">Sondy <code>readinessProbe</code> i <code>livenessProbe</code> zostały dodane, aby Kubernetes mógł automatycznie monitorować stan Podów. <code>ReadinessProbe</code> sprawdza, czy Pod jest gotowy do obsługi ruchu i dopiero wtedy oznacza go jako Ready, co zapobiega kierowaniu żądań do niedziałających kontenerów. <code>LivenessProbe</code> natomiast kontroluje, czy proces w kontenerze działa poprawnie – w przypadku awarii kubelet automatycznie restartuje Pod. Dzięki tym mechanizmom aplikacja jest bardziej odporna na błędy i zapewnia ciągłą dostępność usług.</p>

# 4. Utworzenie usług sieciowych

<p align="justify">W celu zapewnienia komunikacji pomiędzy komponentami aplikacji oraz umożliwienia dostępu do nich z zewnątrz lub wewnątrz klastra, utworzono trzy obiekty typu <code>Service</code>code>. Każdy z nich odpowiadał za ekspozycję konkretnego komponentu: frontend, backend oraz <code>MySQL</code>code>. Zastosowano dwa typy usług – <code>NodePort</code>code> dla frontendu, aby umożliwić dostęp spoza klastra, oraz <code>ClusterIP</code>code> dla backendu i bazy danych, ponieważ te komponenty miały komunikować się wyłącznie wewnętrznie.</p>

## 4.1. Usługa dla Backendu (plik backend-svc.yaml)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys16.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 16. Zawartość pliku backend-svc.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: backend-svc</code> – nazwa usługi dla backendu.

<code>namespace: backend</code> – przestrzeń nazw, w której działa backend.

<code>type: ClusterIP</code> – typ usługi dostępnej tylko wewnątrz klastra.

<code>selector:</code> – etykieta wskazująca, do których Podów kierować ruch.

  <code>app: backend</code> – etykieta Podów backendu.

<code>ports:</code> – lista portów obsługiwanych przez usługę.

  <code>port: 80</code> – port, na którym usługa nasłuchuje.

  <code>targetPort: 80</code> – port w kontenerze, do którego przekazywany jest ruch.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys17.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 17. Utworzenie obiektu Service dla Frontendu</i>
</p>

## 4.2. Usługa dla MySQL (plik mysql-svc.yaml)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys18.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 18. Zawartość pliku mysql-svc.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: mysql-svc</code> – nazwa usługi dla bazy danych.

<code>namespace: backend</code> – przestrzeń nazw, w której działa MySQL.

  <code>type: ClusterIP</code> – usługa dostępna tylko wewnątrz klastra.

<code>selector:</code> – etykieta wskazująca na Pody MySQL.

<code>app: my-sql</code> – etykieta Poda bazy danych.

<code>ports:</code> – lista portów obsługiwanych przez usługę.

  <code>port: 3306</code> – port, na którym usługa nasłuchuje.

  <code>targetPort: 3306</code> – port w kontenerze MySQL, do którego kierowany jest ruch.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys19.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 19. Utworzenie obiektu Service dla Mysql</i>
</p>

## 4.3. Usługa dla dla Frontendu (front-svc.yaml)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys20.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 20. Zawartość pliku frontend-svc.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: frontend-svc</code> – nazwa usługi dla frontendu.

<code>namespace: frontend</code> – przestrzeń nazw, w której działa frontend.

<code>type: NodePort</code> – typ usługi umożliwiający dostęp spoza klastra.

<code>app: frontend</code> – etykieta Podów frontendu.

<code>ports:</code> – lista portów obsługiwanych przez usługę.

  <code>port: 80</code> – port, na którym usługa nasłuchuje.

  <code>targetPort: 80</code> – port w kontenerze, do którego kierowany jest ruch.

  <code>nodePort: 30080</code> – port na węźle, przez który dostępna jest aplikacja z zewnątrz.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys21.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 21. Utworzenie obiektu Service dla Frontendu</i>
</p>

# 5. Utworzenie polityk sieciowych

<p align="justify">W celu kontrolowania komunikacji pomiędzy komponentami aplikacji, utworzono zestaw polityk sieciowych w <code>Kubernetes</code>. Zamiast definiować każdą politykę w osobnym pliku, zastosowano podejście bardziej zorganizowane — wszystkie trzy polityki zostały zapisane w jednym pliku YAML. Dzięki temu możliwe było jednoczesne wdrożenie reguł dla frontendu, backendu oraz bazy danych <code>MySQL</code>, co uprościło zarządzanie konfiguracją i zapewniło spójność reguł.</p>

Polityki zostały zaprojektowane zgodnie z wymaganiami zadania:

→ Frontend może komunikować się tylko z backendem na porcie 80.

→ Backend może komunikować się z frontendem na porcie 80 oraz z MySQL na porcie 3306.

→ MySQL akceptuje połączenia tylko z backendu na porcie 3306.</p>

## 5.1. Polityka dla Frontendu (frontend-egress-policy)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys22.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 22. Fragment dla Frontend-egress-policy z pliku networkpol.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: frontend-egress-policy</code> – nazwa polityki dla frontendu.

<code>namespace: frontend</code> – przestrzeń nazw, w której działa frontend.

<code>app: frontend</code> – polityka dotyczy Podów z etykietą app=frontend.

<code>policyTypes:</code> – lista typów ruchu kontrolowanych przez politykę.

<code>Egress</code> – polityka dotyczy ruchu wychodzącego.

<code>egress:</code> – reguły dla ruchu wychodzącego.

  <code>to:</code> – dozwolone cele ruchu.

  <code>namespaceSelector:</code> – selekcja przestrzeni nazw celu.

  <code>kubernetes.io/metadata.name: backend</code> – docelowy namespace to backend.

  <code>podSelector:</code> – selekcja Podów w docelowym namespace.

  <code>matchLabels:</code> – dopasowanie po etykietach.

<code>app: backend</code> – docelowe Pody to backend.

<code>ports:</code> – lista dozwolonych portów.

  <code>protocol: TCP</code> – dozwolony protokół.

  <code>port: 80</code> – dozwolony port HTTP.

<p align="justify">Frontend pełni rolę warstwy prezentacji i powinien komunikować się wyłącznie z backendem.

▶ Egress: wybrano tylko ruch wychodzący, ponieważ frontend nie musi przyjmować połączeń z innych Podów – jego zadaniem jest wysyłanie żądań do backendu.

▶ Port 80/TCP: to standardowy port HTTP, na którym backend (nginx) nasłuchuje. Dzięki temu frontend może przesyłać żądania do backendu w sposób zgodny z architekturą aplikacji.

▶ NamespaceSelector + PodSelector: ograniczenie do Podów backendu w przestrzeni backend zapewnia, że frontend nie wyśle ruchu do innych usług w klastrze, co zwiększa bezpieczeństwo i izolację.</p>

## 5.2. Polityka dla Backendu (backend-ingress-egress-policy)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys23.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 23. Fragment dla Backend-ingress-egress-policy pliku networkpol.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: backend-ingress-egress-policy</code> – nazwa polityki dla backendu.

<code>namespace: backend</code> – przestrzeń nazw, w której działa backend.

<code>app: backend</code> – polityka dotyczy Podów z etykietą app=backend.

<code>policyTypes:</code> – lista typów ruchu kontrolowanych przez politykę.

<code>Ingress</code> – polityka dotyczy ruchu przychodzącego.

<code>Egress</code> – polityka dotyczy ruchu wychodzącego.

<code>ingress:</code> – reguły dla ruchu przychodzącego.

  <code>from:</code> – dozwolone źródła ruchu.

<code>namespaceSelector:</code> – selekcja przestrzeni nazw źródła.

  <code>kubernetes.io/metadata.name: frontend</code> – źródłowy namespace to frontend.

  <code>podSelector:</code> – selekcja Podów w źródłowym namespace.

  <code>matchLabels:</code> – dopasowanie po etykietach.

  <code>app: frontend</code> – dozwolone źródłowe Pody to frontend.

<code>ports:</code> – lista dozwolonych portów.

  <code>protocol: TCP</code> – dozwolony protokół.

  <code>port: 80</code> – dozwolony port HTTP.

<code>egress:</code> – reguły dla ruchu wychodzącego.

<code>to:</code> – dozwolone cele ruchu.

  <code>app: my-sql</code> – docelowe Pody to MySQL.

<code>ports:</code> – lista dozwolonych portów.

<code>protocol: TCP</code> – dozwolony protokół.

<code>port: 3306</code> – dozwolony port MySQL.

<code>to:</code> – dodatkowy cel ruchu.
  
  <code>namespaceSelector:</code> – selekcja przestrzeni nazw celu.

  <code>kubernetes.io/metadata.name: frontend</code> – docelowy namespace to frontend.

<code>app: frontend</code> – docelowe Pody to frontend.

<code>ports:</code> – lista dozwolonych portów.

  <code>protocol: TCP</code> – dozwolony protokół.

  <code>port: 80</code> – dozwolony port HTTP.

<p align="justify">Backend jest warstwą logiki aplikacji, która musi przyjmować żądania od frontendu i jednocześnie komunikować się z bazą danych oraz z frontendem.

▶ Ingress: konieczne, aby backend mógł odbierać ruch od frontendu. Bez tego frontend nie miałby możliwości przesyłania żądań.

▶ Egress: backend musi wysyłać zapytania do bazy danych MySQL (port 3306) oraz w niektórych scenariuszach odpowiadać do frontendu (port 80).

▶ Port 80/TCP (Ingress): backend nasłuchuje na porcie HTTP, aby obsługiwać żądania z frontendu.

▶ Port 3306/TCP (Egress): to standardowy port MySQL, wymagany do komunikacji z bazą danych.

▶ NamespaceSelector + PodSelector: reguły precyzyjnie wskazują, że backend może rozmawiać tylko z frontendem i MySQL, co eliminuje ryzyko nieautoryzowanych połączeń z innymi Podami.</p>

## 5.3. Polityka dla Mysql (mysql-ingress-policy)

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys24.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 24. Fragment dla Mysql-ingress-policy pliku networkpol.yaml</i>
</p>

<p align="justify">Objaśnienie:</p>

<code>name: mysql-ingress-policy</code> – nazwa polityki dla MySQL.

<code>namespace: backend</code> – przestrzeń nazw, w której działa MySQL.

<code>app: my-sql</code> – polityka dotyczy Podów z etykietą app=my-sql.

<code>policyTypes:</code> – lista typów ruchu kontrolowanych przez politykę.

<code>Ingress</code> – polityka dotyczy ruchu przychodzącego.

<code>ingress:</code> – reguły dla ruchu przychodzącego.

<code>from:</code> – dozwolone źródła ruchu.

  <code>app: backend</code>– dozwolone źródłowe Pody to backend.

<code>ports: – lista dozwolonych portów.</code>

  <code>protocol: TCP</code> – dozwolony protokół.

  <code>port: 3306</code> – dozwolony port bazy MySQL.

<p align="justify">MySQL jest warstwą danych i powinien być dostępny tylko dla backendu.

▶ Ingress: wybrano tylko ruch przychodzący, ponieważ baza danych nie powinna sama inicjować połączeń – jej rolą jest odpowiadanie na zapytania.

▶ Port 3306/TCP: to domyślny port MySQL, na którym serwer nasłuchuje. Ograniczenie do tego portu zapewnia, że backend może korzystać z bazy, ale nie ma możliwości używania innych portów.

▶ PodSelector (app=backend): tylko Pody backendu mogą łączyć się z bazą. Dzięki temu frontend ani inne komponenty nie mają bezpośredniego dostępu do danych, co jest zgodne z zasadą separacji warstw i zwiększa bezpieczeństwo.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys25.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 25. Utworzenie obiektów Networkpolicy</i>
</p>

<p align="justify">Każda polityka została dobrana zgodnie z rolą komponentu w architekturze aplikacji:

▶ Frontend → tylko wysyła ruch do backendu (Egress, port 80).

▶ Backend → odbiera ruch od frontendu i wysyła do MySQL (Ingress + Egress, porty 80 i 3306).

▶ MySQL → przyjmuje ruch tylko od backendu (Ingress, port 3306).

Takie podejście zapewnia minimalne, ale wystarczające reguły komunikacji, zgodne z zasadą least privilege – każdy komponent ma dostęp tylko do tego, czego potrzebuje.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys26.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 26. Nadanie odpowiednich etykiet obiektom bez etykiet</i>
</p>

<p align="justify">Na zakończenie konfiguracji, aby zapewnić pełną spójność i poprawne działanie polityk sieciowych oraz usług, wszystkim obiektom nadano odpowiednie labele. Dzięki temu selektory w plikach YAML mogły jednoznacznie wskazywać właściwe Pody, Deploymenty i Serwisy. Przykładowo, Deployment frontend otrzymał etykietę <code>app=frontend</code>, backend <code>app=backend</code>, a baza danych <code>MySQL app=my-sql</code>. Analogiczne etykiety zostały przypisane usługom sieciowym. Takie podejście gwarantowało, że reguły NetworkPolicy działały zgodnie z założeniami i nie dopuszczały nieautoryzowanego ruchu między komponentami.</p>

# 6. Konfiguracja autoskalowania (Horizontal Pod Autoscaler)

<p align="justify">Ostatnim etapem konfiguracji klastra było wdrożenie mechanizmu automatycznego skalowania replik frontendu w zależności od obciążenia zasobów. W tym celu utworzono obiekt typu <code>HorizontalPodAutoscaler (HPA)</code>, który monitoruje zużycie CPU i pamięci przez Pody frontendu i dynamicznie dostosowuje ich liczbę w zakresie od 3 do 10 replik. Dzięki temu aplikacja może reagować na zmienne obciążenie – zwiększając liczbę instancji w czasie wzmożonego ruchu i zmniejszając ją, gdy zapotrzebowanie spada.</p>

<p align="justify">Objaśnienie:</p>

<code>name:</code> frontend-hpa – nazwa autoskalera dla frontendu.

<code>namespace:</code> frontend – przestrzeń nazw, w której działa Deployment frontend.

<code>minReplicas:</code> 3 – minimalna liczba replik, poniżej której autoskaler nie schodzi.

<code>maxReplicas:</code> 10 – maksymalna liczba replik, której autoskaler nie przekracza.

<code>scaleTargetRef:</code> – odniesienie do obiektu, który ma być skalowany.

<code>name: frontend</code> – nazwa Deploymentu, który ma być skalowany.

<code>behavior:</code> – konfiguracja zachowania skalowania.

<code>scaleUp.stabilizationWindowSeconds: 60</code> – okno czasowe przed zwiększeniem liczby replik (60 sekund).

<code>scaleDown.stabilizationWindowSeconds: 60</code> – okno czasowe przed zmniejszeniem liczby replik (60 sekund).

<code>metrics:</code> – lista metryk, które będą monitorowane.

<code>type: Resource<</code> – typ metryki oparty na zasobach.

<code>resource.name: cpu</code> – monitorowanie zużycia CPU.

<code>target.type: Utilization</code> – typ celu: procentowe wykorzystanie.

<code>averageUtilization: 70</code> – próg skalowania: 70% średniego wykorzystania CPU.

<code>resource.name: memory</code> – monitorowanie zużycia pamięci RAM.

<code>averageUtilization: 70</code> – próg skalowania: 70% średniego wykorzystania pamięci.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys27.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 27. Utworzenie obiektu typu HorizontalPodAutoscaler</i>
</p>

<p align="justify">Wdrożenie HPA pozwala na dynamiczne dostosowanie liczby replik frontendu do aktualnego obciążenia.

▶ minReplicas = 3 – zapewnia minimalną dostępność aplikacji nawet przy niskim ruchu.

▶ maxReplicas = 10 – ogranicza nadmierne skalowanie, chroniąc zasoby klastra.

▶ CPU i RAM jako metryki – wybór obu zasobów pozwala reagować zarówno na intensywne przetwarzanie danych, jak i na wzrost zapotrzebowania na pamięć (np. przy dużych żądaniach HTTP).

▶ StabilizationWindow – zapobiega zbyt częstym zmianom liczby replik, co chroni aplikację przed fluktuacjami i zapewnia płynność działania.</p>

# 7. Weryfikacja działania aplikacji

<p align="justify">Po wdrożeniu wszystkich komponentów aplikacji konieczne było przeprowadzenie testów weryfikujących poprawność konfiguracji. W tym celu użyto poleceń kubectl <code>exec</code>, które pozwalają uruchomić komendy wewnątrz działających Podów. Testy polegały na próbie zestawienia połączenia TCP z odpowiednimi adresami IP i portami usług. Wynik testu jednoznacznie wskazywał, czy komunikacja jest dozwolona (OK), czy też zablokowana (BLOCKED) przez polityki sieciowe.</p>

## 7.1. Wyświetlenie obiektów i adresów IP

Przed rozpoczęciem testów wykonano polecenia 

    kubectl get pods -A
    
    kubectl get svc -A

oraz

    kubectl get deploy -A
    
aby uzyskać listę wszystkich obiektów w klastrze wraz z przypisanymi adresami IP i portami. Dzięki temu możliwe było przygotowanie dokładnych testów łączności pomiędzy komponentami.

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys28.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 28. Wyświetlenie przygotowanych obiektów na klastrze</i>
</p>

<p align="justify">Wyniki:</p>

▶ Frontend-svc (NodePort) – IP: 10.102.84.81, port: 80:30080/TCP

▶ Backend-svc (ClusterIP) – IP: 10.104.250.133, port: 80/TCP

▶ MySQL-svc (ClusterIP) – IP: 10.107.4.201, port: 3306/TCP

## 7.2. Frontend → Backend (port 80)

    kubectl exec -n frontend deploy/frontend -it -- bash -c \
    'timeout 3 bash -c "exec 3<>/dev/tcp/10.104.250.133/80" && echo "Frontend → Backend:80 OK" || echo "Frontend → Backend:80 BLOCKED"'
    
<p align="justify">Wynik: Frontend → Backend:80 OK. Potwierdzono, że frontend może komunikować się z backendem na porcie HTTP 80, zgodnie z polityką sieciową.</p>


## 7.3. Frontend → MySQL (port 3306)

    kubectl exec -n frontend deploy/frontend -it -- bash -c \
    'timeout 3 bash -c "exec 3<>/dev/tcp/10.107.4.201/3306" && echo "Frontend → MySQL:3306 OK" || echo "Frontend → MySQL:3306 BLOCKED"'

<p align="justify">Frontend → MySQL:3306 BLOCKED. Próba połączenia została zablokowana. To zgodne z założeniami – frontend nie powinien mieć bezpośredniego dostępu do bazy danych.</p>


## 7.4. Backend → MySQL (port 3306)

    kubectl exec -n backend deploy/backend -it -- bash -c \
    'timeout 3 bash -c "exec 3<>/dev/tcp/10.107.4.201/3306" && echo "Backend → MySQL:3306 OK" || echo "Backend → MySQL:3306 BLOCKED"'
    
<p align="justify">Backend → MySQL:3306 OK. Backend ma dostęp do bazy danych na porcie 3306, co jest wymagane do obsługi logiki aplikacji.</p>

## 7.5. Backend → Frontend (port 80)

    kubectl exec -n backend deploy/backend -it -- bash -c \
    'timeout 3 bash -c "exec 3<>/dev/tcp/10.102.84.81/80" && echo "Backend → Frontend:80 OK" || echo "Backend → Frontend:80 BLOCKED"'
    
<p align="justify">Backend → Frontend:80 OK. Backend może komunikować się z frontendem na porcie 80, co potwierdza poprawne działanie serwisu i zgodność z polityką sieciową.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys29.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 29. Wykonane testy łączności</i>
</p>

<p align="justify">Weryfikacja działania aplikacji pokazała, że komunikacja w klastrze <code>Kubernetes</code> została poprawnie ograniczona i dopuszczona zgodnie z założeniami. Próba połączenia z bazą danych MySQL z zewnątrz przez port 3306 nie powiedzie się, ponieważ usługa została wystawiona jako <code>ClusterIP</code>, co oznacza, że jest dostępna wyłącznie wewnątrz klastra. Dodatkowo polityka sieciowa dla <code>MySQL</code> zezwala na ruch tylko z Podów backendu, dlatego nawet jeśli ktoś zna adres IP i port, próba połączenia z hosta lub z frontendu zostanie zablokowana. To rozwiązanie chroni bazę danych przed nieautoryzowanym dostępem i wymusza korzystanie z niej wyłącznie przez warstwę logiki aplikacji.</p>

<p align="justify">Podobnie nie uda się połączyć z usługami bez odpowiednich etykiet. W <code>Kubernetes</code> selektory w usługach i politykach sieciowych bazują na etykietach, dlatego tylko Pody oznaczone właściwymi wartościami (<code>app=frontend</code>, <code>app=backend</code>, <code>app=my-sql</code>) są objęte regułami komunikacji. Jeśli Pod nie ma przypisanej etykiety, nie zostanie rozpoznany przez usługę ani politykę i nie będzie mógł nawiązać połączenia. To mechanizm bezpieczeństwa, który eliminuje ryzyko przypadkowych lub nieautoryzowanych Podów wchodzących w interakcję z aplikacją.</p>

<p align="justify">Połączenia działają tam, gdzie zostały jawnie dozwolone. <code>Frontend</code> może komunikować się z backendem na porcie 80, ponieważ polityka frontend-egress-policy oraz usługa backend-svc dopuszczają taki ruch. <code>Backend</code> ma dostęp zarówno do frontendu, jak i do bazy <code>MySQL</code>, co zostało przewidziane w polityce <code>backend-ingress-egress-policy</code>. Dzięki temu backend może odbierać żądania od frontendu i jednocześnie wykonywać zapytania do bazy danych. <code>MySQL</code> przyjmuje połączenia wyłącznie od backendu na porcie 3306, co zostało wymuszone przez politykę <code>mysql-ingress-policy</code>. <code>Frontend</code> nie ma bezpośredniego dostępu do bazy, co jest zgodne z zasadą separacji warstw – frontend komunikuje się z backendem, a backend z bazą danych.</p>

# 8. Test poprawności konfiguracji HPA

<p align="justify">Aby wykazać, że konfiguracja frontendu (Service, HPA, polityki sieciowe) działa poprawnie pod obciążeniem, najlepszym podejściem jest połączenie realnego ruchu HTTP, ciągłej obserwacji autoskalera oraz weryfikacji dostępności aplikacji podczas skalowania. W tym celu najpierw uzyskano adres dostępowy usługi za pomocą polecenia:</p>
  
    minikube service frontend-svc -n frontend --url
    
<p align="justify">W środowisku <code>WSL</code> ruch sieciowy do <code>NodePort</code> bywa niewidoczny bezpośrednio z hosta <code>Windows</code>, a to polecenie tworzy lokalny tunel do usługi i zwraca adres w formie <code>http://127.0.0.1:XXXXX</code>. Dzięki temu testy można wykonywać z hosta bez komplikacji sieciowych i bezpośredniego otwierania portów na węzłach, co w <code>WSL</code> jest bardziej niezawodne.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys30.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 30. Uzyskanie lokalnego adresu usługi dla frontendu</i>
</p>

<p align="justify">Do generowania kontrolowanego obciążenia użyto narzędzia <code>wrk</code>:</p>
  
    wrk -t2 -c20 -d300s http://127.0.0.1:37595/
  
<p align="justify">Parametry zostały dobrane tak, by wiarygodnie pobudzić mechanizmy autoskalera: dwa wątki (t2) i 20 równoległych połączeń (c20) zwykle wystarczają, by podnieść wykorzystanie CPU w kontenerach nginx i wprowadzić HPA w tryb skalowania, a jednocześnie nie przeciążają klastra ani nie prowadzą do sztucznego zatykania stosu sieciowego. Dłuższy czas trwania testu (d300s, czyli 5 minut) jest kluczowy: HPA ma zdefiniowane okna stabilizacji dla scaleUp i scaleDown (60 sekund), więc potrzebujemy wielominutowego okna, by zaobserwować pełny cykl reakcji — wzrost replik powyżej wartości minimalnej i późniejsze stabilne utrzymanie albo powolne zejście, jeśli ruch spadnie. Krótsze testy mogą w ogóle nie przekroczyć progu (70% CPU/70% RAM) albo nie zdołają „przebić się” przez okna stabilizacji, dając mylący obraz braku skalowania.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys31.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 31. Rozpoczęcie testu obciążeniowego</i>
</p>

<p align="justify">W trakcie testu obciążeniowego kluczowe jest równoległe monitorowanie autoskalera i replik. Polecenie:
  
    kubectl get hpa -n frontend -w

pokazuje w czasie rzeczywistym bieżącą średnią wartość CPU i RAM (np. cpu: 3%/70%) oraz aktualną liczbę replik, więc można bezpośrednio obserwować, czy HPA reaguje na ruch zgodnie z konfiguracją (minReplicas=3, maxReplicas=10, target 70%).

<p align="justify">Użycie --url w minikube jest decyzją praktyczną wynikającą z ograniczeń <code>WSL</code> — eliminuje zmienne środowiskowe, które mogłyby zafałszować wynik (np. brak dostępu do NodePort z hosta). <code>wrk</code> generuje realistyczny ruch <code>HTTP</code>, który w przypadku <code>nginx</code> obciąża głównie CPU; to spójne z konfiguracją HPA, gdzie targetem jest średnie wykorzystanie CPU na poziomie 70%. Dodanie metryki pamięci w HPA jest poprawne i kompletną praktyką, ale test z wrk nie jest pamięciochłonny — jeśli celem byłaby weryfikacja również progu dla RAM, test należałoby rozszerzyć o scenariusze generujące większe zapotrzebowanie pamięci (np. serwowanie dużych plików lub testy aplikacji z intensywną alokacją). W tym scenariuszu skupiamy się na dowiedzeniu, że autoskalowanie frontendu zgodnie z progiem CPU i RAMu działa, a pod obciążeniem liczba replik rośnie z 3 w kierunku limitu, stale utrzymując dostępność.</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/97703/FullStack_Zad1/main/rysunki/rys32.png" style="width: 70%; height: 70%" /></p>
<p align="center">
  <i>Rys. 32. Obciążenie obiektu Deploymentu dla Frontendu w czasie</i>
</p>

<p align="justify">Ostatecznym dowodem poprawności konfiguracji jest korelacja trzech obserwacji: po pierwsze, stabilny dostęp do frontendu przez adres z <code>--url</code> przez cały czas trwania testu, co udowadnia poprawną ekspozycję usługi i brak przerw podczas skalowania. Po drugie, wzrost i późniejszy spadek liczby replik obserwowany przez polecenie:</p>
  
    kubectl get hpa -w
    
<p align="justify">oraz w stanie Deploymentu, co potwierdza reakcję <code>HPA</code> na realny wzrost i spadek wykorzystania zasobów. Po trzecie, utrzymanie zasad komunikacji sieciowej niezależnie od skali (frontend nadal ma łączność z backendem, brak bezpośredniego dostępu do MySQL), co pokazują wcześniejsze testy TCP — skalowanie nie pomija NetworkPolicy ani nie zmienia selektorów Service. Konfiguracja jest poprawna: ruch jest obsługiwany, autoskalowanie reaguje, a bezpieczeństwo i separacja warstw są zachowane również pod obciążeniem.</p>

# 9. Część nieobowiązkowa - aktualizacja aplikacji Frontend

## 1. Czy możliwe jest dokonanie aktualizacji aplikacji frontend (np. wersji obrazu kontenera) gdy aplikacja jest pod kontrolą opracowanego autoskalera HPA?

<p align="justify">TAK. Horizontal Pod Autoscaler nie blokuje procesu aktualizacji Deploymentu – <code>HPA</code> działa równolegle z mechanizmami kontrolera Deployment i reaguje na bieżące obciążenie. Oznacza to, że można bez przeszkód zaktualizować obraz kontenera w Deploymentcie, a <code>HPA</code> nadal będzie monitorował metryki i dostosowywał liczbę replik. Dokumentacja Kubernetes potwierdza, że <code>HPA</code> współpracuje z <code>Deploymentami</code> i <code>ReplicaSetami</code>, nie ograniczając ich aktualizacji: <a href="https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/">Kubernetes Docs – Horizontal Pod Autoscaler</a>
.</p>

## 2. Parametry strategii rollingUpdate dla aktualizacji frontendu pod kontrolą HPA

<p align="justify">Aby zagwarantować ciągłość działania aplikacji i jednocześnie nie przekroczyć limitów zasobów zdefiniowanych w przestrzeni nazw frontend, należy odpowiednio dobrać parametry strategii rollingUpdate w Deploymentcie. Przykładowa konfiguracja:</p>

    yaml
    spec:
      strategy:
        type: RollingUpdate
        rollingUpdate:
          maxUnavailable: 1
          maxSurge: 0

<p align="justify">a) Zawsze aktywne 2 Pody: Przy minimalnej liczbie replik ustawionej na 3 (zgodnie z <code>HPA</code>), parametr maxUnavailable: 1 gwarantuje, że w trakcie aktualizacji nigdy nie zostanie wyłączonych więcej niż jeden Pod naraz. Oznacza to, że co najmniej 2 Pody pozostaną aktywne i będą obsługiwać ruch.</p>

<p align="justify">b) Nieprzekroczenie limitów zasobów: Parametr maxSurge: 0 nie pozwala na utworzenie żadnego dodatkowego Poda ponad zdefiniowaną liczbę replik. Dzięki temu w trakcie aktualizacji liczba Podów nie wzrośnie gwałtownie i nie przekroczy limitów CPU, RAM ani maksymalnej liczby replik określonych w ResourceQuota dla namespace frontend.</p>

<p align="justify">c) Korelacja z ustawieniami <code>HPA</code>: dla frontendu zostało skonfigurowane z zakresem od 3 do 10 replik. Strategia rollingUpdate działa w tym zakresie bez konieczności zmian – autoskaler nadal może zwiększać liczbę replik w odpowiedzi na obciążenie. Jeśli jednak w namespace frontend obowiązują bardzo restrykcyjne limity (np. maksymalnie 4 Pody), należałoby dostosować <code>maxReplicas</code> w HPA do tego limitu, aby uniknąć konfliktu między autoskalerem a <code>ResourceQuota</code>. W przeciwnym razie HPA mógłby próbować utworzyć więcej Podów niż dopuszczają limity, co skutkowałoby błędami przy skalowaniu.</p>

<p align="justify">Podsumowanie: Aktualizacja aplikacji <code>frontend</code> jest możliwa nawet pod kontrolą <code>HPA</code>, ponieważ autoskaler nie blokuje procesu Deploymentu. Odpowiednio dobrane parametry rollingUpdate (maxUnavailable: 1, maxSurge: 1) zapewniają, że zawsze pozostaną aktywne co najmniej 2 Pody, a liczba replik nie przekroczy limitów zasobów. W razie potrzeby należy skorelować ustawienia <code>HPA</code> z <code>ResourceQuota</code>, aby uniknąć konfliktów podczas skalowania i aktualizacji.</p>

# Podsumowanie

<p align="justify">Całe zadanie polegało na zbudowaniu kompletnej aplikacji w Kubernetes złożonej z frontendu, backendu i bazy MySQL, wraz z pełną konfiguracją zasobów, usług, polityk sieciowych oraz mechanizmów autoskalowania. Utworzono przestrzenie nazw z kwotami i limitami, wdrożono Deploymenty i Serwisy, a następnie zabezpieczono komunikację za pomocą NetworkPolicy. Testy TCP potwierdziły, że ruch odbywa się wyłącznie w dozwolonych kierunkach: frontend → backend, backend → MySQL, przy jednoczesnym blokowaniu niedozwolonych połączeń.</p>

<p align="justify">Dodatkowo wdrożono Horizontal Pod Autoscaler dla frontendu, który dynamicznie zwiększa liczbę replik w odpowiedzi na obciążenie. Testy obciążeniowe z użyciem wrk wykazały poprawne działanie HPA i utrzymanie dostępności aplikacji. Całość została zaprojektowana zgodnie z zasadą minimalnych uprawnień – każdy komponent ma dostęp tylko do niezbędnych zasobów, co zapewnia bezpieczeństwo, stabilność i skalowalność aplikacji.</p>
