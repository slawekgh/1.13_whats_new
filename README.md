# 1.13_whats_new


podobno minęło już (tylko?) 175 dni od 1.12.0 


Aby korzystać z ficzerów jakie opisuję bardzo często trzeba włączyć tryb experimental - aby nie dzielić zatem nowości na te dostępne w tym trybie i wbudowane na stałe od razu zmieniany wywołanie docker demona:

Warto jednak zauważyć że:

```
Experimental features are now included in the standard Docker binaries as of version 1.13.0.This is a great improvement since 1.12 release. Under Docker 1.12, there was a separate branch for experimental & stable release and one has to pull them via curl utility from test.docker.com & get.docker.com repository respectively. 
```

Teraz zarówno wersja zwykła jak i experimental są w jednej binarce. 

```
[root@cent301 ~]# cat /usr/lib/systemd/system/docker.service  | grep -i execstart
ExecStart=/usr/bin/dockerd --experimental=true

[root@cent301 ~]# docker version
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

sudo systemctl daemon-reload
sudo systemctl restart docker
```

jak widać już opcja experimental jest włączona 
