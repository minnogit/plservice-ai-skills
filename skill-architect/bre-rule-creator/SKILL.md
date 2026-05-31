---
name: bre-rule-creator
description: Expert system per la creazione e modifica di Business Rules (BRU)
author: minnogit
version: "1.2"
tags: ["business-rules", "rule-creation", "bre", "workflow-interattivo"]
triggers: 
  - "crea una regola"
  - "creare una nuova regola business"
  - "nuova regola per"
  - "modifica regola esistente"
  - "aggiungi business rule"
  - "regola BRE"
priority: high
---

# Business Rule Creator: Protocollo Architetturale

**ATTIVAZIONE**: Questa skill si attiva quando l'utente desidera modellare logica di business. L'agente AI DEVE procedere in modalità step-by-step e attendere l'input dopo ogni Prompt AI.

---

## STEP 1: Analisi Scenario

**Azione Agente AI**: Presentati e chiedi la descrizione dello scenario.

- Analizza il testo per capire se riguarda un **Verbale** o un **Nominativo**.
- Identifica l'intento primario tra: `compliance`, `correction`, `communication`, `optimization`, `information`.

**STOP**: Attendi descrizione utente.

---

## STEP 2: Scelta Entità e Caricamento Context + Enum Analysis

In base allo scenario, chiedi conferma dell'entità.
**Azione Agente AI**: Solo DOPO la scelta dell'utente, carica i file pertinenti:

1. **Se Verbale**: Carica `../rules/verbali.rules.json` e `VerbaliRuleService.php`.
2. **Se Nominativo**: Carica `../rules/nominativi.rules.json` e `NominativiRuleService.php`.
3. **SEMPRE**: Carica e analizza gli Enum `lib/autoload/EnumBusinessRulesVerbaleInput.php` e `lib/autoload/EnumBusinessRulesVerbaleOutput.php` per costruire un "semantic index"
4. Usa il semantic index per:
   - Suggerire input esistenti che matchano la descrizione utente
   - Rilevare duplicazioni concettuali
   - Proporre naming consistente per nuovi input

**Algoritmo Matching**:

```php
// Pseudocodice per Agente AI
function trovaMiglioreInput(string $descrizioneUtente): array {
    $scores = [];
    foreach (EnumBusinessRulesVerbaleInput::cases() as $case) {
        $score = 0;
        
        // Match su descrizione
        $score += similarita($descrizioneUtente, $case->getDescrizione());
        
        // Match su keywords
        foreach ($case->getKeywords() as $keyword) {
            if (contiene($descrizioneUtente, $keyword)) {
                $score += 10;
            }
        }
        
        // Match su esempi
        foreach ($case->getEsempi() as $esempio) {
            $score += similarita($descrizioneUtente, $esempio) * 0.5;
        }
        
        $scores[$case->value] = $score;
    }
    
    return ordinaPerScore($scores);
}
```

---

## STEP 2.5: Semantic Analysis degli Input (AI-Powered)

**Azione Agente AI**:

1. Analizza la descrizione utente
2. Per ogni input menzionato:
   - Cerca match negli enum esistenti usando:
     - Descrizione
     - Keywords
     - Esempi
   - Se match > 70%: Suggerisci riutilizzo
   - Se match 40-70%: Chiedi conferma se è lo stesso concetto
   - Se match < 40%: Proponi nuovo input con naming suggerito

**Esempio Output**:

```txt
📊 Analisi Semantica:
- "verbale pagato completamente" 
  ✓ MATCH 85% con 'isPagatoSenzaNotifica'
  → Suggerisco: Usare combinazione:
    - isPagatoSenzaNotifica: true
    - verbaleConPagamentoInferiore: false
    
- "necessita approvazione manuale"
  ✗ NESSUN MATCH
  → Suggerisco nuovo input: 'richiedeApprovazioneManuale'
```

---

## STEP 3: Definizione Logica (WHEN)

Proponi gli input (es. `isDaNotificare`, `hasPagamenti`).

- Se l'input è nuovo, suggerisci il nome seguendo il naming semantico booleano (`is...` o `has...`).
- Verifica che la regola sia **modulare**: uno scenario = una regola.

---

## STEP 4: Definizione Output (THEN)

Classifica gli output seguendo le categorie della Guida:

1. **Actions**: Richiedi `obligation_level` (mandatory/recommended/optional) e `priority`.
2. **Validations**: Specifica `status` (allow/deny/require_approval) e se è `blocking`.
3. **Notifications**: Specifica `severity` e se è `actionable`.

**IMPORTANTE - FORMATO OUTPUT**:

- **NON USARE oggetti inline direttamente nell'array `then`** (es. `{severity: "info", instruction: "..."}`)
- **USARE SEMPRE riferimenti a ID di output definiti nella sezione `outputs`** del file JSON
- Ogni output deve essere definito in `outputs.[type]` e poi referenziato tramite il suo ID in `then.[type]`
- Esempio CORRETTO:

  ```json
  {
    "outputs": {
      "notifications": {
        "messaggioPersonalizzato": {
          "severity": "info",
          "instruction": "Testo chiaro per l'operatore"
        }
      }
    },
    "rules": [
      {
        "then": {
          "notifications": ["messaggioPersonalizzato"]
        }
      }
    ]
  }
  ```

- Esempio ERRATO (causa errore "Illegal offset type"):

  ```json
  {
    "then": {
      "notifications": [
        {
          "severity": "info",
          "instruction": "Testo diretto senza ID"
        }
      ]
    }
  }
  ```

**Check ID**: Assegna l'ID nel blocco corretto (es. V001-V099 base, V200-V299 pagamenti).

---

## STEP 5: Generazione e Validazione

Prima della scrittura finale, esegui un **Self-Check**:

- [ ] L'ID è univoco e nel range corretto?
- [ ] Le `instruction` per l'operatore sono chiare e actionable?
- [ ] Se presenti azioni alternative, è stato usato `mutually_exclusive_group`?
- [ ] Gli input sono definiti come ID riferiti nella sezione `inputs`?
- [ ] Gli output sono definiti come ID riferiti nella sezione `outputs` e NON come oggetti inline diretti?

**Output finale**: Modifica il JSON della regola e `function getVerbaleInput` in `VerbaliRuleService` o `function getNominativoInputs` in `NominativiRuleService` in base dell'entità di base della nuova regola. Se necessario, modifica anche il DataProvider.
