# Changelog

## 1.1.0

**Richiede azioni: no.** Il `.env` esistente funziona senza modifiche; volumi, nomi
di servizi, container e reti invariati.

Con `docker compose up -d` vengono ricreati `mariadb`, `wordpress` e `db-backup`
(Redis no): circa 10-20 secondi di indisponibilità.

### Cambiamenti attivi subito

- **Apache: massimo 10 processi PHP** (`APACHE_MAX_WORKERS`) invece del default
  Debian di 150, che con 1 GB di limite portava a processi uccisi dall'OOM killer
  sotto carico. Ogni processo viene riciclato dopo 1000 connessioni
  (`APACHE_MAX_CONN_PER_CHILD`). Verificare il valore con la misura nel README.
- **MariaDB**: `aria_pagecache_buffer_size` 128M → 32M e `key_buffer_size` 128M → 16M
  (WooCommerce usa InnoDB): memoria coerente con il limite del container.
  `MARIADB_AUTO_UPGRADE=1` per aggiornare le tabelle di sistema dopo una patch release.
- **Header**: `Server: Apache` senza versione, rimosso `X-Powered-By`.
- **Healthcheck** sul container WordPress.
- **db-backup**: variabili migrate alla sintassi v4 (`DB01_*`, `DEFAULT_*`), niente più
  avvisi di deprecazione. Stessa immagine, stessi nomi di file, stesso orario (UTC).
- Costanti di `WORDPRESS_CONFIG_EXTRA` nella forma `defined() || define()`.

### Nuovi parametri (default = comportamento precedente)

- PHP: `PHP_MEMORY_LIMIT` (512M), `PHP_MAX_EXECUTION_TIME` (600),
  `PHP_MAX_INPUT_VARS` (5000), `PHP_OPCACHE_MEMORY` (256).
- MariaDB: `MARIADB_LOG_FILE_SIZE` (64M), `MARIADB_FLUSH_LOG_AT_TRX_COMMIT` (2),
  `MARIADB_IO_CAPACITY` (200), `MARIADB_MAX_CONNECTIONS` (151),
  `MARIADB_TMP_TABLE_SIZE` (16M), `MARIADB_ARIA_PAGECACHE` (32M), `MARIADB_KEY_BUFFER` (16M).
- WordPress: `DISABLE_WP_CRON` (false), da attivare dopo il cron sulla VPS.
- Hardening: `APACHE_BLOCK_UPLOADS_PHP` (false), `APACHE_BLOCK_XMLRPC` (false).
- Backup: `BACKUP_TIMEZONE` (Etc/GMT).
- CPU: `WP_CPU_LIMIT`, `MARIADB_CPU_LIMIT` (0 = nessun limite, come prima).

### Altro

- Commenti corretti: `WORDPRESS_CONFIG_EXTRA` viene letto a ogni richiesta (non solo al
  primo avvio), percorso dei backup `/backup`, comando di ripristino funzionante.
- Aggiunti `README.md`, `.env.example`, `.gitignore`; rimosso `docker-compose.yml.bak`
  (usava `tiredofit/db-backup:latest`, che oggi non esegue backup).

Rollback: commit `deb958d` + `docker compose up -d`.

## 1.0.0 (`deb958d`)

Versione iniziale.
