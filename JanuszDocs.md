# Klaster Janusz
---
## Zespół

### Lider zespołu: Marek Sośnicki
Odpowiedzialności lidera:
- plan spotkań
- pilnowanie terminów
- wstępne planowanie sprintu
- pilnowanie żeby nikt nie wychodził poza swoje kompetencje  

### Odpowiedzialności

#### Komunikacja: Anna Matloch
- wysyłanie maila z podsumowaniem weekly (co się udało, co planujemy w kolejnym tygodniu, jakie napotkaliśmy przeszkody)
- utrzymywanie scrumwise
- dokumentacja spotkań

#### Specyfikacja: Marek Sośnicki
- pilnowanie aby każdy dostarczył specyfikację swojej odpowiedzialności przed rozpoczęciem zadania
- pilnowanie, żeby projekty były ze sobą spójne

#### Dokumentacja: Damian Książak
- pilnowanie aby każdy dostarczał swoją dokumentację

#### Infrastruktura: Mateusz Stadnik
- gitlab
- jenkins
- utrzymywanie serwera

#### Jakość: Jacek Szymonek
- pilnowanie aby każda MR miała jednego reviewera (kto zależnie od technologii)
- coding guideline
- jakość dostarczanego kodu i rozwiązań

## Cel projektu
Zbudowanie klastra obliczeniowego, sprawdzenie jego mocy obliczeniowej i wydajności. Napisanie aplikacji obsługującej klaster.

## Opis projektu
- zbudowanie klastra obliczeniowego złożonego z płytek orange PI
- konfiguracja softu na płytkach
- napisanie aplikacji monitorującej stan 
- rozwój aplikacji obsługującej klaster
- stworzenie programów benchmarkowych i zbadanie możliwości obliczeniowej dla różnych problemów
- ocena korzystności budowy klastra w oparciu o użyte przez nas podzespoły
- webowa aplikacja: monitorowanie stanu klastra, diagnostyka, deployment aplikacji na klastrze, zdalne zarządzanie klastrem
- architektura master-slaves

## Planowane technologie
- OS płytek - wstępnie najnowszy Raspbian
- Obliczenia: Message Passing Interface - libopenmpi-1.10.2 i boost::mpi (1.60.0)
- socket.io
- webowa aplikacja do zarządzania projektem:
   - backend: python
   - baza danych: PostgreSQL
   - frontend: angular.js


## Definition of Done
- kod przeszedł review przynajmniej dwóch osób
- kod jest dostarczony do mastera
- napisane są testy jednostkowe/modułowe sprawdzające kod
- wszystkie nowe i stare testy przechodzą
- przeprowadzono zaplanowane testy hardware'owe 
- poprawiono ewentualne błędy wykryte podczas fazy testowania sprzętu

## Linki

Zadania: https://www.scrumwise.com/scrum/#/overview/project/janusz-klaster/id-149459-0-5   
Ip: klasterjanusz.ddns.net   
Jenkins: https://11883.s.time4vps.eu:8443  
Dla potrzeb rejestracji różnych serwisów dla klastra:
- login: klasterjanusz@gmail.com
- hasło: poleCebuli07 
---

# Konfiguracja sprzętowo-software'owa
## Sprzęt i OS
* Maszynka: Orange Pi PC x8 + Orange Pi Plus (master)
* Armbian (Jessie server)
* `uname -a`: `Linux orangepipc 3.4.110-sun8i #18 SMP PREEMPT Tue Mar 8 20:03:32 CET 2016 armv7l GNU/Linux`
* gcc 4.9.2-10

## Biblioteki
W projekcie korzystamy z
* `openmpi-1.10.2`
* `boost::mpi` w wersji 1.60.0.

## Konfiguracja mastera
* Hostname: `master`
* Użytkownik: `master`, hasło: [tajne]
* Program do wykonania trafia do `/var/mpi-env/`, folder należy do użytkownika `master` (`chown -R`) 

## Konfiguracja workerów
* Identyczna dla wszystkich, z wyjątkiem hostname
* Hostname jest w formacie `slaveXY` (np. slave01, slave09)  
* Użytkownik to `slave`, hasło `slave` (na wszystkich slave'ach)  
* Program do wykonania trafia do `/var/mpi-env/` (na wszystkich slave'ach), folder należy do użytkownika `slave` (chown -R)  
* Aby umożliwić nieinteraktywne logowanie się z mastera na slave'y przez ssh, należy:
    - Na masterze wygenerować parę kluczy: `ssh-keygen -t rsa` - domyślna lokalizacja, no passphrase.
    - Na masterze: `cat .ssh/id_rsa.pub | ssh slave@slaveXY 'cat >> .ssh/authorized_keys'`


## Instalacja openmpi
* Pobierz i rozpakuj źródła z https://www.open-mpi.org/software/ompi/v1.10/downloads/openmpi-1.10.2.tar.bz2.
* `./configure --prefix=/opt/openmpi-1.10.2 --enable-heterogeneous CFLAGS=-march=armv7-a`   
* (opcja ```enable-heterogeneous``` dotyczy przypadku różnych maszyn)
* `sudo make all install`
* Następnie w pliku `/etc/profile` dodaj na końcu linijkę `PATH=$PATH:/opt/openmpi-1.10.2/bin` i przeloguj się

## Instalacja boost::mpi (wymaga openmpi)
* Ustawianie większego swapu (potrzebne do kompilacji):
    * `sudo dd if=/dev/zero of=/var/bigswap bs=1M count=1024`
    * `sudo chmod 600 /var/bigswap`
    * `sudo mkswap /var/bigswap`
    * Zamień obecną linjkę ze swapem z `/etc/fstab` na `/var/bigswap swap swap defaults 0 0`
    * `swapoff -a`
    * `swapon -a`
* Instalacja zależności
    * `sudo apt-get install libbz2-dev python-dev`
* Pobierz źródła boosta z http://pkgs.fedoraproject.org/repo/pkgs/boost/boost_1_60_0.tar.bz2/65a840e1a0b13a558ff19eeb2c4f0cbe/boost_1_60_0.tar.bz2 i rozpakuj.
* `bootstrap.sh --prefix=/opt/boost_1_60`
* Na końcu pliku project-settings.jam dopisać linijkę `using mpi : /opt/openmpi-1.10.2/bin/mpicc ;` <-- ważne spacje
* `sudo ./b2 toolset=gcc variant=release link=shared threading=multi address-model=32 --with-mpi install` (omiń opcję `--with-mpi` aby zainstalować wszystkie biblioteki)

## Środowisko   
* W HOME slave'a utwórz plik `.janusz-env` o treści:
    ```
    #!/bin/bash
    export BOOST_ROOT=/opt/boost_1_60
    export OPENMPI_ROOT=/opt/openmpi-1.10.2
    export PATH=$OPENMPI_ROOT/bin:$PATH
    export LD_LIBRARY_PATH=$OPENMPI_ROOT/lib:$BOOST_ROOT/lib:$LD_LIBRARY_PATH
    ```
   i na samą górę `.bashrc` dodaj `source ~/.janusz-env`

## Uruchamianie testowego programu
* Z poziomu master wywołujemy `mpirun -v --display-map --report-bindings --hostfile hosts <nazwa programu>`, gdzie plik `hosts` zawiera informację o dostępnych maszynach, przykładowo:
    ```
    localhost slots=4
    slave@slave01 slots=4
    ```

## Wgrywanie obrazu karty na kartę SD
* `dd if=<plik obrazu> of=<urzadzenie karty> bs=4M conv=fsync`, gdzie urządzenie karty to przykładowo `/dev/mmcblk0`, a plik obrazu to `janusz-slave.img`

## Referencje
http://www.armbian.com/orange-pi-pc/  
http://www.linuxproblem.org/art_9.html  
https://www.open-mpi.org/doc/v1.10/man1/mpirun.1.php  
https://rhinohide.wordpress.com/2012/02/26/openmpi-on-raspberry-pi/   
http://techtinkering.com/2009/12/02/setting-up-a-beowulf-cluster-using-open-mpi-on-linux/  
https://superuser.com/questions/616520/what-to-do-with-a-linux-os-microsd-image-file 
---
# Master node network configuration
### **DHCP server configuration**
1. `sudo apt-get install udhcpd`
2. create config file */etc/udhcpd.conf*:

 ```cpp
 start 192.168.0.20
 end 192.168.0.254
 interface eth0
 opt dns 8.8.8.8 8.8.4.4
 option subnet 255.255.255.0
 opt router 192.168.0.1
 opt domain local
```
3. Set static network configuration for *eth0*: */etc/network/interfaces*     
  
```cpp
# Wired adapter #1
auto eth0
iface eth0 inet static
address 192.168.0.1
netmask 255.255.255.0
broadcast 192.168.0.255
```
4. Enable udhcp deamon service:
`systemctl enable udhcpd`


### **External network**
1. Connect external adapter to USB port
2. Add *eth1* configuration to: */etc/network/interfaces*:

    ```cpp
    auto eth1
    iface eth1 inet dhcp
    ```
3. restart *eth1* interface or **reboot board**
`ip link set dev eth0 down`
`ip link set dev eth0 up`

### **Noip DUC client configuration**
Instalacja: https://www.noip.com/support/knowledgebase/installing-the-linux-dynamic-update-client-on-ubuntu/
Załączanie podczas boota:
1. Skopiuj `/usr/local/src/noip-XXXX/debian.noip2.sh` do `/etc/init.d`
2. Do pliku `/etc/rc.local` przed linijką `exit 0` dodaj linijkę `/etc/init.d/debian.noip2.sh start`
3. Sprawdzenie czy klient działa: `sudo /usr/local/bin/noip2 -S`
4. Możliwe, że trzeba będzie otworzyć port 8245.

### Przykład konfiguraCji sieciowej dla slave01
`cat /etc/network/interfaces` daje    
```cpp
auto eth0  
iface eth0 inet static
address 192.168.0.11
netmask 255.255.255.0
gateway 192.168.0.1
broadcast 192.168.0.255
auto lo
iface lo inet loopback
```
  
# Przenoszenie plików między maszynami

Na masterze uruchomiony jest server ftp, poprzez który uploadujemy pliki.
Przenoszenie plików z mastera na slavy odbywa się przez komendę rsync. Pliki gotowe do przekazania slave'om umieszczamy w ```/var/mpi-env``` i w tym folderze odpalamy   
```rsync -avzh * slave@<ip>:/var/mpi-env```

---
# Architektura sieciowa
Node'y klastra połączone są switchem, każdy z nich ma ustalony statyczny adres IP. Połączenie z zewnętrzną siecią odbywa się przez kartę sieciową USB podpiętą do mastera. 

---
# Architektura software'owa
![ARCHITECTURE](/uploads/b6863d9af28b1d4d2c1419565b94e062/ARCHITECTURE.jpg)

---
# Specyfikacja Janusz Event Generator

## Omówienie 
  
Janusz Event Generator to zestaw skryptów generujący eventy o polach i nazwach podanych w JanuszEventDefinition.

## Format wiadomości

Wiadomości są dwóch typów - from i to slave, zapisane są one w JanuszEventDefinitions jako plik JSON. Plik zawiera dwa pola eventsFromSlave - zawierający eventy ze slave'a i eventsToSlave, zawierająca eventy do slave. Wewnątrz tych pól powinny znajdowac się tablice, których elementy to obiekty zawierające

- eventName - stringa zawierające 
- parameters - zawierające tablice obiektów zawierających nastepujące pola
   - name - nazwe parametu
   - type - nazwe typu. Dozwolne typy to string, int, double, bool, char

## Wygenerowany format wiadomości

Pierwszym bajtem wiadomości jest numer (typ wiadomości) numer ten jest obliczany na podstawie pozycji wewnątrz pliku, od góry do dołu, najpierw z tablicy eventsToSlave później z tablicy eventsFromSlave. W wygenerowanych plikach ten numer jest enumem. 
Następnie w wiadomości podawane są poszczególne parametry, zapisane w bajtach w Little Endian - dla typów prostych  i jako poszczególne znaki zakończone \0  - dla stringów.

## Generator C++
Generator c++ generuje folder events w którym znajdują się nastepujące pliki:
-Events.h/.cpp - Plik zawierajacy interfejs IEvent, wygenerowanie deklaracje wiadomości dziedziczace po interfejsie oraz enum zawierajacy typy wiadomości.
-EventTranslator.h/.cpp Klasa która służy do serializacji (metoda serialize) / deserializacji(deserialize) eventów. Formatem wyjściowym/wejściowym jest vector<char>.

W przypadku niepowodzenia pojawi się błąd w wywołaniu.
Oraz katalog details który zawiera funkcje do serializacji/deserializacji.
Ten folder należy wgrać do folderu src w JanuszSlaveController


## Generator Python
Generator wygeneruje plik events.py, który należy umieścić w katalogu JanuszMasterServer/server/

Plik ten zawiera wszystkie wiadomości plus metody do deserializacji oraz serializacji

---
# Janusz Slave Controller

Aplikacja odpalana bedzie na slaveach, napisana jest w c++.
Komunikacja z Serverem Pythonowym (Janusz MasterServer ) odbywa się po TCP- w C++ użyta jest biblioteka boost::asio. 

Aby skompilować należy mieć:  

    linuxa
    gcc 5.0 lub wyżej
    boost 1.60 lub wyżej

Budowanie aplikacji.

    pobrać repozytorium - posiadając klucz ssh skonfigurowany z gitlabem
    stworzyć folder w którym do przechowywania plikow wynikowych budowania
    wewnątrz folderu wywołać cmake -DBUILD_TARGET=ON -DGENERATE_EVENTS=ON $ścieżkaDoRepo
    wywołać make
    odpalić aplikacje ./JanuszSlaveController

Budowanie testów -Jak wyżej tylko cmake wywołać z parametrami: cmake -DBUILD_TARGET -DTEST_ENABLED=ON   $ścieżkaDoRepo

    odpalić tests/JanuszSlaveControllerTests


W katalogu głównym znajduje się także skrypt generateEvents.sh - generuje on eventy za pomoca JanuszEventGenerator, odpalenie tego skryptu spowoduje przegenerowanie eventów z najnowszych definicji.

Aplikacja powinna być używana na slave'ach klastra i powina łączyć się z JanuszMasterServer

---
# Specyfikacja JanuszSlaveController

## Wprowadzenie
JanuszSlaveController to aplikacja odpalana na każdym ze slave'ów klastra "Janusz". Aplikacja służy do zbierania odczytów z płytek oraz wykonywania zadanych poleceń. Aplikacja komunikuje się z JanuszMasterServer. 

## Komunikacja

Protokołem komunikacji jest TCP. Dwa pierwsze bajty to rozmiar wiadomości która nadejdzie zapisany na 16 bitach. Format dalszej częsci wiadomości jest specyfikowany przez JanuszEventGenerator. Rodzaje wiadomości i ich pola są w JanuszEventDefinitions.

## Konfiguracja
Defaultowy server którego szuka aplikacja to na razie localhost:8013
Parametry te można zmienić tworząc plik /etc/JanuszSlaveControllerConfig 
Format tego pliku jest nastepujacy

    serverIp <ipServera>
    serverPort <portServera
    logLevel [0-3]

gdzie ip iport servera to wartości używane do znalezienia w sieci JanuszMasterServer.
LogLevel może mieć wartości
    
    0-Debug
    1-Warning  
    2-Info
    3-Error

## Podłaczenie do JanuszMasterServer.

Wykryciu w zadanym ip JanuszMasterServer'a aplikacja wyśle do niego pierwszą wiadomość: HwDataInfoEvent zawierająca hostname płytki.


## Odczyty

### Ram & CPu

Po otrzymaniu HwStatusReq aplikacja wyślę w odpowiedzi parametry zadane w formacie wiadomości: ram usage. swap usage, cpu usage, cpu0-4 usage. Dane te pobrane zostaną z pliku /proc/stat oraz z informacji systemowych.

## Czynności

### Reset płytki

Po otrzymaniu HwResetReq aplikacja zresetuje płytkę poleceniem sysctrl reboot

---
# Specyfikacja Janusz Master Server

## Omówienie 
  
Janusz Master Server to aplikacja zarządzająca klastrem. Jest ona serwerem dla Janusz Slave Controllers.

## Format wiadomości do SlaveController

Wiadomości są dwóch typów - from i to slave, zapisane są one w JanuszEventDefinitions jako plik JSON. Plik zawiera dwa pola eventsFromSlave - zawierający eventy ze slave'a i eventsToSlave, zawierająca eventy do slave. Wewnątrz tych pól powinny znajdowac się tablice, których elementy to obiekty zawierające

- eventName - stringa zawierające 
- parameters - zawierające tablice obiektów zawierających nastepujące pola
   - name - nazwe parametu
   - type - nazwe typu. Dozwolne typy to string, int, double, bool, char

## Format ramki wiadomości JanuszEventProtocol oraz wiadomości

Ramka oraz format jest identyczna jak w JanuszSlaveController. W ramce wysyłany jest 2 bajtowy rozmiar wiadomości, następnie wiadomość.

Pierwszym bajtem wiadomości jest numer (typ wiadomości) numer ten jest obliczany na podstawie pozycji wewnątrz pliku, od góry do dołu, najpierw z tablicy eventsToSlave później z tablicy eventsFromSlave. W wygenerowanych plikach ten numer jest enumem. 
Następnie w wiadomości podawane są poszczególne parametry, zapisane w bajtach w Little Endian - dla typów prostych  i jako poszczególne znaki zakończone \0  - dla stringów

## Komunikacja z aplikacją webową

Odbywa się za pomocą websocketów, wysyłane wiadomości są obiektami JSON.

Event reset: {"reset": true}, na topicu o nazwie : [node_name]_reset, np. Slave_1_reset

---
# Produkcyjne ustawienia do odpalenia instancji aplikacji webowej przez developera
## Repozytorium

### Pobranie repozytorium:
* git clone https://11883.s.time4vps.eu:8085/klaster/janusz.git

### W przypadku komunikatu "SSL certificate problem: self signed certificate":
* git config --global http.sslverify false

## Potrzebne paczki (zainstalować)

* python-pip
* python-dev
* libpq-dev
* postgresql 
* postgresql-contrib

## Tworzenie PostgreSQL i użytkownika

### Stworzenie katalogu do danych:
* sudo mkdir /var/lib/postgres/data

### Ustaw właściciela:
* sudo chown -c -R postgres:postgres /var/lib/postgres

### Inicjalizacja:
* sudo -i -u postgres
* initdb -D '/var/lib/postgres/data'

### Wylogowanie i wystartowanie serwera:
* logout
* sudo systemctl start postgresql

### Automatyczne uruchamianie PostgreSQL po restarcie:
* sudo systemctl enable postgresql

### Tworzenie bazy:
* sudo su - postgres
* psql
* CREATE DATABASE janusz_db;
* CREATE USER janusz WITH PASSWORD 'poleCebuli07';
* GRANT ALL PRIVILEGES ON DATABASE janusz_db TO janusz;
* \q
* exit

## Tworzenie wirtalnego środowiska:

### Instalacja virtualenv'a:
* sudo pip install virtualenv

### Stworzenie środowiska w dowolnym katalogu (najlepiej obok projektu - poza gitem):
* virtualenv janusz_env

### Aktywacja środowiska (najlepiej dodać alias):
* vim ~/.bashrc
* dodajemy na przykład: alias janusz="source ~/Pulpit/local_app/janusz_env/bin/activate && cd ~/Pulpit/local_app/janusz/web_app/janusz/"
* source ~/.bashrc
* janusz
* (deactivate wyłącza środowisko)

### Instalacja paczek w wirtualnym środowisku (za pomocą requirements.txt):
* aktywacja środowiska za pomocą aliasu: janusz (jeśli nie jest aktywowane wcześniej)
* ./configure.sh (można iść spokojnie na herbatę)

Migracje:
---------

### Przy pierwszym uruchomieniu:
* ./manage.py migrate
* ./manage.py createsuperuser (wpisać odpowiednio: janusz, klasterjanusz@gmail.com, poleCebuli07)

## Uruchomienie instancji:


### Ustawienia gunicorna:
* uruchamiany przez skrypt start_services.sh (nie wymaga ustawiania)

### Ustawienia nginxa:
* sudo vim /etc/nginx/sites-available/janusz
```    
server { 
    listen 80;
    server_name klasterjanusz.ddns.net;

    location = /favicon.ico { access_log off; log_not_found off; }
    location /static/ {
        root /home/deployment/janusz/web_app/janusz;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/deployment/janusz/web_app/janusz/janusz.sock;
    }
    }
```
* sudo ln -s /etc/nginx/sites-available/janusz /etc/nginx/sites-enabled
* sudo service nginx restart

### Start aplikacji:
* ./start_services.sh


---
# Lokalne ustawienia do odpalenia instancji aplikacji webowej przez developera
## Repozytorium

### Pobranie repozytorium:
* git clone https://11883.s.time4vps.eu:8085/klaster/janusz.git

### W przypadku komunikatu "SSL certificate problem: self signed certificate":
* git config --global http.sslverify false

Potrzebne paczki (zainstalować)
-------------------------------
* python-pip
* python-dev
* libpq-dev
* postgresql 
* postgresql-contrib

Tworzenie PostgreSQL i użytkownika
----------------------------------

### Stworzenie katalogu do danych:
* sudo mkdir /var/lib/postgres/data

### Ustaw właściciela:
* sudo chown -c -R postgres:postgres /var/lib/postgres

### Inicjalizacja:
* sudo -i -u postgres
* initdb -D '/var/lib/postgres/data'

### Wylogowanie i wystartowanie serwera:
* logout
* sudo systemctl start postgresql

### Automatyczne uruchamianie PostgreSQL po restarcie:
* sudo systemctl enable postgresql

### Tworzenie bazy:
* sudo su - postgres
* psql
* CREATE DATABASE janusz_db;
* CREATE USER janusz WITH PASSWORD 'poleCebuli07';
* GRANT ALL PRIVILEGES ON DATABASE janusz_db TO janusz;
* \q
* exit

Tworzenie wirtalnego środowiska:
--------------------------------

### Instalacja virtualenv'a:
* sudo pip install virtualenv

### Stworzenie środowiska w dowolnym katalogu (najlepiej obok projektu - poza gitem):
* virtualenv janusz_env

### Aktywacja środowiska (najlepiej dodać alias):
* vim ~/.bashrc
* dodajemy na przykład: alias janusz="source ~/Pulpit/local_app/janusz_env/bin/activate && cd ~/Pulpit/local_app/janusz/web_app/janusz/"
* source ~/.bashrc
* janusz
* (deactivate wyłącza środowisko)

### Instalacja paczek w wirtualnym środowisku (za pomocą requirements.txt):
* aktywacja środowiska za pomocą aliasu: janusz (jeśli nie jest aktywowane wcześniej)
* ./configure.sh (można iść spokojnie na herbatę)

Migracje:
---------

### Przy pierwszym uruchomieniu:
* ./manage.py migrate
* ./manage.py createsuperuser (wpisać odpowiednio: janusz, klasterjanusz@gmail.com, poleCebuli07)

Uruchomienie instancji:
-----------------------

### Komenda:
* ./run_dev.sh

### W przeglądarce:
* http://127.0.0.1:8000/


Specyfikacja aplikacji webowej
====================================================================
Backend
------------

### Nginx
* wersja 1.6.2
### Gunicorn
* wersja:19.6.0
### PostgreSQL
### Django
### Django REST framework

Frontend
------------
### HTML:
* wersja: 5.0
### CSS
### AngularJS


