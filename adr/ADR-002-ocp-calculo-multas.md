# ADR-002 — Introducir estrategias extensibles para el cálculo de multas

**Fecha:** 2025-02-10
**Estado:** ✅ Aceptado
**Principio SOLID:** O — Open/Closed Principle (OCP)

---

## Contexto

Cuando un usuario devuelve un libro con retraso, el sistema calcula una multa. Actualmente esa lógica vive dentro de `PrestamoService` con una cadena de condicionales según el tipo de usuario:

**Código actual (con el problema):**

```java
// PrestamoService.java — lógica de multas acoplada con condicionales
@Service
public class PrestamoService {

    public double calcularMulta(Prestamo prestamo, String tipoUsuario) {
        long diasRetraso = ChronoUnit.DAYS.between(
            prestamo.getFechaVencimiento(), LocalDate.now()
        );
        if (diasRetraso <= 0) return 0;

        // ❌ Cada nuevo tipo de usuario exige modificar este método
        if (tipoUsuario.equals("ESTUDIANTE")) {
            return diasRetraso * 0.50;   // $0.50 por día
        } else if (tipoUsuario.equals("DOCENTE")) {
            return diasRetraso * 0.25;   // $0.25 por día (tarifa reducida)
        } else if (tipoUsuario.equals("EXTERNO")) {
            return diasRetraso * 1.00;   // $1.00 por día
        } else {
            return diasRetraso * 0.50;
        }
    }
}
```

**¿Cuál es el problema?**

Cada vez que se añade un nuevo tipo de usuario (por ejemplo, `INVESTIGADOR` o `JUBILADO`), hay que **modificar** `PrestamoService`. Esto:
- Rompe código que ya funciona.
- Obliga a revisar y actualizar todas las pruebas existentes del método.
- Mezcla la lógica de negocio de préstamos con las reglas de tarifas.

---

## Decisión

Introducimos la interfaz `EstrategiaMulta` con un único método `calcular()`. Cada tipo de usuario tiene su propia clase que implementa esa interfaz. `PrestamoService` usa la estrategia que recibe, sin importarle cuál es.

**Código corregido:**

```java
// EstrategiaMulta.java — interfaz (contrato cerrado a modificación)
public interface EstrategiaMulta {
    /**
     * Calcula el monto de la multa dado un número de días de retraso.
     * @param diasRetraso días transcurridos después de la fecha de vencimiento
     * @return monto de la multa en dólares
     */
    double calcular(long diasRetraso);
}


// MultaEstudiante.java
public class MultaEstudiante implements EstrategiaMulta {
    private static final double TARIFA = 0.50;

    @Override
    public double calcular(long diasRetraso) {
        return diasRetraso * TARIFA;
    }
}


// MultaDocente.java
public class MultaDocente implements EstrategiaMulta {
    private static final double TARIFA = 0.25;

    @Override
    public double calcular(long diasRetraso) {
        return diasRetraso * TARIFA;
    }
}


// MultaExterno.java
public class MultaExterno implements EstrategiaMulta {
    private static final double TARIFA = 1.00;

    @Override
    public double calcular(long diasRetraso) {
        return diasRetraso * TARIFA;
    }
}


// ✅ Nueva estrategia añadida sin tocar ninguna clase existente
// MultaInvestigador.java
public class MultaInvestigador implements EstrategiaMulta {
    private static final double TARIFA = 0.10; // tarifa preferencial

    @Override
    public double calcular(long diasRetraso) {
        return diasRetraso * TARIFA;
    }
}


// PrestamoService.java — cerrado a modificación respecto al cálculo de multas
@Service
public class PrestamoService {

    public double calcularMulta(Prestamo prestamo, EstrategiaMulta estrategia) {
        long diasRetraso = ChronoUnit.DAYS.between(
            prestamo.getFechaVencimiento(), LocalDate.now()
        );
        if (diasRetraso <= 0) return 0;

        // ✅ No sabe qué estrategia es; solo la usa
        return estrategia.calcular(diasRetraso);
    }
}


// MultaFactory.java — selecciona la estrategia según el tipo de usuario
@Component
public class MultaFactory {

    public EstrategiaMulta obtener(String tipoUsuario) {
        return switch (tipoUsuario) {
            case "ESTUDIANTE"    -> new MultaEstudiante();
            case "DOCENTE"       -> new MultaDocente();
            case "EXTERNO"       -> new MultaExterno();
            case "INVESTIGADOR"  -> new MultaInvestigador();
            default              -> new MultaEstudiante();
        };
    }
}
```

**¿Cómo se usa en conjunto?**

```java
// En el controlador o caso de uso
EstrategiaMulta estrategia = multaFactory.obtener(usuario.getTipo());
double montoMulta = prestamoService.calcularMulta(prestamo, estrategia);
```

### Principio SOLID aplicado — OCP

> "Las entidades de software deben estar abiertas para extensión y cerradas para modificación."

**Antes:** añadir `INVESTIGADOR` → modificar `PrestamoService` (riesgo de regresión).  
**Después:** añadir `INVESTIGADOR` → crear `MultaInvestigador` (cero riesgo sobre código existente).

```
Agregar nuevo tipo de usuario:
  ANTES → modificar PrestamoService    ← toca código que ya funciona
  AHORA → crear nueva clase            ← no toca nada existente
```

**¿Qué está "cerrado"?** La interfaz `EstrategiaMulta` y el método `calcularMulta` de `PrestamoService`.  
**¿Qué está "abierto"?** El conjunto de implementaciones concretas de `EstrategiaMulta`.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Guardar tarifas en base de datos | Cubre multas de tarifa fija, pero no puede expresar reglas más complejas (ej. tarifa progresiva, exenciones por historial) |
| Usar herencia en lugar de composición | La herencia genera jerarquías frágiles. La composición mediante la interfaz es más flexible y testeable |

---

## Consecuencias

### Positivas
- Cada estrategia tiene su propia prueba unitaria independiente.
- Añadir una nueva tarifa no requiere tocar ni revisar las clases existentes.
- `PrestamoService` permanece estable ante cambios en la política de multas.

### Negativas / trade-offs
- `MultaFactory` sigue siendo un punto de modificación cuando se añade un tipo nuevo. Aceptable: el cambio es de una sola línea en el `switch`.
- Se crean varias clases pequeñas. En sistemas con muchos tipos de usuario puede parecer excesivo; valorar si una tabla de configuración en base de datos sería suficiente.
