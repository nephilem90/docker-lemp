Configurazioni docker / docker-compose
=====

### Esempi build immagini PHP:

```bash
docker build \
-t matiux/php:7.3.5-fpm-alpine3.9-dev .
```

Build passando degli argomenti:

```bash
docker build \
--build-arg GITHUB_TOKEN={github-token} \
--build-arg BITBUCKET_CONSUMER_KEY={bitbucket-consumer-key} \
--build-arg BITBUCKET_CONSUMER_SECRET={bitbucket-consumer-secret} \
-t matiux/php:7.3.5-fpm-alpine3.9-base .
```

### Gestione hub:
* Login all'hub: `docker login`
* Push immagine sull'hub: `docker push matiux/php:7.3.5-fpm-alpine3.9-dev`
* Pull immagine dall'hub: `docker pull matiux/php:7.3.5-fpm-alpine3.9-dev`

### Gestione container:
Il file `dc` è un link simbolico a `./docker/dc.sh` ed è una scorciatoia per i comandi `docker-compose` e `composer` e alcuni task per il progetto. Di base, per quanto riguarda docker-compose, lo script `dc` è un wrapper del comando `docker-compose` al quale però, tramite script vengono aggiunti vari parametri. Ogni comando wrappa il comando ma non le opzioni. Ad esempio per fare l'up dei servizi mettendoli in background tramite `dc` si fa `./dc up -d`

* `./dc`: Shortcut per `docker-compose ps`
* `./dc up`: Shortcut per `docker-compose up`. (Es. `./dc up -d`)
* `./dc enter`: Entra nel container php come utente `utente`
* `./dc enter-root`: Entra nel container php come utente `root`
* `./dc build`: Esegue il build di un container quando si modifica il Dockerfile (Es. `./dc build php`)
* `./dc down`: Smonta i servizi. (Es. `./dc down -v` per smontare anche i volumi)
* `./dc purge`: Smonta i servizi e cancella anche le immagini dal disco
* `./dc log`: Mette in ascolto sui log sei servizi
* `./dc *` (qualsiasi altro comando docker-compose)
* `./dc composer` Da accesso a composer fuori dal container

I comandi `./dc up` e `./dc build` copiano dalla macchina host la configurazione git e la configurazione ssh. Questo da modo di lavorare dall'interno del container php come se si fosse sulla macchina host, utile ad esempio per l'installazione di repo composer privati.

### Editing .gitignore:

Aggiungere queste righe al file `.gitignore` del progetto

```bash
.idea
docker/docker-compose.override.yml
docker/php/ssh
docker/php/git
docker/data/shell_history/.zsh_history
```

### Gestione docker-compose:

Per dockerizzare un progetto PHP eseguire i seguenti step:

* Copiare la cartella `docker` e il link simbolico `dc` nella root del progetto da dockerizzare tranne il contenuto della cartela `docker/php`
* Copiare la cartella `docker/php/alpine/*/*/dev/conf` in `docker/php/conf`
* Create il file `docker/php/Dockerfile`
    * Personalizzare il `Dockerfile` specifico per il progetto. (Vedi esempio sotto)
* Copiare `docker/docker-compose.override.dist.yml` in `docker/docker-compose.override.yml`
    * Aggiungere `docker/docker-compose.override.yml` al `.gitignore` del progetto
    * Configurare le porte dei servizi. Il file `docker/docker-compose.override.yml` viene mergiato con il file `docker/docker-compose.yml`. La logica è di non versionare le porte dei servizi nel progetto. In questo modo ogni sviluppatore può configurarsi le porte come vuole anche nel caso in cui tenga due progetti in up con docker-compose; in questo modo le porte non vanno in conflitto.
* Personalizzare il file `docker/docker-compose.yml`
    * Nel servizio `web` in `docker/docker-compose.yml` settare il virtualhost per il framework utilizzato. Ci sono un po' di configurazioni in `docker/nginx` 
* Usare il comando `./dc` per file l'up dei servizi (Vedi istruzioni sopra)

Dentro la cartella `docker` c'è il file `docker-compose` e `docker-compose.`

### Esempio Dockerfile specifico per un progetto:

```bash
FROM matiux/php:7.3.5-fpm-alpine3.9-dev

USER root

RUN apk update \
    && apk add --no-cache \
    autoconf \
    bzip2-dev \
    freetype \
    freetype-dev \
    gettext-dev \
    libjpeg-turbo-dev \
    g++ \
    make \
    && pecl install -o \
    mongodb \
    && docker-php-ext-enable \
    mongodb \
    && docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install -j$(nproc) \
    bz2 \
    calendar \
    exif \
    gd \
    xsl \
    && composer self-update \
    && apk del --purge \
    autoconf \
    openssl-dev \
    g++ \
    make

COPY conf/xdebug-starter.sh /usr/local/bin/xdebug-starter
RUN chmod +x /usr/local/bin/xdebug-starter
RUN /usr/local/bin/xdebug-starter

USER utente

RUN echo 'alias test="./vendor/bin/simple-phpunit --exclude-group slow"' >> /home/utente/.zshrc \
    && echo 'alias test-all="./vendor/bin/simple-phpunit"' >> /home/utente/.zshrc \
    && echo 'alias sfcc="rm -Rf var/cache/*"' >> /home/utente/.zshrc

COPY --chown=utente:utente ssh/* /home/utente/.ssh/
COPY --chown=utente:utente git/.gitconfig /home/utente/.gitconfig
RUN  chmod 600 /home/utente/.ssh/id_rsa

```

### Xdebug:

Nelle immagini `dev` di PHP viene creato lo script di configurazione `/usr/local/etc/php/conf.d/xdebug.ini`. La configurazione è uguale tra Mac e Linux ad eccezione del campo `xdebug.remote_host`. Per gestire il corretto indirizzo, nel Dockerfile specifico del progetto bisogna eseguire lo script `xdebug-starter`.

```bash
COPY conf/xdebug-starter.sh /usr/local/bin/xdebug-starter
RUN chmod +x /usr/local/bin/xdebug-starter
RUN /usr/local/bin/xdebug-starter
```

Lo script si occupa di identificare il giusto host in base al sistema operativo e di settarlo nel file `/usr/local/etc/php/conf.d/xdebug.ini`. Nella cartella `doc/xdebug/phpstorm` ci sono degli screenshots per la configurazione di xdebug su PhpStorm.

### Immagini PHP:
* PHP 7.3.5 fpm - Apline 3.9 (`docker/php/alpine/3.9/7.3.5-fpm`)
* PHP 7.1.29 fpm - Apline 3.9 (`docker/php/alpine/3.9/7.1.29-fpm`)
* PHP 7.0.33 fpm - Apline 3.7 (`docker/php/alpine/3.7/7.0.33-fpm`)

### Miscellanea:
* Le cartelle `public` e `docker/php/test` servono a provare la configurazione `docker-compose` simulando una document root per il virtual host.
* Le immagini `dev` sono tutte configurate con oh-my-zsh, vim, xdebug e shell history. Il file viene creato in `docker/data/shell_history/.zsh_history` ma ovviamente non è versionato.
* Quando si usa un'immagine mysql / mariadb è possibile caricare un dump di un db (compresso o no) mettendolo nella cartella `docker/data/db`. Montando il volume nell'apposito entrypoint, in fase di up questo verrà caricato.

```bash
servicemysql:
    image: mysql:5.7.24
    volumes:
        - app_database:/var/lib/mysql
        - ./mysql/custom.cnf:/etc/mysql/conf.d/custom.cnf
        - ./data/db:/docker-entrypoint-initdb.d
```

* Dentro a ogni container è possibile raggiungere gli altri, usando come host il nome del servizio definito nel file `docker/docker-compose.yml`. Ad esempio l'host del db potrebbe essere `servicemysql`

### Comandi Docker utili:

* Pulizia totale: `docker system prune -af --volumes`
* Stop di tutti i container: `docker stop $(docker ps -aq)`
* Eliminazione di tutti i container stoppati: `docker rm -f $(docker ps -aq)`
* Eliminazione delle immagini: `docker rmi -f $(docker images -aq)`
