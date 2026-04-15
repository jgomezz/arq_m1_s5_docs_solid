# ADR-004 — Garantizar que todos los tipos de libro son intercambiables en el sistema de préstamos

**Fecha:** 2025-02-24
**Estado:** ✅ Aceptado
**Principio SOLID:** L — Liskov Substitution Principle (LSP)

---

## Contexto

La biblioteca maneja tres tipos de libros: libros físicos, libros digitales y revistas. El equipo modeló esta jerarquía con herencia, haciendo que todos extendieran la clase `Libro`:

**Código actual (con el problema):**

```java
// Libro.java — clase base
public class Libro {
    private Long id;
    private String titulo;
    private String autor;

    public void reservar() {
        // reserva el ejemplar físico
        System.out.println("Ejemplar reservado: " + titulo);
    }

    public void extenderPlazo(int diasExtra) {
        // extiende el préstamo
        System.out.println("Plazo extendido " + diasExtra + " días");
    }
}


// LibroDigital.java
public class LibroDigital extends Libro {

    @Override
    public void reservar() {
        // ❌ Los libros digitales no se reservan: hay copias ilimitadas
        throw new UnsupportedOperationException("Los libros digitales no se reservan");
    }
}


// Revista.java
public class Revista extends Libro {

    @Override
    public void extenderPlazo(int diasExtra) {
        // ❌ Las revistas no se pueden extender: plazo fijo de 3 días
        throw new UnsupportedOperationException("Las revistas no admiten extensión de plazo");
    }
}
```

**¿Cuál es el problema?**

`PrestamoService` trata todos los libros de forma uniforme, pero algunas subclases lanzan excepciones para operaciones que la clase base promete soportar:

```java
// PrestamoService.java
public void procesarSolicitud(Libro libro, String operacion) {
    if (operacion.equals("RESERVAR")) {
        libro.reservar();        // ❌ explota si el libro es LibroDigital
    }
    if (operacion.equals("EXTENDER")) {
        libro.extenderPlazo(7);  // ❌ explota si el libro es Revista
    }
}
```

Esto viola LSP: **no se puede sustituir `Libro` por cualquiera de sus subtipos** sin que el programa falle. El código cliente debe conocer el tipo concreto para evitar errores, lo cual genera verificaciones defensivas con `instanceof` por todo el sistema.

---

## Decisión

Reestructuramos la jerarquía separando las capacidades en **interfaces independientes**. Cada tipo de libro implementa solo las interfaces que genuinamente soporta. `PrestamoService` trabaja con las interfaces, no con la clase base.

**Código corregido:**

```java
// Reservable.java — capacidad de ser reservado
public interface Reservable {
    void reservar();
    void liberarReserva();
}


// Extensible.java — capacidad de extender el plazo
public interface Extensible {
    void extenderPlazo(int diasExtra);
    int getPlazoPorDefecto();
}


// Libro.java — contrato base con solo lo que TODOS los libros comparten
public abstract class Libro {
    private Long id;
    private String titulo;
    private String autor;

    public abstract boolean estaDisponible();
    public abstract String getTipo();

    // getters...
}


// LibroFisico.java — soporta reserva Y extensión
public class LibroFisico extends Libro implements Reservable, Extensible {

    private boolean reservado = false;

    @Override
    public void reservar() {
        this.reservado = true;
        System.out.println("Ejemplar físico reservado: " + getTitulo());
    }

    @Override
    public void liberarReserva() {
        this.reservado = false;
    }

    @Override
    public void extenderPlazo(int diasExtra) {
        System.out.println("Plazo extendido " + diasExtra + " días: " + getTitulo());
    }

    @Override
    public int getPlazoPorDefecto() { return 14; }

    @Override
    public boolean estaDisponible() { return !reservado; }

    @Override
    public String getTipo() { return "FISICO"; }
}


// LibroDigital.java — NO es reservable (copias ilimitadas), SÍ es extensible
public class LibroDigital extends Libro implements Extensible {

    @Override
    public void extenderPlazo(int diasExtra) {
        System.out.println("Acceso digital extendido " + diasExtra + " días: " + getTitulo());
    }

    @Override
    public int getPlazoPorDefecto() { return 7; }

    @Override
    public boolean estaDisponible() { return true; } // siempre disponible

    @Override
    public String getTipo() { return "DIGITAL"; }
}


// Revista.java — SÍ es reservable, NO es extensible (plazo fijo)
public class Revista extends Libro implements Reservable {

    private boolean reservada = false;

    @Override
    public void reservar() {
        this.reservada = true;
        System.out.println("Revista reservada: " + getTitulo());
    }

    @Override
    public void liberarReserva() {
        this.reservada = false;
    }

    @Override
    public boolean estaDisponible() { return !reservada; }

    @Override
    public String getTipo() { return "REVISTA"; }
}


// PrestamoService.java — trabaja con interfaces, no con tipos concretos
@Service
public class PrestamoService {

    public void reservar(Libro libro) {
        // ✅ Solo actúa si el libro realmente soporta reserva
        if (libro instanceof Reservable reservable) {
            reservable.reservar();
        } else {
            System.out.println(libro.getTipo() + " no requiere reserva previa.");
        }
    }

    public void extenderPlazo(Libro libro, int diasExtra) {
        // ✅ Solo actúa si el libro realmente soporta extensión
        if (libro instanceof Extensible extensible) {
            extensible.extenderPlazo(diasExtra);
        } else {
            System.out.println(libro.getTipo() + " tiene plazo fijo, no se puede extender.");
        }
    }
}
```

**Prueba unitaria que demuestra la sustitución correcta:**

```java
class LibroTest {

    @Test
    void libroFisico_debeReservarseSinExcepcion() {
        Libro libro = new LibroFisico();
        // ✅ Se puede sustituir Libro por LibroFisico sin que nada explote
        assertDoesNotThrow(() -> {
            if (libro instanceof Reservable r) r.reservar();
            if (libro instanceof Extensible e) e.extenderPlazo(7);
        });
    }

    @Test
    void libroDigital_noLanzaExcepcionPorNoSerReservable() {
        Libro libro = new LibroDigital();
        // ✅ No lanza excepción: simplemente no implementa Reservable
        assertFalse(libro instanceof Reservable);
        assertTrue(libro instanceof Extensible);
    }

    @Test
    void revista_noLanzaExcepcionPorNoSerExtensible() {
        Libro libro = new Revista();
        // ✅ No lanza excepción: simplemente no implementa Extensible
        assertTrue(libro instanceof Reservable);
        assertFalse(libro instanceof Extensible);
    }
}
```

### Principio SOLID aplicado — LSP

> "Los subtipos deben poder sustituir a sus tipos base sin alterar la corrección del programa."  
> — Barbara Liskov, 1987

**Antes:** sustituir `Libro` por `LibroDigital` o `Revista` podía lanzar `UnsupportedOperationException` en tiempo de ejecución. El contrato de la clase base se rompía en las subclases.

**Después:** cada subtipo cumple completamente el contrato de las interfaces que declara implementar. Ninguna sustitución lanza excepciones inesperadas.

```
ANTES:
  Libro.reservar()          → LibroDigital lanza excepción  ❌
  Libro.extenderPlazo()     → Revista lanza excepción       ❌

DESPUÉS:
  Reservable.reservar()     → solo quien puede hacerlo lo declara  ✅
  Extensible.extenderPlazo()→ solo quien puede hacerlo lo declara  ✅
```

**Señal de alerta LSP:** si encuentras `UnsupportedOperationException`, `instanceof` en cascada, o métodos vacíos heredados que "no aplican", probablemente hay una violación de LSP.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Verificar el tipo con `instanceof` antes de cada operación | Funciona pero obliga al cliente a conocer los tipos concretos. El problema de LSP se traslada al código cliente |
| Tener una clase base diferente por cada tipo de libro sin jerarquía común | Pierde el polimorfismo útil: no se puede tratar una lista mixta de libros de forma uniforme |

---

## Consecuencias

### Positivas
- Ninguna operación lanza `UnsupportedOperationException`. El contrato siempre se cumple.
- Se eliminan los `instanceof` defensivos del código de servicio.
- Añadir un nuevo tipo de libro (ej. `Audiolibro`) solo requiere decidir qué interfaces implementa.
- Las pruebas unitarias son predecibles: no hay rutas de excepción ocultas.

### Negativas / trade-offs
- La jerarquía se vuelve más horizontal (más interfaces, menos herencia profunda). Requiere mayor atención al diseño inicial.
- El uso de `instanceof` en `PrestamoService` para despachar sigue siendo necesario; se puede eliminar con doble despacho (Visitor pattern) si la jerarquía crece mucho.
