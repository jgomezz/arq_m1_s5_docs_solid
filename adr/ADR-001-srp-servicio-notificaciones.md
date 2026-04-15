# ADR-001 — Separar el envío de notificaciones del servicio de préstamos

**Fecha:** 2025-02-03
**Estado:** ✅ Aceptado
**Principio SOLID:** S — Single Responsibility Principle (SRP)

---

## Contexto

La clase `PrestamoService` es responsable de registrar préstamos y, además, de enviar correos electrónicos al usuario cuando se registra o devuelve un libro.

**Código actual (con el problema):**

```java
// PrestamoService.java — tiene más de una responsabilidad
@Service
public class PrestamoService {

    private final PrestamoRepository repository;

    public PrestamoService(PrestamoRepository repository) {
        this.repository = repository;
    }

    public Prestamo registrarPrestamo(Usuario usuario, Libro libro) {
        Prestamo prestamo = new Prestamo(usuario, libro, LocalDate.now().plusDays(14));
        repository.save(prestamo);

        // ❌ Responsabilidad de notificación mezclada con lógica de préstamo
        JavaMailSenderImpl mailSender = new JavaMailSenderImpl();
        mailSender.setHost("smtp.biblioteca.com");
        SimpleMailMessage mensaje = new SimpleMailMessage();
        mensaje.setTo(usuario.getEmail());
        mensaje.setSubject("Préstamo registrado");
        mensaje.setText("Tienes 14 días para devolver: " + libro.getTitulo());
        mailSender.send(mensaje);

        return prestamo;
    }
}
```

**¿Cuál es el problema?**

`PrestamoService` tiene **dos razones para cambiar**:
- Si cambia la regla de negocio del préstamo (por ejemplo, de 14 a 21 días).
- Si cambia el proveedor de correo o el texto de la notificación.

Esto también dificulta las pruebas: para testear `registrarPrestamo` hay que configurar un servidor SMTP real o usar mocks complejos.

---

## Decisión

Extraemos el envío de notificaciones a una clase dedicada `NotificacionService`. `PrestamoService` delega en ella sin conocer los detalles del envío.

**Código corregido:**

```java
// NotificacionService.java — responsabilidad única: enviar notificaciones
@Service
public class NotificacionService {

    private final JavaMailSender mailSender;

    public NotificacionService(JavaMailSender mailSender) {
        this.mailSender = mailSender;
    }

    public void notificarPrestamo(Usuario usuario, Libro libro) {
        SimpleMailMessage mensaje = new SimpleMailMessage();
        mensaje.setTo(usuario.getEmail());
        mensaje.setSubject("Préstamo registrado");
        mensaje.setText("Tienes 14 días para devolver: " + libro.getTitulo());
        mailSender.send(mensaje);
    }

    public void notificarDevolucion(Usuario usuario, Libro libro) {
        SimpleMailMessage mensaje = new SimpleMailMessage();
        mensaje.setTo(usuario.getEmail());
        mensaje.setSubject("Devolución registrada");
        mensaje.setText("Gracias por devolver: " + libro.getTitulo());
        mailSender.send(mensaje);
    }
}


// PrestamoService.java — responsabilidad única: gestionar préstamos
@Service
public class PrestamoService {

    private final PrestamoRepository repository;
    private final NotificacionService notificaciones;

    public PrestamoService(PrestamoRepository repository,
                           NotificacionService notificaciones) {
        this.repository = repository;
        this.notificaciones = notificaciones;
    }

    public Prestamo registrarPrestamo(Usuario usuario, Libro libro) {
        Prestamo prestamo = new Prestamo(usuario, libro, LocalDate.now().plusDays(14));
        repository.save(prestamo);

        // ✅ Delega sin conocer cómo se envía la notificación
        notificaciones.notificarPrestamo(usuario, libro);

        return prestamo;
    }
}
```

### Principio SOLID aplicado — SRP

> "Un módulo debe tener una, y solo una, razón para cambiar."

| Clase | Única razón de cambio |
|-------|----------------------|
| `PrestamoService` | Reglas de negocio del préstamo |
| `NotificacionService` | Canales o contenido de notificaciones |

**Antes:** un cambio en el proveedor de correo obligaba a modificar `PrestamoService`.  
**Después:** ese cambio solo afecta a `NotificacionService`.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Dejar todo en `PrestamoService` y solo extraer la configuración SMTP | El problema de fondo persiste: la clase sigue teniendo dos razones de cambio |
| Usar eventos de dominio (Spring Events) para desacoplar completamente | Válido a futuro, pero añade complejidad innecesaria para el estado actual del sistema |

---

## Consecuencias

### Positivas
- `PrestamoService` se puede probar con un mock simple de `NotificacionService`, sin configurar SMTP.
- Cambiar el proveedor de correo no toca la lógica de préstamos.
- `NotificacionService` puede reutilizarse desde otros servicios (ej. recordatorios de vencimiento).

### Negativas / trade-offs
- Se añade una clase y una dependencia más. Para sistemas muy pequeños puede parecer excesivo.
- El flujo completo ahora involucra dos clases en lugar de una.
