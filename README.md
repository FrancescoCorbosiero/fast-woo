# fast-woo

Stack Docker per WordPress + WooCommerce dietro un Caddy Docker Proxy esterno:
MariaDB 11.4 LTS, WordPress (Apache + PHP 8.3), Redis object cache, dump
automatici del database. Un unico `docker-compose.yml` copre da un piccolo
negozio a cataloghi di decine di migliaia di prodotti: cambiano solo i valori in `.env`.

## Primo avvio

```bash
cp .env.example .env
```

Compilare le variabili obbligatorie in `.env`, poi:

```bash
docker compose up -d
```

Prerequisito: il network esterno `caddy` e il Caddy Docker Proxy già attivi sulla VPS.

## Aggiornare in produzione

Ogni versione mantiene gli stessi nomi di servizi, container, volumi e reti, e dà
un default a ogni nuova variabile: il `.env` esistente funziona senza modifiche e i
dati nei volumi non vengono toccati. Le note di ogni versione sono in `CHANGELOG.md`
("Richiede azioni: sì/no").

Dalla cartella del progetto sulla VPS:

```bash
git pull
```

```bash
docker compose config -q && docker compose pull
```

```bash
docker compose up -d
```

- `config -q` valida il file prima di toccare qualcosa; `pull` scarica le immagini
  in anticipo, così la ricreazione dei container è più breve.
- Compose ricrea **solo** i container la cui configurazione è cambiata.
- Con un solo container WordPress uno zero-downtime vero non è possibile: se cambia
  solo WordPress il sito è giù pochi secondi; se cambia anche MariaDB circa 10-20 secondi.
- Se non usi git, copia **tutti** i file del repository (anche `apache/`, `uploads.ini`,
  `opcache.ini`). Se un file montato manca, Docker crea al suo posto una cartella vuota
  e Apache/PHP non partono.

Rollback: tornare alla versione precedente e rilanciare `up -d`.

```bash
git checkout <tag-o-commit-precedente>
```

```bash
docker compose up -d
```

## Cron (dalla VPS)

WooCommerce usa WP-Cron e Action Scheduler (email, webhook, abbonamenti, statistiche).
Invece di farli partire dalle visite, si eseguono ogni minuto dal crontab della VPS.
`docker exec` usa le stesse variabili d'ambiente e lo stesso utente del container.

1. Aggiungere al crontab della VPS (`crontab -e`, utente che può usare docker),
   sostituendo il nome del container (`<PROJECT_NAME>-wordpress`):

   ```bash
   * * * * * docker exec -u www-data miosito-wordpress php /var/www/html/wp-cron.php >/dev/null 2>&1
   ```

2. **Solo dopo**, impostare `DISABLE_WP_CRON=true` in `.env` e rilanciare
   `docker compose up -d` (viene ricreato solo WordPress).

L'ordine conta: al contrario, i job resterebbero fermi nel frattempo.

## Backup e ripristino del database

Il container `db-backup` crea un dump compresso ogni notte (`BACKUP_SCHEDULE`, nel
fuso `BACKUP_TIMEZONE`, default UTC) nel volume `<PROJECT_NAME>-backups-data`, e
cancella quelli più vecchi di `BACKUP_RETENTION_MINUTES`. La copia fuori dal server
è affidata al backup della VPS: il dump deve girare **prima** della sua finestra.
Uno snapshot della VPS fotografa MariaDB in funzione: per ripristinare il database
si usa il dump, non i file di MariaDB.

Attenzione: `docker compose down -v` cancella anche il volume dei backup.

Backup manuale:

```bash
docker compose exec db-backup backup01-now
```

Elenco dei backup:

```bash
docker compose exec db-backup ls -la /backup
```

Ripristino (sovrascrive il database attuale):

```bash
docker compose cp db-backup:/backup/<file>.sql.gz .
```

```bash
gunzip -c <file>.sql.gz | docker compose exec -T mariadb sh -c 'exec mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" "$MARIADB_DATABASE"'
```

Dopo il ripristino, svuotare la object cache:

```bash
docker compose exec redis redis-cli FLUSHALL
```

## Dimensionamento

Le due dimensioni sono indipendenti:

- **Catalogo → database**: `MARIADB_*`, `REDIS_MAXMEMORY`, `PHP_MAX_INPUT_VARS`
  (prodotti con molte varianti), `WP_MAX_MEMORY_LIMIT` (import in admin).
- **Traffico → PHP**: `APACHE_MAX_WORKERS`, `WP_MEMORY_LIMIT_CONTAINER`, page cache e CDN.

### Quanti processi PHP (`APACHE_MAX_WORKERS`)

Ogni processo Apache contiene un interprete PHP. Troppi processi rispetto alla
memoria del container fanno intervenire l'OOM killer: con il vecchio default di
Apache (150 processi, limite 1 GB) un test con 120 utenti contemporanei ha servito
66 richieste su 1500 in 11 minuti; con 16 processi 1500 su 1500, zero errori.

1. Misurare la RAM per processo sul sito reale, con traffico normale:

   ```bash
   docker exec miosito-wordpress sh -c 'for p in $(pgrep -x apache2); do awk "{print (\$2-\$3)*4}" /proc/$p/statm; done' | sort -n | awk 'NR>1 {s+=$1; n++; if ($1>m) m=$1} END {printf "RAM privata per processo: media %.0f MB, massimo %.0f MB (%d processi)\n", s/n/1024, m/1024, n}'
   ```

2. Calcolare:
   `APACHE_MAX_WORKERS = min( 2-3 x vCPU , (WP_MEMORY_LIMIT_CONTAINER - PHP_OPCACHE_MEMORY - 100 MB) / RAM massima per processo )`

Oltre 2-3 processi per vCPU la CPU è già satura: altri processi aggiungono solo
memoria e latenza. Per il traffico anonimo la leva vera è la page cache (plugin
che rispetta carrello/checkout/account) più una CDN.

### Valori indicativi

Da validare con la misura sopra: con 30+ plugin un processo pesa tipicamente 60-120 MB.

| Scenario | VPS | WordPress (memoria / processi) | MariaDB (memoria / buffer pool) | Redis (maxmemory) |
|---|---|---|---|---|
| Piccolo, poco traffico | 2 vCPU / 4 GB | 1G / 8 | 768M / 384M | 128mb |
| Piccolo, molto traffico | 4-8 vCPU / 8 GB | 3G / 24 | 1G / 512M | 256mb |
| Grande, poco traffico | 4 vCPU / 16 GB | 2G / 12 | 6G / 4G | 1gb |
| Grande, molto traffico | 16 vCPU / 32 GB | 6G / 32-48 | 16G / 12G | 2gb |

### Più store sulla stessa VPS

I limiti di memoria e CPU sono tetti per singolo store: la somma dei tetti di memoria
deve lasciare spazio a sistema, Caddy e altri siti. I tetti di CPU (`WP_CPU_LIMIT`,
`MARIADB_CPU_LIMIT`) impediscono a uno store in picco di rallentare tutto il resto;
la loro somma può superare i core disponibili, perché raramente i picchi coincidono.

Esempio: 3 store su una VPS da 16 core / 64 GB con altri siti statici
(circa 9,5 GB per store, ~29 GB in totale; ogni store usa al massimo 5+3 core):

```
WP_MEMORY_LIMIT_CONTAINER=4G
WP_MEMORY_RESERVATION=1G
WP_CPU_LIMIT=5
APACHE_MAX_WORKERS=16
MARIADB_MEMORY_LIMIT=5G
MARIADB_MEMORY_RESERVATION=1G
MARIADB_CPU_LIMIT=3
MARIADB_BUFFER_POOL=3G
MARIADB_LOG_FILE_SIZE=512M
MARIADB_IO_CAPACITY=1000
REDIS_MAXMEMORY=384mb
REDIS_MEMORY_LIMIT=576M
```

`MARIADB_BUFFER_POOL=3G` va bene per database fino a ~2 GB. Peso del database:

```bash
docker compose exec mariadb sh -c 'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD" -N -e "SELECT CONCAT(ROUND(SUM(data_length+index_length)/1024/1024), \" MB\") FROM information_schema.tables WHERE table_schema=\"$MARIADB_DATABASE\""'
```

Regole: buffer pool ~50-60% di `MARIADB_MEMORY_LIMIT` sotto i 2 GB, ~70-75% sopra;
`REDIS_MEMORY_LIMIT` ~1,5 x `REDIS_MAXMEMORY`. Tutte le variabili sono descritte in
`.env.example`.

## Note

- Il tag `wordpress:7-php8.3-apache` sceglie PHP e Apache. Il core di WordPress vive nel
  volume e si aggiorna da WordPress: cambiare il tag non aggiorna un sito esistente.
- `WORDPRESS_CONFIG_EXTRA` viene letto a ogni richiesta dal `wp-config.php` generato
  dall'immagine, quindi le costanti nel `.env` si applicano con `docker compose up -d`.
  Verifica (deve restituire un numero maggiore di 0):

  ```bash
  docker compose exec wordpress grep -c getenv_docker wp-config.php
  ```

- L'immagine `tiredofit/db-backup` è fuori manutenzione dal 12/03/2026: resta pinnata
  per digest (v4.1.100). Non usare il tag `:latest`, che oggi non esegue backup.
- Le email partono solo con un plugin SMTP configurato: l'immagine non contiene `sendmail`.
