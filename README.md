# 1.13_whats_new

Od czerwca zeszłego roku kiedy to pojawiła się wersja 1.12 minęło już sporo czasu, więc ekipa z Dockera postanowiła wydać kolejną już wersję - 1.13

W niniejszym artykule będę opisywał zarówno 1.13 (styczeń 2017) jak i nowość z lutego 1.13.1 która wprowadziła kolejne zmiany. 

Warto od razu zaznaczyć że w momencie pisania tego tekstu dokonano podziału dockera na enterprise edition i community edition. Dodatkowo wraz z tym podziałem powołano nowy schemat nazewnictwa oparty o miesiąc/rok tak więc obecnie mamy już 17.03 (CE realesowane miesięcznie zaś EE kwartalnie). O różnicach między 17.03 a 1.13 nie będę pisał bo ich nie znalazłem. Na moje oko wersja 17.03 to z grubsza (z technicznego punktu widzenia) coś w stylu 1.13.2 - zawiera bugfixy do 1.13 i 1.13.1, jej API jest zgodne z API z 1.13 , ma zaimplementowane wszystkie nowe polecenia z 1.13 i zachowuje się identycznie. 

Oczywiście jeśli jednak coś znajdę co różni te wersje to dorzucę stosowne info w komentarzach. 

Wróćmy zatem do zmian technicznych w 1.13 bo te interesują nas najbardziej. 
Aby korzystać z ficzerów jakie opisuję bardzo często trzeba włączyć tryb experimental dockera. Warto od razu zauważyć że od 1.13 docker wbudował obydwie wersje (stable i experimental) w jedną binarkę 

Aby nie dzielić nowości z 1.13 na experimental i stable od razu zmieniamy wywołanie docker demona:

```


# cat /usr/lib/systemd/system/docker.service  | grep -i execstart
ExecStart=/usr/bin/dockerd --experimental=true

# sudo systemctl daemon-reload
# sudo systemctl restart docker


# docker version
Client:
 Version:      1.13.1
 API version:  1.26
 Go version:   go1.7.5
 Git commit:   092cba3
 Built:        Wed Feb  8 06:38:28 2017
 OS/Arch:      linux/amd64

Server:
 Version:      1.13.1
 API version:  1.26 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   092cba3
 Built:        Wed Feb  8 06:38:28 2017
 OS/Arch:      linux/amd64
 Experimental: true

```

## Docker service logs

To jedna z pierwszych nowości jakie omówimy , czy jej wprowadzenie było niezbędne można dyskutować - zawsze można było zainstalować przecież logspouta w trybie swarm global osiągając ten sam efekt. Docker słynie jednak z zastępowania rozwiązań firm trzecich swoimi własnymi odpowiednikami. Podobnie zresztą będzie przy metrics które robią to samo co cAdvisor. 



Aby skorzystać z docker service logs tworzymy 4 kontenerowy service którego zadaniem jest logowanie hostname na wyjście:

```
# docker service create --name w8 --replicas=4 alpine /bin/sh -c "while [ 1 ] ; do hostname  ; sleep 2 ; done "
idcgz0xw4qefhvi5h41zyerrv
```


jak widać swarm uruchomił 4 kontenery:

```
# docker service ps w8 
ID            NAME  IMAGE          NODE     DESIRED STATE  CURRENT STATE                   ERROR  PORTS
meb3inrben6p  w8.1  alpine:latest  cent302  Running        Running less than a second ago         
resh1ax75psj  w8.2  alpine:latest  cent301  Running        Running 3 seconds ago                  
o9qr3re09hwd  w8.3  alpine:latest  cent302  Running        Running less than a second ago         
uv9ki5h7pngc  w8.4  alpine:latest  cent301  Running        Running 3 seconds ago                  
```

sprawdźmy ID kontenerów na lokalnym węźle:

```
# docker ps | grep w8 
46f49b352012        alpine@sha256:dfbd4a3a8ebca874ebd2474f044a0b33600d4523d03b0df76e5c5986cb02d7e8   "/bin/sh -c 'while..."   

About a minute ago   Up About a minute                       w8.4.uv9ki5h7pngc582wnun98ks05
f30d0c5e9956        alpine@sha256:dfbd4a3a8ebca874ebd2474f044a0b33600d4523d03b0df76e5c5986cb02d7e8   "/bin/sh -c 'while..."   

About a minute ago   Up About a minute                       w8.2.resh1ax75psjvrr3f97s348kw
```

kążdy z nich loguje na stdout swój hostname: 

```
# docker logs -f 46f49b352012 | head -3 
46f49b352012
46f49b352012
46f49b352012

# docker logs -f f30d0c5e9956  | head -3 
f30d0c5e9956
f30d0c5e9956
f30d0c5e9956
```

pozostaje zbieranie logów z całego service - do tego służy nowe polecenie:

```
# docker service  logs -f w8 

w8.1.meb3inrben6p@cent302    | 00a9d341b302
w8.2.resh1ax75psj@cent301    | f30d0c5e9956
w8.3.o9qr3re09hwd@cent302    | 6c093ff89396
w8.4.uv9ki5h7pngc@cent301    | 46f49b352012


w8.1.meb3inrben6p@cent302    | 00a9d341b302
w8.2.resh1ax75psj@cent301    | f30d0c5e9956
w8.3.o9qr3re09hwd@cent302    | 6c093ff89396
w8.4.uv9ki5h7pngc@cent301    | 46f49b352012
```

## Logspout - kompatybilność 

Sprawdźmy wywołanego nieco wcześniej do tablicy logspouta. Skoro teoretycznie potrafi routować wszystkie logi na zewnątrz to powinien poradzić sobie również z docker service logs. 

Uruchamiamy logspouta:

```
docker run -d -v /var/run/docker.sock:/tmp/docker.sock --name logspout -e LOGSPOUT=ignore -e DEBUG=true --publish=8000:80 

gliderlabs/logspout:master syslog://172.17.0.1:5000 
```

Obserwujemy listing zdarzeń jakie via REST wystawia kontener logspouta na TCP/8000 

```
curl 0:8000/logs
```

Na drugim terminalu tworzymy 4 kontenerowy serwis (w5) którego jedyną funkcją jest pisanie na STDOUT :

```
# docker service  ps w5 
ID            NAME  IMAGE          NODE     DESIRED STATE  CURRENT STATE          ERROR  PORTS
mxcem2viifbw  w5.1  alpine:latest  cent503  Running        Running 7 seconds ago         
bl4s0ux9f30a  w5.2  alpine:latest  cent503  Running        Running 7 seconds ago         
qcw3pcux3iyc  w5.3  alpine:latest  cent502  Running        Running 8 seconds ago         
yvbebbsmkdew  w5.4  alpine:latest  cent501  Running        Running 8 seconds ago    
```


niestety logspout uruchomiony na node=cent501 zbiera logi tylko z kontenera uruchomionego na cent501:

```
w5.4.yvbebbsmkdewlafbf9s05knsz|7ba88f114552
w5.4.yvbebbsmkdewlafbf9s05knsz|7ba88f114552
w5.4.yvbebbsmkdewlafbf9s05knsz|7ba88f114552
w5.4.yvbebbsmkdewlafbf9s05knsz|7ba88f114552
```

równocześnie inny logspout odpalony na node=cent503 (gdzie są 2 kontenery) pokazuje logi tylko z tych 2 kontenerów


Niestety Issue z pytaniem czy logspout planuje fixa aby zbierać logi ze swarm service logs:
https://github.com/gliderlabs/logspout/issues/271

zamknęli z komentarzem:
No. It only reads logs from the local machine. You'll need to run logspout on every system in your swarm cluster.

Oczywiście logspout ma konkurencję , nie wróżę świetlanej przyszłości przy takim podejściu, była kiedyś taka marka SAAB :-) ... 



##Docker squash image layers

Historycznie przypomnę pewien wątek ("Can't stack more than 42 aufs layers")
https://github.com/docker/docker/issues/1171

Niezależnie od tego czy problemy z ilością warstw dla różnych driverów są faktycznie problemami czy sztuczną barierą którą można zaniedbać mimo wszystko ekipa Dockera postanowiła dodać funkcjonalność spłaszczania obrazów. 

Pokażmy najpierw klasyczne piętrowe obrazy na przykładzie - tworzymy Dockerfile który na gołym IMG=alpine buduje 3 warstwy:

```
# cat Dockerfile 
FROM alpine
RUN echo 1 > /plik
RUN echo 2 >> /plik
RUN echo 3 >> /plik
```
budujemy z tego arcydzieła obraz :

```
# docker build -t test_img_02 .
```

jak widać nie jest on zbyt płaski i zawiera każdą sekcję RUN w postaci 1 extra warstwy 


```
# docker history test_img_02
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
dcefa0bda16e        46 seconds ago      /bin/sh -c echo 3 >> /plik                      6 B                 
258404afe6f5        47 seconds ago      /bin/sh -c echo 2 >> /plik                      4 B                 
fc9880909684        48 seconds ago      /bin/sh -c echo 1 > /plik                       2 B                 
88e169ea8f46        8 weeks ago         /bin/sh -c #(nop) ADD file:92ab746eb22dd3e...   3.98 MB             
```


inspect na obrazie również pokazuje 4 warstwy

```
# docker inspect test_img_02 | tail -10 
            "Type": "layers",
            "Layers": [
                "sha256:60ab55d3379d47c1ba6b6225d59d10e1f52096ee9d5c816e42c635ccc57a5a2b",
                "sha256:8f6a439091db9c676e931bf8da05add5fc01b644ad3325db5421c64a98574995",
                "sha256:8f9aee109583484f37647d88277f4967514a52c3eef10c61a8fc5e154cb93df6",
                "sha256:7b23033eb2828a85c392d5a59006c209ebc2b4fd63b2d61c31819493798feac4"
            ]
        }
    }
]
```


Jak widać każda linijka z Dockerfile jest reprezentowana przez oddzielną warstwę.
Ten pierwszy layer ("sha256:60ab55d3379d47c1ba6b6225d59d10e1f52096ee9d5c816e42c635ccc57a5a2b") to oczywiście alpine:

```
# docker inspect alpine  | tail -10 
            }
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:60ab55d3379d47c1ba6b6225d59d10e1f52096ee9d5c816e42c635ccc57a5a2b"
```




W 1.13.x aby nie robić wielu warstw wprowadzono opcję squash dla IMG:

```
# docker build --squash -t test_img_03 .
```

tym razem docker history pokazuje że warstw mamy dużo mniej:

```
# docker history test_img_03
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
2b4cfdfb143f        45 seconds ago                                                      6 B                 merge 

sha256:dcefa0bda16ef2140bb8cd2a4b1a5eb2d5592d0e8ef3cc80693fe7447ac78103 to 

sha256:88e169ea8f46ff0d0df784b1b254a15ecfaf045aee1856dca1ec242fdd231ddd
<missing>           6 minutes ago       /bin/sh -c echo 3 >> /plik                      0 B                 
<missing>           6 minutes ago       /bin/sh -c echo 2 >> /plik                      0 B                 
<missing>           6 minutes ago       /bin/sh -c echo 1 > /plik                       0 B                 
<missing>           8 weeks ago         /bin/sh -c #(nop) ADD file:92ab746eb22dd3e...   3.98 MB             
```

docker inspect również:

```
# docker inspect test_img_03 | tail -10 
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:60ab55d3379d47c1ba6b6225d59d10e1f52096ee9d5c816e42c635ccc57a5a2b",
                "sha256:7b23033eb2828a85c392d5a59006c209ebc2b4fd63b2d61c31819493798feac4"
            ]
        }
```



Jak widać powyżej obraz zbudowany z opcją squash składa się już tylko z 2 warstw (pierwsza to parent image, druga to skompresowane następne 3 warstwy z deltą zmian - skompresowane do jednej) 

(ten missing to zupełnie normalna sprawa) 


##Docker metrics in Prometheus format

TL;DR: zamiast używać cadvisora można teraz wykorzystać mechanizmy wbudowane w docker engine.


Prometeusz czyli celebryta wśród systemów monitoringu robi zawrotną karierę i każdy dzisiaj chce mieć co najmniej jednego exportera. Obecna lista konektorów przysparza o ból głowy i wkrótce zapewne lodówki, pralki czy drzwi od stodoły będą miały URI="/metrics". Na dzień dzisiejszy rozpoznanie nowego softu można ograniczyć do sprawdzenia resta jakiego wystawia, można również dowolne rozwiązania od razu skreślić i skrytykować za brak exportera. 

Jak wiadomo już wieki temu do exportowania metryk w świecie dockera służył tandem cAdvisor + node exporter. Oczywiście żaden z nich nie pochodzi z fabryki Docker więc _koniecznie_musiano_ dostarczyć "markowy" odpowiednik. 

Do systemd exec startu silnika poza flagą experimental dodajemy --metrics:

```
# cat /usr/lib/systemd/system/docker.service  | grep ExecStart
ExecStart=/usr/bin/dockerd --experimental=true --metrics-addr=0.0.0.0:4999
```

Od tej pory na TCP/4999 wystawiany jest rest typu "prometeuszowy metrics" na którym mamy kilkadziesiąt statsów wprost z wnętrza silnika: 

```
# curl 0:4999/metrics 2>&1 | head 
[...]
# HELP engine_daemon_container_actions_seconds 

The number of seconds it takes to process each container action
# TYPE engine_daemon_container_actions_seconds histogram
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.005"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.01"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.025"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.05"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.1"} 1
engine_daemon_container_actions_seconds_bucket{action="changes",le="0.25"} 1
```

Czy to zagrosi cAdvisorovi ? No nie sądzę, zwłaszcza jak sie popatrzy kim jest jego ojciec i że ten ojciec z dockerem się chyba nie lubi lub za chwilę pokaże konkurencyjne kontenery :-) 




## User Experience i porządki w API 

Dbając o czystość, spójność i przejrzystość dokonano kluczowych zmian w API. Z grubsza chodzi o to żeby wprowadzić w miarę takie same polecenia do kontenerów jak i obrazów itd. Mi tam akurat nie przeszkadzało że nie ma spójnej wizji i jest ogólny chlew (co nawet miało swój urok) no ale ktoś zdecydował że trzeba zrobić porządek. Ja natomiast omówię tu nowe przydatne API - docker system 

```
docker system:
Commands:
  df          Show docker disk usage
  events      Get real time events from the server
  info        Display system-wide information
  prune       Remove unused data
```

### docker system df 

zacznijmy od "df", odpalamy kontener:


```
docker run -tdi ubuntu:14.04
```

potem kontener z volumenem, w środku generacja danych 4 MB :

```
docker run -ti -v /dane --name=test_alpine01 alpine sh 
/ # dd if=/dev/zero of=/dane/PLIK bs=4k  count=1000
```


docker system df pokaże nam wszystko co narobiliśmy:

```
# docker system df
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              2                   2                   191.9 MB            0 B (0%)
Containers          2                   2                   384 B               384 B (100%)
Local Volumes       1                   1                   4.096 MB            0 B (0%)
```



wersja bardziej szczegółowa:

```
# docker system df -v 
Images space usage:

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE                SHARED SIZE         UNIQUE 

SIZE         CONTAINERS
ubuntu              14.04               b969ab9f929b        5 weeks ago         187.9 MB            0 B                 187.9 

MB            1
alpine              latest              88e169ea8f46        2 months ago        3.98 MB             0 B                 3.98 MB 

            1


Containers space usage:

CONTAINER ID        IMAGE               COMMAND             LOCAL VOLUMES       SIZE                CREATED             STATUS  

            NAMES
e9945a3a28cd        alpine              "sh"                1                   96 B                33 seconds ago      Up 31 

seconds       test_alpine01
4639e2890fb2        ubuntu:14.04        "/bin/bash"         0                   0 B                 33 seconds ago      Up 31 

seconds       cranky_clarke

Local Volumes space usage:

VOLUME NAME                                                        LINKS               SIZE
f08a3b5b8b0ac37b8301eabfe7a3c26cfb63f783fb13df46b086165d19292c31   1                   4.096 MB
```


### docker system prune 


stopujemy obydwa kontenery i wykonujemy docker system prune 

```
# docker system prune
WARNING! This will remove:
- all stopped containers
- all volumes not used by at least one container
- all networks not used by at least one container
- all dangling images
Are you sure you want to continue? [y/N] y
Deleted Containers:
e9945a3a28cd6f088fe8d9ed3e30cb3225f3440dbf13434605fcf42ef959ba4f
4639e2890fb2d708447b317830247e48a8ca5f80d5521835d587ea5c84ebc437

Deleted Volumes:
f08a3b5b8b0ac37b8301eabfe7a3c26cfb63f783fb13df46b086165d19292c31

Total reclaimed space: 4.096 MB
```

Poza szeroko działającym system prune są jeszcze 3 polecenia bardziej szczegółowe:


```
# docker container prune 
WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N] y
Total reclaimed space: 0 B

# docker volume prune
WARNING! This will remove all volumes not used by at least one container.
Are you sure you want to continue? [y/N] y
Total reclaimed space: 0 B

# docker image prune 
WARNING! This will remove all dangling images.
Are you sure you want to continue? [y/N] y
Total reclaimed space: 0 B
```

## Stack / Docker Compose / DAB 

TL;DR: Wersja 1.13.x wprowadza wsparcie formatu docker-compose.yml do budowania stacków (już bez DAB) 

A oto szczegóły:

Jak pamiętamy docker-compose mógł obsługiwać swarma ale tylko do wersji 1.11. 

Wraz z nadejściem swarm-mode (swarm wbudowany, od wersji 1.12) docker-compose przestał być wspieraną formą deploymentu usług na swarm i wg niektórych źródeł zaczał mieć nawet status depreciated. Jako następca i alternatywa dla docker-compose w środowisku swarm zaproponowano format DAB (“Distributed Application Bundle”)

Jednocześnie wprowadzono pojęcie swarm docker stack. Celem usystematyzowania przypomnijmy jego definicję i funkcję:
Stack to zbiór usług swarma składających się na środowisko aplikacyjne. 
Stack umożliwia automatyzację instalacji wielu powiązanych ze sobą (zależnościami) serwisów swarma. 

W hierarachi od dołu najpierw zatem mamy kontenery, potem swarm-service (jako zbiór N kontenerów rozpędzonych z jednego obrazu) a następnie na szczycie jest stack który z kolei jest zbiorem serwisów. 

I teraz najważniejsze - Docker w 1.13 znacząco uproscił deployment wielowarstowych środowisk - teraz już nie trzeba używać do tego formatu DAB (który nie do końca wiadomo jaką ma przyszłość) ale wystrarczy do tego ceu używać starego formatu docker-compose.

Jest to jeden z najważniejszych i kluczowych ficzerów w 1.13. 

Dla niektórych wręcz jest to triumfalny i wyczekiwany powrót starego i lubianego formatu docker-compose.yml. Niektórzy idą jeszcze dalej ze spekulacjami i wieszczą naturalną śmierć formatu DAB.

Celem przeprowadzenia LABu tworzymy obraz z którego będziemy rozpędzać kontenery serwujące swój numer rewizji i hostname na porcie 80.

oto Dockerfile:

```
FROM ubuntu:14.04
RUN apt-get -y update && apt-get install -y apache2 && apt-get install -y dnsutils && apt-get install -y curl
CMD apachectl start && echo 1 > /var/www/html/index.html && hostname >> /var/www/html/index.html ; tail -f /dev/null
```
budujemy:

```
docker build -t apacz01 . 
```

tak zbudowany obraz dystrybuujemy na wszystkie węzły swarma (np via docker save , scp , docker load) 


Oto bardzo prosty docker-compose-v3 który uruchomi swarm service oparty o nasz IMG:

```
# cat docker-compose.yml 
version: "3"

services:

  web04:
    image: apacz01
    ports:
      - "9004:80"
    deploy:
      replicas: 2
```



robimy deploy stacka nową metodą:

```
# docker stack deploy --compose-file docker-compose.yml stack_01
Creating network stack_01_default
Creating service stack_01_web04
```

jak widać nasza 2 kontenerowa usługa wstała :

```
# docker service  ls
ID            NAME            MODE        REPLICAS  IMAGE
yjzaowkmrnbb  stack_01_web04  replicated  2/2       apacz01


# docker service ps stack_01_web04 
ID            NAME              IMAGE     NODE     DESIRED STATE  CURRENT STATE          ERROR  PORTS
twjbod862ybu  stack_01_web04.1  apacz01  cent503  Running        Running 8 seconds ago         
vh5w1guhkgl8  stack_01_web04.2  apacz01  cent501  Running        Running 9 seconds ago         
```

a nawet działa ingresowo-ipvs'owy load-balancing (zwraca raz jeden raz drugi kontener) :

```
# curl 0:9004
534c4c112613
# curl 0:9004
27d441d1a697
# curl 0:9004
534c4c112613
```


Listing stacków pracujących na swarmie:

```
# docker stack ls
NAME      SERVICES
stack_01  1
```

Listing kontenerów pracujących w stacku:

```
# docker stack ps stack_01 
ID            NAME              IMAGE     NODE     DESIRED STATE  CURRENT STATE          ERROR  PORTS
twjbod862ybu  stack_01_web04.1  apacz01  cent503  Running        Running 3 minutes ago         
vh5w1guhkgl8  stack_01_web04.2  apacz01  cent501  Running        Running 3 minutes ago         
```

Listing serviców pracujących w stacku:

```
# docker stack services stack_01 
ID            NAME            MODE        REPLICAS  IMAGE
yjzaowkmrnbb  stack_01_web04  replicated  2/2       apacz01
```


warto zauważyć że sieć typu overlay (o nazwie stack_01_default) została dociągnięta TYLKO do tych węzłów swarma na których są kontenery usługi (jak to w SDN zwykło działać) 

Na koniec usuwamy stack :

```
# docker stack  rm stack_01 
Removing service stack_01_web04
Removing network stack_01_default
```


