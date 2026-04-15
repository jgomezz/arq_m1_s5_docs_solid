# ADR-005 — Segregar la interfaz de gestión de usuarios según su rol

**Fecha:** 2025-03-03
**Estado:** ✅ Aceptado
**Principio SOLID:** I — Interface Segregation Principle (ISP)

---

## Contexto

El sistema tiene tres actores que interactúan con los datos de un usuario: el **Bibliotecario** (gestiona préstamos), el **Administrador** (gestiona cuentas y bloqueos) y el **Usuario** (consulta su propio historial). Los tres usan el mismo servicio `UsuarioService`, que expone una única interfaz con todos los métodos del sistema:

**Código actual (con el problema):**

```java
// UsuarioService.java — interfaz única con demasiados métodos
public interface UsuarioService {

    // Operaciones que usa el Bibliotecario
    Usuario buscarPorId(Long id);
    List<Prestamo> obtenerPrestamosActivos(Long usuarioId);

    // Operaciones que usa el Administrador
    void bloquearUsuario(Long id, String motivo);
    void desbloquearUsuario(Long id);
    void cambiarTipoUsuario(Long id, String nuevoTipo);
    void eliminarUsuario(Long id);

    // Operaciones que usa el propio Usuario
    void actualizarEmail(Long id, String nuevoEmail);
    HistorialPrestamos verHistorial(Long id);
}
```

**Código que consume la interfaz:**

```java
// BibliotecarioController.java
@RestController
public class BibliotecarioController {

    // ❌ Depende de UsuarioService completo, pero solo usa 2 métodos
    private final UsuarioService usuarioService;

    public BibliotecarioController(UsuarioService usuarioService) {
        this.usuarioService = usuarioService;
    }

    @GetMapping("/usuarios/{id}/prestamos")
    public List<Prestamo> verPrestamos(@PathVariable Long id) {
        return usuarioService.obtenerPrestamosActivos(id);
    }
}
```

**¿Cuál es el problema?**

`BibliotecarioController` depende de `UsuarioService`, pero de sus 8 métodos solo usa 2. Esto genera:

- **Acoplamiento innecesario:** un cambio en `eliminarUsuario` (que el bibliotecario nunca llama) puede obligar a recompilar y desplegar `BibliotecarioController`.
- **Dificultad para crear dobles de prueba:** el mock de `UsuarioService` en los tests de `BibliotecarioController` debe implementar los 8 métodos aunque solo 2 sean relevantes.
- **Interfaces que "mienten":** al ver que `BibliotecarioController` depende de `UsuarioService`, un desarrollador nuevo asume que usa toda la interfaz, cuando en realidad usa una fracción mínima.
- **Violación de mínimo privilegio:** el Bibliotecario tiene acceso (técnico) a `eliminarUsuario` aunque no debería tenerlo por diseño.

---

## Decisión

Dividimos `UsuarioService` en tres interfaces específicas, una por cada actor del sistema. Cada controlador depende solo de la interfaz que necesita.

**Código corregido:**

```java
// ConsultaUsuarioService.java — lo que necesita el Bibliotecario
public interface ConsultaUsuarioService {
    Usuario buscarPorId(Long id);
    List<Prestamo> obtenerPrestamosActivos(Long usuarioId);
}


// AdministracionUsuarioService.java — lo que necesita el Administrador
public interface AdministracionUsuarioService {
    void bloquearUsuario(Long id, String motivo);
    void desbloquearUsuario(Long id);
    void cambiarTipoUsuario(Long id, String nuevoTipo);
    void eliminarUsuario(Long id);
}


// PerfilUsuarioService.java — lo que necesita el propio Usuario
public interface PerfilUsuarioService {
    void actualizarEmail(Long id, String nuevoEmail);
    HistorialPrestamos verHistorial(Long id);
}


// UsuarioServiceImpl.java — implementación única que cumple los tres contratos
@Service
public class UsuarioServiceImpl
        implements ConsultaUsuarioService,
                   AdministracionUsuarioService,
                   PerfilUsuarioService {

    private final UsuarioRepositorio repositorio;

    public UsuarioServiceImpl(UsuarioRepositorio repositorio) {
        this.repositorio = repositorio;
    }

    // --- ConsultaUsuarioService ---
    @Override
    public Usuario buscarPorId(Long id) {
        return repositorio.buscarPorId(id)
            .orElseThrow(() -> new UsuarioNoEncontradoException(id));
    }

    @Override
    public List<Prestamo> obtenerPrestamosActivos(Long usuarioId) {
        return repositorio.listarPrestamosActivos(usuarioId);
    }

    // --- AdministracionUsuarioService ---
    @Override
    public void bloquearUsuario(Long id, String motivo) {
        Usuario usuario = buscarPorId(id);
        usuario.bloquear(motivo);
        repositorio.guardar(usuario);
    }

    @Override
    public void desbloquearUsuario(Long id) {
        Usuario usuario = buscarPorId(id);
        usuario.desbloquear();
        repositorio.guardar(usuario);
    }

    @Override
    public void cambiarTipoUsuario(Long id, String nuevoTipo) {
        Usuario usuario = buscarPorId(id);
        usuario.setTipo(nuevoTipo);
        repositorio.guardar(usuario);
    }

    @Override
    public void eliminarUsuario(Long id) {
        repositorio.eliminar(id);
    }

    // --- PerfilUsuarioService ---
    @Override
    public void actualizarEmail(Long id, String nuevoEmail) {
        Usuario usuario = buscarPorId(id);
        usuario.setEmail(nuevoEmail);
        repositorio.guardar(usuario);
    }

    @Override
    public HistorialPrestamos verHistorial(Long id) {
        return repositorio.obtenerHistorial(id);
    }
}


// ✅ Cada controlador depende solo de lo que usa
@RestController
@RequestMapping("/bibliotecario")
public class BibliotecarioController {

    private final ConsultaUsuarioService consulta; // solo 2 métodos visibles

    public BibliotecarioController(ConsultaUsuarioService consulta) {
        this.consulta = consulta;
    }

    @GetMapping("/usuarios/{id}")
    public Usuario verUsuario(@PathVariable Long id) {
        return consulta.buscarPorId(id);
    }

    @GetMapping("/usuarios/{id}/prestamos")
    public List<Prestamo> verPrestamos(@PathVariable Long id) {
        return consulta.obtenerPrestamosActivos(id);
    }
}


@RestController
@RequestMapping("/admin")
public class AdminController {

    private final AdministracionUsuarioService adminService; // solo métodos de admin

    public AdminController(AdministracionUsuarioService adminService) {
        this.adminService = adminService;
    }

    @PutMapping("/usuarios/{id}/bloquear")
    public void bloquear(@PathVariable Long id, @RequestBody String motivo) {
        adminService.bloquearUsuario(id, motivo);
    }

    @DeleteMapping("/usuarios/{id}")
    public void eliminar(@PathVariable Long id) {
        adminService.eliminarUsuario(id);
    }
}
```

**Prueba unitaria simplificada gracias a la segregación:**

```java
class BibliotecarioControllerTest {

    @Test
    void verPrestamos_debeRetornarListaDelServicio() {
        // ✅ Mock de solo 2 métodos, no de 8
        ConsultaUsuarioService mockConsulta = mock(ConsultaUsuarioService.class);
        List<Prestamo> prestamos = List.of(new Prestamo(), new Prestamo());
        when(mockConsulta.obtenerPrestamosActivos(1L)).thenReturn(prestamos);

        BibliotecarioController controller = new BibliotecarioController(mockConsulta);
        List<Prestamo> resultado = controller.verPrestamos(1L);

        assertEquals(2, resultado.size());
    }
}
```

### Principio SOLID aplicado — ISP

> "Los clientes no deben verse forzados a depender de interfaces que no usan."  
> — Robert C. Martin

**Antes:** `BibliotecarioController` dependía de una interfaz con 8 métodos y usaba 2.  
**Después:** depende de `ConsultaUsuarioService` con exactamente los 2 métodos que necesita.

```
ANTES:
  BibliotecarioController → UsuarioService (8 métodos)
                            usa: 2   ignora: 6   ❌

DESPUÉS:
  BibliotecarioController → ConsultaUsuarioService (2 métodos)
                            usa: 2   ignora: 0   ✅

  AdminController         → AdministracionUsuarioService (4 métodos)
  PerfilController        → PerfilUsuarioService (2 métodos)
```

**Regla práctica para detectar violaciones de ISP:** si al crear el mock de una dependencia en un test debes implementar métodos que el test nunca llama, probablemente la interfaz es demasiado grande.

### Alternativas consideradas

| Alternativa | Por qué se descartó |
|-------------|---------------------|
| Mantener una sola interfaz y documentar qué métodos usa cada actor | La documentación se desactualiza; el compilador no la verifica. La segregación es estructural, no documental |
| Crear tres clases de servicio separadas sin interfaz común | Duplica la implementación. La solución correcta es una implementación con múltiples contratos |

---

## Consecuencias

### Positivas
- Los mocks en los tests de cada controlador son simples: implementan solo los métodos relevantes.
- Un cambio en `eliminarUsuario` no recompila ni impacta `BibliotecarioController`.
- El diseño refleja el principio de mínimo privilegio: cada actor solo ve las operaciones que le corresponden.
- Incorporar un nuevo actor (ej. `Auditor` que solo lee) requiere una nueva interfaz pequeña, sin tocar las existentes.

### Negativas / trade-offs
- Más interfaces en el proyecto. Para sistemas pequeños puede parecer excesivo; se justifica cuando hay múltiples actores con necesidades distintas.
- El desarrollador debe recordar qué interfaz inyectar en cada componente. Una buena configuración del contenedor de Spring mitiga esto.
