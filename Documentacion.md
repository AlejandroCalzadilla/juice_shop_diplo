
## Proyecto 

nombre OWASP Juice Shop
proyecto tipo laboratorio con vulnerabilidades de la 

![alt text](image.png)

## Tabla de Amenazas STRIDE

| Componente   | Tipo (STRIDE)           | Descripción                                                                 | Estado  | Prioridad | Mitigación                                                        |
|--------------|-------------------------|-----------------------------------------------------------------------------|---------|-----------|-------------------------------------------------------------------|
| Cliente      | Spoofing                | Un atacante puede suplantar a otro usuario mediante credenciales débiles.   | Abierto | Alta      | Autenticación robusta, contraseñas fuertes, tokens seguros.       |
| Cliente      | Repudiation             | El usuario puede negar acciones si no existen registros adecuados.           | Abierto | Media     | Implementar logs y auditoría de acciones.                         |




| Componente   | Tipo (STRIDE)           | Descripción                                                                 | Estado  | Prioridad | Mitigación                                                        |
|--------------|-------------------------|-----------------------------------------------------------------------------|---------|-----------|-------------------------------------------------------------------|
| Servidor     | Spoofing                | El servidor acepta conexiones no autenticadas o de fuentes no confiables.   | Abierto | Media     | Validar origen y autenticación de usuarios.                       |
| Servidor     | Tampering               | Modificación de datos en tránsito entre servidor y base de datos.            | Abierto | Alta      | Validar y sanear datos antes de almacenarlos.                     |
| Servidor     | Repudiation             | Acciones no registradas en el servidor.                                     | Abierto | Media     | Auditoría y logs detallados.                                      |
| Servidor     | Information Disclosure  | Filtrado de información sensible por errores o configuraciones inseguras.    | Abierto | Crítica   | Configuración segura y control de errores.                        |
| Servidor     | Denial of Service       | Saturación por ataques de fuerza bruta o peticiones masivas.                 | Abierto | Alta      | Protección contra DoS y balanceo de carga.                        |
| Servidor     | Elevation of Privilege  | Vulnerabilidades permiten acceso a funciones administrativas.                | Abierto | Crítica   | Control estricto de permisos y validación de entradas.            |







| Componente   | Tipo (STRIDE)           | Descripción                                                                 | Estado  | Prioridad | Mitigación                                                        |
|--------------|-------------------------|-----------------------------------------------------------------------------|---------|-----------|-------------------------------------------------------------------|
| Base de Datos| Tampering               | Inyecciones NoSQL modifican datos maliciosamente.                           | Abierto | Crítica   | Validar y sanear consultas.                                       |
| Base de Datos| Repudiation             | Cambios no rastreables en la base de datos.                                 | Abierto | Media     | Auditoría y logs de cambios.                                      |
| Base de Datos| Information Disclosure  | Consultas mal diseñadas exponen datos sensibles.                            | Abierto | Crítica   | Control de acceso y consultas seguras.                            |
| Base de Datos| Denial of Service       | Consultas pesadas o maliciosas saturan la base de datos.                    | Abierto | Alta      | Optimización y límites de recursos.                               |
