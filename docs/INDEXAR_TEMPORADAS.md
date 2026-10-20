# Indexar las temporadas de un año nuevo (producción)

Procedimiento verificado el 2026-10-20. Aplica a cualquier año (2027, 2028, ...).

## Por qué hace falta

- `/v4/seasons/{año}/{temporada}` **solo lee la colección `anime` de Mongo local**. No consulta MAL en el momento. Si los animes de ese año no están guardados, devuelve una lista vacía, sin error.
- Nada de lo que corre a diario agrega animes nuevos: `indexer:anime-current-season` y `indexer:anime-schedule` solo refrescan lo que ya existe, e `indexer:anime-sweep` solo **borra** animes que MAL eliminó.
- MAL anuncia animes con el tiempo, así que hay que volver a correr el script de cada año de vez en cuando.

## Entorno de producción

- Servidor: el de producción, en la carpeta del proyecto (donde está el `docker-compose.yml`).
- Contenedor de la API: `jikan-api-jikan_rest-1`. Los otros: `jikan-api-mongodb-1`, `jikan-api-redis-1`, `jikan-api-typesense-1`.
- Dentro del contenedor el proyecto está en `/app` y la API responde en `localhost:8080`.
- Log: `season_index.log` en la carpeta del servidor. **Ya tiene historial: usar siempre `>>`, nunca `>`.**

## El script `season_range_index.php`

Vive en la carpeta del servidor (copiar al contenedor con `docker cp`). Lee `SEASON_START_YEAR` y `SEASON_END_YEAR`. Recorre winter/spring/summer/fall, pide cada temporada a MAL con `JikanParser->getSeasonal` y luego cada anime a `http://localhost:8080/v4/anime/{id}`, con 5 s de pausa. Solo agrega o actualiza; no borra.

```php
<?php
require __DIR__.'/vendor/autoload.php';
$app = require_once __DIR__.'/bootstrap/app.php';
$kernel = $app->make(Illuminate\Contracts\Console\Kernel::class);
$kernel->bootstrap();

// Adjust these two to change the covered range.
$startYear = (int) (getenv('SEASON_START_YEAR') ?: 2027);
$endYear   = (int) (getenv('SEASON_END_YEAR') ?: 2027);
$seasons   = ['winter', 'spring', 'summer', 'fall'];

$totalCached = 0;
$totalSeasons = 0;

for ($year = $startYear; $year <= $endYear; $year++) {
    foreach ($seasons as $season) {
        $totalSeasons++;
        echo "=== {$season} {$year} ===\n";

        try {
            $results = json_decode(
                app('SerializerV4')->serialize(
                    app('JikanParser')->getSeasonal(new \Jikan\Request\Seasonal\SeasonalRequest($year, $season)),
                    'json'
                ),
                true
            );
        } catch (\Throwable $e) {
            echo "  [SKIP SEASON] {$e->getMessage()}\n";
            continue;
        }

        $anime = $results['anime'] ?? [];
        $itemCount = count($anime);
        echo "  {$itemCount} entries\n";

        foreach ($anime as $i => $entry) {
            $id = $entry['mal_id'];
            $title = $entry['title'];
            $url = "http://localhost:8080/v4/anime/{$id}";

            $ch = curl_init($url);
            curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
            curl_setopt($ch, CURLOPT_TIMEOUT, 30);
            curl_exec($ch);
            $code = curl_getinfo($ch, CURLINFO_HTTP_CODE);
            curl_close($ch);

            $num = $i + 1;
            echo "  [{$num}/{$itemCount}] {$id} - {$title} => HTTP {$code}\n";
            $totalCached++;
            sleep(5);
        }
    }
}

echo "\nDone. Seasons processed: {$totalSeasons}, anime cache attempts: {$totalCached}\n";
```

## Pasos

Los pasos 1 a 3 son de solo lectura. Si alguno no da el resultado esperado, **no seguir**.

1. El archivo no debe existir ya en el contenedor (si existe, no sobrescribir sin revisar):
   ```bash
   docker exec jikan-api-jikan_rest-1 ls -l /app/season_range_index.php
   ```
   Esperado: `No such file or directory`.

2. El proyecto debe estar en `/app`:
   ```bash
   docker exec jikan-api-jikan_rest-1 ls /app/vendor/autoload.php /app/bootstrap/app.php
   ```

3. La API debe responder dentro del contenedor:
   ```bash
   docker exec jikan-api-jikan_rest-1 wget -q -O /dev/null -S http://localhost:8080/v4/anime/1 2>&1 | head -3
   ```
   Esperado: `HTTP/1.1 200 OK`. Si falla, el script marcaría todos los animes como fallidos.

4. Confirmar el script en el servidor y copiarlo:
   ```bash
   grep -n "SEASON_" season_range_index.php
   docker cp season_range_index.php jikan-api-jikan_rest-1:/app/season_range_index.php
   docker exec jikan-api-jikan_rest-1 grep -n "SEASON_" /app/season_range_index.php
   ```

5. Respaldar el log antes de agregarle (el `-n` impide sobrescribir una copia existente):
   ```bash
   cp -n season_index.log season_index.log.bak-AAAAMMDD
   ls -l season_index.log season_index.log.bak-AAAAMMDD
   ```
   Los dos deben pesar lo mismo.

6. Lanzar el año deseado (cambiar `AÑO`):
   ```bash
   echo "=== Corrida AÑO inicio $(date) ===" >> season_index.log
   nohup docker exec -e SEASON_START_YEAR=AÑO -e SEASON_END_YEAR=AÑO jikan-api-jikan_rest-1 php /app/season_range_index.php >> season_index.log 2>&1 &
   ```

7. Seguir el avance (`Ctrl+C` sale sin detener el script):
   ```bash
   tail -f season_index.log
   ```
   Termina con `Done. Seasons processed: 4, anime cache attempts: N`.
   - `0 entries` en una temporada es normal si MAL aún no anunció animes.
   - Un código HTTP distinto de 200 es un fallo individual y no daña lo ya guardado.

8. Para detenerlo, identificar el PID exacto y usar `kill <PID>`. Nunca `pkill` por patrón (`uvicorn`, `php`, etc.): el servidor es compartido con otros servicios.

## Notas

- El archivo copiado vive en la capa del contenedor, no en un volumen. Si se recrea el contenedor, hay que copiarlo de nuevo. Los datos de Mongo, Redis y Typesense están en volúmenes y no se pierden.
- `CurrentSeasonIndexer` (`indexer:anime-current-season`) tiene un `try/catch` vacío: el `file_get_contents` queda fuera del `try` y el comando se cae al primer fallo. No está corregido.
