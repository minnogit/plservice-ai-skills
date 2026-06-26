---
name: dashboard-endpoint
description: genera o refactorizza endpoint API dashboard con cache Redis
---

# SKILL: Generatore Endpoint Dashboard con Cache Redis

Sei un Engineering Manager esperto di backend PHP. Il tuo compito è generare o rifattorizzare endpoint API in `lib/api/v1/dashboard/` per contatori e statistiche, implementando cache distribuita via `RedisManager` con fail-safe automatico sul database.

---

## Input richiesti (chiedere se mancano)

1. **Nome file**: es. `count_verbali_scaduti.php`
2. **Prefisso cache key**: es. `count_verb_scaduti_`
3. **Logica di business**: service + method da chiamare, oppure descrizione della query/operazione
4. **TTL (opzionale)**: default 300 secondi (5 minuti). Usa 120 per dati dinamici

---

## Pattern di riferimento

Leggere `docs/patterns/ENDPOINT-API-PATTERN.md` prima di scrivere codice. Le varianti da usare:

| Scenario | Pattern | Cache |
|----------|---------|-------|
| Conteggio `{ "count": N }` | Variante A | Redis 300" |
| Dati strutturati `{ ... }` | Variante B | Redis 300" / 120" |
| Endpoint senza cache | Pattern Base | No |
| Endpoint AJAX (feedback utente) | Pattern Base + campo `message` | Opzionale |

---

## Regole tassative

### 1. Multi-tenancy

Supportare sia la sessione che il parametro esplicito `dbase` (per agenti AI esterni):

```php
$dbase = $_GET['dbase'] ?? $session->dbase;
if (!$dbase) {
    http_response_code(400);
    exit(json_encode(['error' => 'Parametro dbase mancante']));
}
$cacheKey = 'prefisso_' . $dbase;
```

### 2. RedisManager

- **Nessun `require_once`**: `RedisManager` è caricato via autoload.
- **Cache-aside**: `RedisManager::get($key)` → se `!== false`, `exit($cached)`
- **Fail-safe**: se Redis è offline, `get()` restituisce `false` → si procede col database senza interrompere
- **Nessun try-catch** intorno alle chiamate Redis: sono già protette internamente
- **Serializzazione**: si cache la **stringa JSON**, non l'array PHP. Su hit: `exit($cached)`

```php
RedisManager::get(string $key): false|string
RedisManager::setex(string $key, int $ttl, string $value): bool
RedisManager::del(string $key): int
```

### 3. Organizzazione del codice

- Usare servizi dal **container PHP-DI** (`get_container_instance()` + `$container->get(Classe::class)`), non query SQL direttamente nell'endpoint
- `try-catch` solo sulla **business logic**, non sulla lettura cache

### 4. TTL

| Contesto | TTL |
|----------|-----|
| Dati dashboard (conteggi) | 300 secondi (5 minuti) |
| Dati dinamici (ultimi verbali, etc.) | 120 secondi (2 minuti) |

### 5. Error handling

- Cache hit: `exit($cached)` — la stringa è già JSON valido, nessun re-encoding
- Errore business logic: `http_response_code(500)` + `error_log()` + messaggio utente generico
- **Mai esporre `$e->getMessage()`** nella risposta JSON. Sempre `error_log((string) $e)` lato server

### 6. Output JSON

- Castare esplicitamente i valori numerici: `(int)$count`, `(float)$importo`
- Per risposte complesse, preservare la struttura originale dell'array

---

## Template — Variante A (conteggio semplice)

Usare per endpoint che restituiscono `{ "count": N }`.

```php
<?php
// /lib/api/v1/dashboard/[NOME_FILE]
header('Content-Type: application/json; charset=utf-8');
require_once $_SERVER['DOCUMENT_ROOT'] . '/vendor/autoload.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/lib/common.php';

if (!UserSession::startApi()) {
    http_response_code(401);
    exit(json_encode(['error' => 'Non autenticato']));
}

$session = UserSession::get();
$dbase = $_GET['dbase'] ?? $session->dbase;
if (!$dbase) {
    http_response_code(400);
    exit(json_encode(['error' => 'Parametro dbase mancante']));
}

$cacheKey = '[PREFISSO]_' . $dbase;

// 1. Lettura dalla cache
if (($cached = RedisManager::get($cacheKey)) !== false) {
    exit($cached);
}

// 2. Business logic
$container = get_container_instance();
$service = $container->get(Classe::class);

try {
    $result = $service->metodo();
} catch (Throwable $e) {
    http_response_code(500);
    error_log((string) $e);
    exit(json_encode(['error' => 'Errore interno del server']));
}

$jsonResponse = json_encode(['count' => (int)$result]);

// 3. Scrittura in cache
RedisManager::setex($cacheKey, [TTL], $jsonResponse);

echo $jsonResponse;
```

---

## Template — Variante B (risposta complessa)

Usare per endpoint che restituiscono un array/oggetto strutturato.

```php
<?php
// /lib/api/v1/dashboard/[NOME_FILE]
header('Content-Type: application/json; charset=utf-8');
require_once $_SERVER['DOCUMENT_ROOT'] . '/vendor/autoload.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/lib/common.php';

if (!UserSession::startApi()) {
    http_response_code(401);
    exit(json_encode(['error' => 'Non autenticato']));
}

$session = UserSession::get();
$dbase = $_GET['dbase'] ?? $session->dbase;
if (!$dbase) {
    http_response_code(400);
    exit(json_encode(['error' => 'Parametro dbase mancante']));
}

$cacheKey = '[PREFISSO]_' . $dbase;

// 1. Lettura dalla cache
if (($cached = RedisManager::get($cacheKey)) !== false) {
    exit($cached);
}

// 2. Business logic
$container = get_container_instance();
$service = $container->get(Classe::class);

try {
    $data = $service->metodo();
    $result = ['chiave' => $data, 'altra_chiave' => $valore];
} catch (Throwable $e) {
    http_response_code(500);
    error_log((string) $e);
    exit(json_encode(['error' => 'Errore interno del server']));
}

$jsonResponse = json_encode($result);

// 3. Scrittura in cache
RedisManager::setex($cacheKey, [TTL], $jsonResponse);

echo $jsonResponse;
```

---

## Template — Pattern Base (senza cache)

Usare per endpoint senza dati ripetuti o che scrivono dati (POST/PUT/DELETE).

```php
<?php
// /lib/api/v1/dashboard/[NOME_FILE]
header('Content-Type: application/json; charset=utf-8');
require_once $_SERVER['DOCUMENT_ROOT'] . '/vendor/autoload.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/lib/common.php';

if (!UserSession::startApi()) {
    http_response_code(401);
    exit(json_encode(['error' => 'Non autenticato']));
}

try {
    $container = get_container_instance();
    $service = $container->get(Classe::class);
    $session = UserSession::get();

    // ... business logic ...

    echo json_encode($result);
} catch (Exception $e) {
    error_log((string) $e);
    http_response_code(400);
    echo json_encode(['error' => 'Errore durante l\'elaborazione della richiesta.']);
} catch (Throwable $e) {
    error_log((string) $e);
    http_response_code(500);
    echo json_encode(['error' => 'Internal Server Error']);
}
```

---

## Risposte AJAX per notifiche frontend

Quando l'endpoint è chiamato da `fetch()`/`$.ajax()`, includere nella response i campi `success` e `message` per `showNotification()`:

```php
echo json_encode([
    'success' => true,
    'message' => 'Operazione completata.',
    'data'    => $data
]);
```

Vedi `docs/patterns/sistema_di_notifiche.md` — Caso B (Chiamate AJAX) e la SKILL `display-notification`.

---

## Esempi di refactoring (da APCu a Redis)

### Prima (APCu)
```php
$cacheTtl = 300;
if (function_exists('apcu_fetch')) {
    $cached = apcu_fetch($cacheKey, $success);
    if ($success) {
        exit(json_encode(['count' => $cached]));
    }
}
// ... logica ...
$count = ...;
if (function_exists('apcu_store')) {
    apcu_store($cacheKey, $count, $cacheTtl);
}
echo json_encode(['count' => $count]);
```

### Dopo (Redis)
```php
if (($cached = RedisManager::get($cacheKey)) !== false) {
    exit($cached);
}
// ... logica ...
$jsonResponse = json_encode(['count' => (int)$result]);
RedisManager::setex($cacheKey, 300, $jsonResponse);
echo $jsonResponse;
```

### Cosa cambia
- Rimuovere `$cacheTtl` (va nel secondo parametro di `setex`)
- Rimuovere i blocchi `if (function_exists('apcu_*'))`
- Cache hit: `exit($cached)` — la stringa è già JSON
- Cache miss: `json_encode()` → `setex()` → `echo`
- Cast esplicito dei valori: `(int)$count`, `(float)$importo`

---

## Documenti correlati

| Documento | Cosa contiene |
|-----------|---------------|
| `docs/patterns/ENDPOINT-API-PATTERN.md` | Guida completa ai pattern endpoint |
| `docs/patterns/Pattern_uso_sessione.md` | Pattern 3 — Endpoint API (JSON) |
| `docs/patterns/DEPENDENCY-INJECTION-GUIDE.md` | Container PHP-DI e servizi |
| `docs/patterns/MYSQLI-PATTERN.md` | Prepared statements per query DB |
| `docs/patterns/sistema_di_notifiche.md` | Caso B — Chiamate AJAX |
| SKILL `display-notification` | Regole notifiche utente |
