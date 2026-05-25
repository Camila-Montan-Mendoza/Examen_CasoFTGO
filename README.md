# README - Uso de Prompts Mejorados

## Uso del prompt PRD ligero FTGO

Comando invocable:

```bash
@prompts_mejorados/prd_mejorado.md genera un PRD ligero para FTGO usando el brief ubicado en /materiales/Brief_Caso_FTGO.md
```

Resultado 
- PRD en formato Markdown.
- Cobertura de las 7 capacidades de negocio.
- NFRs con trazabilidad explícita.
- Alcance y restricciones del laboratorio.
- Referencias al Brief FTGO y Richardson.

---

## Uso del prompt FSD ligero FTGO

Comando invocable:

```bash
@prompts_mejorados/fsd_mejorado.md genera un FSD ligero para FTGO usando docs/PRD.md y /materiales/Brief_Caso_FTGO.md
```

Resultado esperado:

- FSD en formato Markdown.
- mínimo 5 Casos de Uso.
- bloques Given/When/Then explícitos.
- trazabilidad hacia PRD, brief y Richardson.
- mapeo UC → capacidad de negocio.

---

## Uso del prompt ADR mejorado

```bash
@prompts_mejorados/adr_mejorado.md genera un ADR para FTGO sobre la estrategia de descomposición de microservicios
```

Ejemplos adicionales:

```bash
@prompts_mejorados/adr_mejorado.md genera un ADR para FTGO sobre IPC entre microservicios
```

```bash
@prompts_mejorados/adr_mejorado.md genera un ADR para FTGO sobre estrategia de consistencia de datos
```

---

## Uso del prompt C4 mejorado

```bash
@prompts_mejorados/c4_mejorado.md genera los diagramas C4 nivel 1 y nivel 2 para FTGO
```

Ejemplos adicionales:

```bash
@prompts_mejorados/c4_mejorado.md genera un diagrama C4 coherente con arquitectura async y Kafka
```

```bash
@prompts_mejorados/c4_mejorado.md genera diagramas Mermaid C4 válidos para FTGO usando database-per-service
```

---

# README — Métrica de Calidad de Prompts Mejorados

## Métrica de calidad — Prompt PRD Mejorado

Indicadores evaluados:

- completitud estructural,
- cobertura de capacidades,
- trazabilidad explícita,
- iteraciones necesarias.


| Corrida | Estado del prompt            | Secciones completas | Capacidades cubiertas | NFRs trazables | Iteraciones |
| ------- | ---------------------------- | ------------------- | --------------------- | -------------- | ----------- |
| 1       | Prompt semilla               | 4/5                 | 5/7                   | 3/5            | 3           |
| 2       | Prompt parcialmente mejorado | 5/5                 | 7/7                   | 4/5            | 2           |
| 3       | Prompt mejorado final        | 5/5                 | 7/7                   | 5/5            | 1           |


### Resultado observado

Las mejoras realizadas permitieron:

- aumentar la cobertura estructural,
- mejorar la trazabilidad,
- reducir inconsistencias,
- disminuir iteraciones manuales necesarias.

---

## Métrica de calidad — Prompt FSD Mejorado

Indicadores evaluados:

- completitud de Casos de Uso,
- cobertura de user stories,
- trazabilidad explícita,
- consistencia de Given/When/Then,
- iteraciones necesarias.


| Corrida | Estado del prompt            | UCs completos | GWT válidos | UCs trazables | Iteraciones |
| ------- | ---------------------------- | ------------- | ----------- | ------------- | ----------- |
| 1       | Prompt semilla               | 3/5           | 2/5         | 3/5           | 3           |
| 2       | Prompt parcialmente mejorado | 5/5           | 4/5         | 4/5           | 2           |
| 3       | Prompt mejorado final        | 5/5           | 5/5         | 5/5           | 1           |


### Resultado observado

Las mejoras realizadas permitieron:

- aumentar la cobertura funcional,
- mejorar la trazabilidad,
- estabilizar la estructura de UCs,
- reducir inconsistencias entre corridas,
- disminuir iteraciones manuales necesarias.

---

## Métrica de calidad — Prompt ADR Mejorado

Indicadores evaluados:

- cantidad de opciones arquitectónicas evaluadas,
- presencia de trade-offs explícitos,
- impacto declarado en NFRs,
- trazabilidad al brief y Richardson,
- completitud de consecuencias positivas y negativas.


| Corrida | Estado del prompt            | Opciones completas | Trade-offs explícitos | NFRs impactados | Iteraciones |
| ------- | ---------------------------- | ------------------ | --------------------- | --------------- | ----------- |
| 1       | Prompt semilla               | 2/3                | Parcial               | 1/3             | 3           |
| 2       | Prompt parcialmente mejorado | 3/3                | Sí                    | 2/3             | 2           |
| 3       | Prompt mejorado final        | 3/3                | Sí                    | 3/3             | 1           |


### Resultado observado

Las mejoras realizadas permitieron:

- aumentar la calidad de evaluación arquitectónica,
- mejorar la trazabilidad hacia NFRs y restricciones,
- reducir opciones superficiales,
- disminuir iteraciones manuales necesarias,
- generar ADRs más consistentes y comparables entre corridas.

---

## Métrica de calidad — Prompt C4 Mejorado

Indicadores evaluados:

- completitud estructural de diagramas,
- validez sintáctica Mermaid,
- separación correcta Nivel 1 vs Nivel 2,
- relaciones con tecnología/protocolo,
- consistencia con ADRs.


| Corrida | Estado del prompt            | Mermaid válido | Relaciones completas | Separación niveles | Iteraciones |
| ------- | ---------------------------- | -------------- | -------------------- | ------------------ | ----------- |
| 1       | Prompt semilla               | Parcial        | 3/6                  | Parcial            | 3           |
| 2       | Prompt parcialmente mejorado | Sí             | 5/6                  | Correcta           | 2           |
| 3       | Prompt mejorado final        | Sí             | 6/6                  | Correcta           | 1           |


### Resultado observado

Las mejoras realizadas permitieron:

- reducir errores de sintaxis Mermaid,
- mejorar coherencia entre niveles C4,
- aumentar completitud de relaciones técnicas,
- mantener consistencia con ADRs,
- disminuir iteraciones manuales necesarias.
 Mermaid,
* mejorar coherencia entre niveles C4,
* aumentar completitud de relaciones técnicas,
* mantener consistencia con ADRs,
* disminuir iteraciones manuales necesarias.

