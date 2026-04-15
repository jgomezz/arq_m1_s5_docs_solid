# ADR — Architecture Decision Records

Índice de todas las decisiones de arquitectura del Sistema de Biblioteca Digital.

---

## ¿Qué es un ADR?

Un **Architecture Decision Record** es un documento breve que registra una decisión importante de diseño: qué se decidió, por qué, y cuáles son sus consecuencias. Vive junto al código y se versiona en Git igual que cualquier archivo fuente.

---

## Índice

| ID | Título | Principio | Estado | Fecha |
|----|--------|-----------|--------|-------|
| [ADR-001](./ADR-001-srp-servicio-notificaciones.md) | Separar el envío de notificaciones del servicio de préstamos | SRP | ✅ Aceptado | 2025-02-03 |
| [ADR-002](./ADR-002-ocp-calculo-multas.md) | Introducir estrategias extensibles para el cálculo de multas | OCP | ✅ Aceptado | 2025-02-10 |
| [ADR-003](./ADR-003-dip-repositorio-prestamos.md) | Abstraer la persistencia de préstamos mediante interfaz | DIP | ✅ Aceptado | 2025-02-17 |
| [ADR-004](./ADR-004-lsp-tipos-de-libro.md) | Garantizar que todos los tipos de libro son intercambiables | LSP | ✅ Aceptado | 2025-02-24 |
| [ADR-005](./ADR-005-isp-gestion-usuarios.md) | Segregar la interfaz de gestión de usuarios según su rol | ISP | ✅ Aceptado | 2025-03-03 |

---

## Plantilla para nuevos ADR

Copia este bloque en un archivo `ADR-00N-titulo-corto.md` y completa cada sección:

```markdown
# ADR-00N — Título de la decisión

**Fecha:** AAAA-MM-DD
**Estado:** Propuesto | Aceptado | Rechazado | Supersedido
**Principio SOLID:** S | O | L | I | D

---

## Contexto

_¿Qué problema existe hoy? ¿Qué dolor genera?_

## Decisión

_¿Qué se decidió? Escríbelo en forma afirmativa._

### Principio SOLID aplicado

_¿Cómo aplica concretamente el principio? Muestra el antes y el después con código Java._

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| ... | ... |

## Consecuencias

### Positivas
- ...

### Negativas / trade-offs
- ...
```

---

## Ciclo de vida de un ADR

```
Propuesto ──→ Aceptado ──→ Supersedido
     │
     └──→ Rechazado
```

Un ADR **nunca se elimina**. Si la decisión cambia, se crea uno nuevo y el anterior pasa a `Supersedido`.
