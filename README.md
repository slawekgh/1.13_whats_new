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

