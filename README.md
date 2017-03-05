# 1.13_whats_new

Od czerwca zeszłego roku minęło już sporo czasu więc ekipa z Dockera postanowiła wydać kolejną już wersję - 1.13

W niniejszym artykule będę opisywał zarówno 1.13 (styczeń 2017) jak i nowość z lutego 1.13.1 która wprowadziła kolejne zmiany. 

W momencie wypuszczania niniejszego artykułu dokonano podziału na enterprise edition i community edition a wraz z tym podziałem powołano nowy schemat nazewnictwa oparty o miesiąc/rok tak więc obecnie mamy już 17.03 (CE realesowane miesięcznie zaś EE kwartalnie) 

Wróćmy jednak do zmian technicznych bo te interesują nas najbardziej. 
Aby korzystać z ficzerów jakie opisuję bardzo często trzeba włączyć tryb experimental. Warto od razu zauważyć że od 1.13 docker wbudował obydwie wersje (stable i experimental) w jedną binarkę 
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

To jedna z pierwszych nowości jakie omówimy , czy jej wprowadzenie było niezbędne można dyskutować - zawsze można było zainstalować przecież logspouta w trybie swarm global osiągając ten sam efekt. Docker słynie jednak z zastępowania rozwiązań firm trzecich swoimi własnymi odpowiednikami (ubik?). 




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

