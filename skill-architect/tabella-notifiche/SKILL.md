---
name: tabella-notifiche
description: crea una nuova tabella nella sezione invio notifiche
---

# Aggiungere una Nuova Tabella Server-Side nella sezione notifiche

## 1. Creare Handler Backend

File: `gestione_verbali/notifiche/non_spediti/handlers/MyNewTableHandler.php`

```php
<?php
class MyNewTableHandler extends AbstractDataTableHandler
{
    protected function getColumnMap(): array {
        // Elenco delle colonne mappate con i dati della query
        return [
            2 => 'campo_db_1',
            3 => 'campo_db_2'
        ];
    }
    
    protected function getColumnDefs(): array {
        return [
            // Tipo dati
            [
                'type' => 'numeric-comma',
                'targets' => [2]
            ],

            // Checkbox selezione
            [
                'orderable' => false,
                'className' => 'select-checkbox',
                'targets' => 0
            ],

            // ID nascosto
            [
                'visible' => false,
                'searchable' => false,
                'className' => 'col-id',
                'targets' => 1,
                'orderable' => false
            ],

            // Larghezze
            [
                'targets' => [2, 3],
                'width' => '150px'
            ],

            // Campi calcolati non ordinabili
            [
                'orderable' => false,
                'targets' => [3]
            ]
        ];
    }
    
    protected function getData(int $length, int $offset, string $orderBy, string $orderDir, string $search): array {
        // Query database
    }
    
    protected function getTotalCount(): int {
        // Count query
    }
}
```

## 2. Registrare Handler

File: `gestione_verbali/notifiche/non_spediti/handlers/AbstractDataTableHandler.php`

```php
private static array $handlers = [
    'PND_DUP' => 'PndDataTableHandler',
    'MY_NEW_TABLE' => 'MyNewTableHandler', // <-- aggiungere
];
```

## 3. Inizializzare in index.php

File: `gestione_verbali/notifiche/non_spediti/index.php`

```javascript
DataTableFactory.init({
    tableId: '#tabella_principale',
    tipoSpedizione: 'MY_NEW_TABLE',
    ajaxUrl: 'ajax_datatable.php',
    callbacks: {
      onRowSelect: function(selectedIds) { console.log(selectedIds); }
    }
});
```
