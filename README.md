## eBPF powered Network Service and Security Observability for Kubernetes

### Autorzy:

- Jakub Cudak
- Piotr Aksamit
- Karol Jurzec
- Paweł Zaręba

### Rok: 2024

### Grupa: gr.5

## Spis treści

1. [Wprowadzenie](#wprowadzenie)
2. [Podstawy teoretyczne / Stos technologiczny](#podstawy-teoretyczne--stos-technologiczny)
3. [Opis koncepcji Case Study](#opis-koncepcji-case-study)
4. [Architektura rozwiązania](#architektura-rozwiązania)
5. [Opis konfiguracji środowiska](#opis-konfiguracji-środowiska)
6. [Metody instalacji](#metody-instalacji)
7. [Jak odtworzyć? - krok po kroku](#jak-odtworzyć---krok-po-kroku)
8. [Kroki wdrożenia demo](#kroki-wdrożenia-demo)
9. [Prezentacja wyników](#prezentacja-wyników)
10. [Podsumowanie - wnioski](#podsumowanie---wnioski)
11. [Źródła](#źródła)

## Wprowadzenie

Hubble to zaawansowane narzędzie do obserwacji i nadzoru bezpieczeństwa w sieci. Jest zaprojektowane specjalnie dla
środowiska Kubernetes i opiera swoje działanie na technologii eBPF (Extended Berkeley Packet Filter). To rozwiązanie
umożliwia pełną widoczność i śledzenie ruchu sieciowego w czasie rzeczywistym, oferując niezbędne informacje do
monitorowania wydajności, debugowania problemów oraz zapewniania bezpieczeństwa zwłaszcza w rozbudowanej sieci aplikacji
mikroserwisowych. Celem tej pracy jest zapoznanie się z technologią Hubble oraz zaprezentowanie jej możliwości na
przykładzie konkretnego zbioru serwisów działających w środowisku Kubernetes.

## Podstawy teoretyczne / Stos technologiczny

Hubbel zbudowany jest na bazie technologi Cilium i eBPF, które umożliwiają głęboki wgląd w komunikację i zachowanie
usług, a także infrastrukturę sieciową w całkowicie przejrzysty sposób.

Cilium jest oprogramowaniem open source służącym do zabezpieczania łączności sieciowej pomiędzy usługami aplikacji
wdrożonymi przy użyciu platform zarządzania konteneramit takimi jak Docker i Kubernetes. Jego podstawą jest technologia
jądra Linuksa zwana eBPF, która umożliwia dynamiczne wprowadzanie logiki kontroli bezpieczeństwa i kontroli w samym
Linuksie.

Głównym celem eBPF jset bezpieczne i wydajne rozszerzenie możliwości jądra w czasie wykonywania, bez konieczności
wprowadzania zmian w kodzie źródłowym jądra lub ładowania jego modułów. Bezpieczeństwo zapewnia wbudowany w jądro
weryfikator, który przeprowadza statyczną analizę kodu i odrzuca programy, które ulegają awarii, zawieszają się lub w
inny sposób negatywnie wpływają na jądro.

Hubbel ponadto posiada swój graficzny interfejs instalowany jako część głównego serwisu. Pozwala on na automatyczne
wykrywanie wykresu zależności usług dla klastrów Kubernetes, umożliwiając przyjazną dla użytkownika wizualizację i
filtrowanie przepływów danych jako mapy usług.

Zdecydowaliśmy się na uzycie następującego stosu technologicznego:

- Minikube
- MongoDB
- Python + FastAPI
- NodeJS
- Hubble

## Opis koncepcji Case Study

Usługi:

- 1 sewer mongo db
- 3 serwery w pythonie
- Hubble

Koncept:
Monitorowanie, obciążenie procesora, ramu poprzez ciąg odpytywanie bazy danych i innych serwerów.

Serwery:

- serwer 3 odpytuje serwer 2 i nie oczekuje na odpowiedź
- serwer 2 odpytuje serwer 1 i czeka na odpowiedź
- serwer 1 odpytuje mongo db i też czeka na odpowiedź

## Architektura rozwiązania

Zdecydowaliśmy się na analizę ruchu sieciowego w środowisku symulującym mikroserwisową aplikację uzywającą bazy danych
MongoDB.
Architektura będzie wyglądała następująco:
![architecture](img/arch_overview.png)

Zrealizujemy 3 serwisy symulujące środowisko produkcyjne, które będą porozumiewać się ze sobą za pomocą protokołu http.
Serwisy zostaną zrealizowane w technologiach Python i/lub JavaScript.
Będą one generowały obciąenie rzędu kilku-kilkunastu zapytań na sekundę,
co pozwoli na satysfakcjonujące wyniki obserwacji.

W późniejszym stadium pracy mozemy wykonać migrację na klastrowy deployment mongodb (prawdopodobnie PSA),
w celu obserwacji komunikacji pomiędzy węzłami klastra bazodanowego.

## Opis konfiguracji środowiska

Jako środowisko do uruchomienia naszego rozwiązania wybraliśmy Minikube, który pozwala na uruchomienie lokalnego
klastra.
Dzięki temu będziemy mogli w pełni kontrolować środowisko, w którym nasze serwisy będą działać.

W naszym środowisku będziemy potrzebować kilku serwisów:

Trzy kontenery będące serwisami będą komunikować się ze sobą za pomocą protokołu HTTP. Dzięki takiemu rozdzieleniu
serwisów możliwe będzie wygodniejsze monitorowanie działania każdego z serwisów. Za pomocą logowania kontenerów jesteśmy
w stanie ręcznie sprawdzać zgodność monitoringu Hubble'a z faktycznymi danymi.

## Metody instalacji

Poniżej zebrane zostały metody instalacji niezbędnych narzędzi i serwisów, które będą potrzebne do uruchomienia
aplikacji. Środowiskiem docelowym jest system Linux (wystarczającym odpowiednikiem może być także WSL z systemem Linux),
ale w większości przypadków polecenia można dostosować także do systemu MacOS.

### Kubectl

Kubectl to narzędzie do interakcji z klastrem k8s. Będziemy go często potrzebować.
Aby go zainstalować (zakładając środowisko Linux, x86), wykonaj:

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

### Minikube

Minikube to lokalny klaster k8s, którego będziemy używać w tym projekcie.
Instrukcja instalacji znajduje się tutaj: https://minikube.sigs.k8s.io/docs/start/
Jeśli jesteś leniwy, możesz skopiować i wkleić te polecenia (zakładając środowisko Linux, x86):

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube && rm minikube-linux-amd64
```

### Helm

Helm można traktować jako "menedżera pakietów dla Kubernetes". Wiele aplikacji dostarcza tzw. "chart Helm", czyli gotowy
przepis na wdrożenie aplikacji do klastra k8s. Te charty Helm mają zestaw domyślnych wartości, które można nadpisać (jak
ma to miejsce w tym projekcie).
Będziesz musiał zainstalować Helm na swoim lokalnym systemie, więc oto jak to zrobić (
instrukcja: https://helm.sh/docs/intro/install/):

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

### Serwisy Python

Aby zainstalować serwisy Python, wykonaj:

```bash
cd services/service1
docker build -t <your docker account>/service1:latest .
cd ../service2
docker build -t <your docker account>/service2:latest .
cd ../service3
docker build -t <your docker account>/service3:latest .
```

### Cilium/Hubble

Oto przewodnik, który przeprowadzi Cię przez szybki proces domyślnej instalacji. Automatycznie wykryje i wykorzysta
najlepszą możliwą konfigurację dla używanej dystrybucji Kubernetes.

### Instalacja Cilium CLI

Zainstaluj najnowszą wersję Cilium CLI. Narzędzie Cilium CLI można używać do instalacji Cilium, sprawdzania stanu
instalacji Cilium oraz włączania/wyłączania różnych funkcji (np. clustermesh, Hubble).

Linux:

```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

MacOs:

```
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "arm64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-darwin-${CLI_ARCH}.tar.gz{,.sha256sum}
shasum -a 256 -c cilium-darwin-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-darwin-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-darwin-${CLI_ARCH}.tar.gz{,.sha256sum}
```

Inne: <https://github.com/cilium/cilium-cli/releases/tag/v0.16.9>

### Instalacja Cilium

Install Cilium on kubernetes Cluster:

```
cilium install --version 1.15.5
```

To valideate if the Cilium was correclty installed we can run `cilium status --wait`:

```
$ cilium status --wait
   /¯¯\
/¯¯\__/¯¯\    Cilium:         OK
\__/¯¯\__/    Operator:       OK
/¯¯\__/¯¯\    Hubble:         disabled
\__/¯¯\__/    ClusterMesh:    disabled
   \__/

DaemonSet         cilium             Desired: 2, Ready: 2/2, Available: 2/2
Deployment        cilium-operator    Desired: 2, Ready: 2/2, Available: 2/2
Containers:       cilium-operator    Running: 2
                  cilium             Running: 2
Image versions    cilium             quay.io/cilium/cilium:v1.9.5: 2
                  cilium-operator    quay.io/cilium/operator-generic:v1.9.5: 2
```

## Jak odtworzyć? - krok po kroku

Po tym można uruchomić klaster za pomocą `minikube start` - będzie potrzebował jakiegoś sterownika, Docker będzie w
porządku, ponieważ jest preferowaną opcją (alternatywnie, można użyć podman). Minikube obsługuje wiele backendów (
zobacz tutaj: https://minikube.sigs.k8s.io/docs/drivers/), ale nie jest to istotne na potrzeby niniejszej aplikacji.

### MongoDB

Teraz, gdy nasze środowisko jest gotowe do pracy, możemy zacząć instalować usługi k8s.
Na początek będziemy musieli dodać kilka repozytoriów Helm do naszego systemu. Ten projekt będzie używał dwóch:
repozytorium `bitnami`, które dostarcza większość powszechnie używanych usług, oraz repozytorium `cowboysysop`, które
dostarcza Mongo Express - webowe GUI, które umożliwia interakcję z instalacją MongoDB.

Aby dodać repozytoria:

To add the repositories:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add cowboysysop https://cowboysysop.github.io/charts/
helm repo update # pull the data from newly added repositories
```

Po skonfigurowaniu wszystkiego możemy przystąpić do instalacji MongoDB w naszym lokalnym klastrze k8s. Na potrzeby tego
projektu użyjemy architektury PSA (więcej informacji można znaleźć
tutaj: https://www.mongodb.com/docs/v4.0/core/replica-set-architecture-three-members/#primary-with-a-secondary-and-an-arbiter-psa).

Aby zainstalować tę usługę za pomocą Helm, wykonaj następujące polecenie:

```bash
helm install mongo bitnami/mongodb -f mongo_values.yaml
```

Po tym możesz sprawdzić, czy k8s pods zostały poprawnie utworzone za pomocą `kubectl get pods -w` (-w oznacza watch).
Pamiętaj, że ta instalacja utworzy 8GB PVC (persistent volume claim) na Twoim lokalnym systemie plików.

Aby faktycznie interagować z naszą instalacją, skorzystamy z Mongo Express. Zainstaluj go za pomocą:

```bash
helm install mongo-express cowboysysop/mongo-express -f mongo_express_values.yaml
```

Ponownie, możemy sprawdzić, czy pod został poprawnie zainicjalizowany za pomocą `kubectl get pods -w`.

Aby uzyskać dostęp do webowego GUI z przeglądarki, musisz wykonać następujące polecenie:

bash

```bash
kubectl port-forward <the name of the mongo-express pod> 8081:8081
```

Teraz możesz uzyskać dostęp do GUI, odwiedzając `localhost:8081` w swojej przeglądarce.

### Serwisy Python

Aby uruchomić deployment serwisów Python, wykonaj:

```bash
kubectl apply -f deployments/service1-deployment.yaml
kubectl apply -f deployments/service2-deployment.yaml
kubectl apply -f deployments/service3-deployment.yaml
```

### Enable Hubble / Grafana & Prometheus

Hubble jest warstwą obserwowalności Cilium i może być używana do uzyskania widoczności na poziomie całego klastra w
warstwie sieciowej i zabezpieczeń klastra Kubernetes. Aby włączyć Hubble, uruchom polecenie cilium hubble enable.
Alternatywnie, jeśli chcesz dodać również Grafanę i Prometheusa do obserwowalności, zamiast cilium hubble enable
uruchom:

```bash
helm upgrade cilium cilium/cilium --version 1.15.5 \
   --namespace kube-system \
   --reuse-values \
   --set hubble.relay.enabled=true \
   --set hubble.ui.enabled=true \
   --set prometheus.enabled=true \
   --set operator.prometheus.enabled=true \
   --set hubble.enabled=true \
   --set hubble.metrics.enableOpenMetrics=true \
   --set hubble.metrics.enabled="{dns,drop,tcp,flow,port-distribution,icmp,httpV2:exemplars=true;labelsContext=source_ip\,source_namespace\,source_workload\,destination_ip\,destination_namespace\,destination_workload\,traffic_direction}"
```

Uruchom cilium status, aby sprawdzić, czy Hubble jest włączony i działa:

```
$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:         OK
 \__/¯¯\__/    Operator:       OK
 /¯¯\__/¯¯\    Hubble:         OK
 \__/¯¯\__/    ClusterMesh:    disabled
    \__/

DaemonSet         cilium                   Desired: 3, Ready: 3/3, Available: 3/3
Deployment        cilium-operator          Desired: 1, Ready: 1/1, Available: 1/1
Deployment        hubble-relay             Desired: 1, Ready: 1/1, Available: 1/1
Containers:       cilium                   Running: 3
                  cilium-operator          Running: 1
                  hubble-relay             Running: 1
Image versions    cilium-operator          quay.io/cilium/operator-generic:v1.9.5: 1
                  hubble-relay             quay.io/cilium/hubble-relay:v1.9.5: 1
                  cilium                   quay.io/cilium/cilium:v1.9.5: 3
```

### Instalacja Hubble client:

Aby zainstalować Hubble Client, wykonaj:

Linux:

```
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
sha256sum --check hubble-linux-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-linux-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-linux-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

MacOs:

```
HUBBLE_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/hubble/master/stable.txt)
HUBBLE_ARCH=amd64
if [ "$(uname -m)" = "arm64" ]; then HUBBLE_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/hubble/releases/download/$HUBBLE_VERSION/hubble-darwin-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
shasum -a 256 -c hubble-darwin-${HUBBLE_ARCH}.tar.gz.sha256sum
sudo tar xzvfC hubble-darwin-${HUBBLE_ARCH}.tar.gz /usr/local/bin
rm hubble-darwin-${HUBBLE_ARCH}.tar.gz{,.sha256sum}
```

Windows:

```
curl -LO "https://raw.githubusercontent.com/cilium/hubble/master/stable.txt"
set /p HUBBLE_VERSION=<stable.txt
curl -L --fail -O "https://github.com/cilium/hubble/releases/download/%HUBBLE_VERSION%/hubble-windows-amd64.tar.gz"
curl -L --fail -O "https://github.com/cilium/hubble/releases/download/%HUBBLE_VERSION%/hubble-windows-amd64.tar.gz.sha256sum"
certutil -hashfile hubble-windows-amd64.tar.gz SHA256
type hubble-windows-amd64.tar.gz.sha256sum
:: verify that the checksum from the two commands above match
tar zxf hubble-windows-amd64.tar.gz
```

Aby włączyć interfejs Hubble, uruchom polecenie cilium hubble enable --ui, a następnie otwórz go poleceniem cilium
hubble ui lub kubectl port-forward -n kube-system svc/hubble-ui <numer portu>:80 &. Jeśli przeglądarka nie otworzyła
automatycznie interfejsu, otwórz stronę http://localhost:12000 w swojej przeglądarce. Powinieneś zobaczyć ekran z
zaproszeniem do wyboru namespace. Użyj rozwijanego menu selektora namespace w lewym górnym rogu, aby wybrać namespace i
wybierz default.

## Kroki wdrożenia demo

Aby wdrożyć demo, najpierw przygotowano środowisko poprzez instalację Grafany i Hubble'a. Instalacja Grafany obejmowała pobranie oprogramowania, konfigurację połączeń do źródeł danych oraz stworzenie dashboardów wizualizujących dane. Równocześnie zainstalowano Hubble'a, skonfigurowano połączenia do monitorowanych zasobów i dodano reguły monitorowania. Następnie skonfigurowano źródła danych zarówno dla Grafany, jak i Hubble'a, a także stworzono dashboardy w Grafanie, które prezentują dane w czytelny sposób. W końcowej fazie integracji, Hubble został połączony z Grafaną, co umożliwiło przesyłanie i wyświetlanie danych monitorowania na dashboardach Grafany. Wszystkie te kroki zapewniły pełne wdrożenie systemu monitorowania.

## Prezentacja wyników 

![architecture](img/grafana.jpeg)

Na powyższym zdjęciu widzimy przykładowy dashboard w Grafanie zawierający panele wizualizacyjne, które prezentują dane monitorowania. Panele mogą obejmować: wykresy liniowe pokazujące zmiany w czasie, tabele z danymi liczbowymi lub wskaźniki procentowe. Dashboard jest skonfigurowany tak, aby prezentować dane w czytelny sposób, ułatwiając analizę i identyfikację potencjalnych problemów.

![architecture](img/hubble_graph.jpeg)

Zdjęcie przedstawia graf zależności między serwisami generowane przez Hubble na podstawie ruchu sieciowego. UI pokazuje ponadto analize ruchu sieciowego w różnych segmentach sieci. Hubble dostarcza szczegółowych informacji na temat stanu sieci, co pozwala na szybką identyfikację problemów i podjęcie odpowiednich działań oraz ułatwia analize zależności między serwisami. 

Wyniki prezentowane w Grafanie i Hubble pozwalają na kompleksowe monitorowanie środowiska. Kluczowe korzyści to:

- szybka identyfikacja problemów dzięki czytelnym wizualizacjom.
- możliwość śledzenia trendów i zmian w czasie.
- proaktywne podejście do zarządzania infrastrukturą dzięki alertom i raportom.

Dzięki integracji Grafany i Hubble, uzyskujemy pełny obraz stanu naszego środowiska, co umożliwia lepsze zarządzanie i optymalizację zasobów.

## Podsumowanie - wnioski

Eksperyment pozwolił nam zweryfikować zalety rozwiązania, jakim jest komponent Hubble oraz docenić prostotę jego zaufania.
Jako jego największą zaletę postrzegamy wizualizację grafu zależności serwisów - jest to funkcjonalność praktycznie niemożliwa do zrealizowania bez uciekania się do użycia zewnętrznego narzędzia,
a w przypadku korzystania z rozwiązania Hubble otrzymujemy ją natychmiast. Funkcjonalność ta może potencjalnie bardzo ułatwić administrację klastrem Kubernetes, czy np. wdrożenie nowego administratora w obcą domenę.
Jedną z funkcjonalności reklamowanych przez Hubble jest też zbieranie wyświetlanie metryk dotyczących komunikacji sieciowej, która również w znacznym stopniu ułatwia administrację.


Problemem na który natrafiliśmy podczas pracy z tym rozwiązaniem, była jego instalacja. 
Z niewiadomych przyczyn w niektórych środowiskach (na przykład przy użyciu Windows Subsystem for Linux) komponentu nie udawało się poprawnie zainicjalizować, przez co jego działanie było niemożliwe.
Warto mieć na uwadzę możliwe problemy z instalacją, rozważając wdrożenie tego serwisu.


## Źródła
- https://www.mongodb.com/docs/
- https://docs.cilium.io/en/stable/helm-reference/
- https://github.com/cilium/hubble
- https://helm.sh/docs/intro/cheatsheet/
- https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands
