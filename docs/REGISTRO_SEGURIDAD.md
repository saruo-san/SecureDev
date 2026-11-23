# Registro de seguridad · Sistema Alumnos

Estudiante: Javier Alexis Muñoz Toro   Repositorio: https://github.com/saruo-san/SecureDev   Fecha: 2026-11-23

Completa este registro a medida que avanzas. Cada fila debe poder seguirse hasta un **build de Jenkins**
y un **commit** (usa el identificador `VULN-xx` en el mensaje del commit).

## 1. Revisiones de seguridad realizadas

| #  | Etapa del SDLC            | Herramienta / técnica   | Build | Resultado (resumen)                                                                                                                                                                         |
| -- | ------------------------- | ------------------------ | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R1 | Código (SAST)            | Bandit en el pipeline    | #2    | 2 hallazgos High / 2 Medium / 1 Low (0/0/0 tras parches en Build #3 y #5)                                                                                                                   |
| R2 | Dependencias (SCA)        | pip-audit en el pipeline | #2    | 19 vulnerabilidades en 4 paquetes (0 vulnerabilidades tras PRs de Dependabot en Build #5)                                                                                                   |
| R3 | Pruebas dinámicas (DAST) | OWASP ZAP sobre staging  | #2    | 3 High / 3 Medium / 1 Low (0 High / 0 Medium / 1 Low tras parches en Build #3 y #5)                                                                                                         |
| R4 | Revisión manual          | Lectura de `app.py`    | —    | Se identificó clave secreta hardcodeada, credenciales con MD5 débil, SQLi en `/alumnos/buscar`, XSS en `/hello`, `debug=True` y ausencia de protección anti-CSRF                   |
| R5 | Producción               | Grafana + Prometheus     | #5    | Targets `produccion` (puerto 5000) y `staging` (puerto 5001) en estado `UP`; dashboard de seguridad y alerta `Alerta Fuerza Bruta Login` operativos y validados con ataque simulado |

## 2. Vulnerabilidades identificadas y corregidas

| ID      | Vulnerabilidad (CWE)                                                                                 | Detectada por                               | Ubicación                | Riesgo | Corrección aplicada                                                                                                                                                                                                           | Commit      | Build que lo verifica |
| ------- | ---------------------------------------------------------------------------------------------------- | ------------------------------------------- | ------------------------- | ------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ----------- | --------------------- |
| VULN-01 | Uso de secreto hardcodeado y hash criptográfico débil MD5 (CWE-798, CWE-328)                       | Bandit (B105, B324) y revisión manual      | `app.py:20-23, 217`     | Alto   | Se reemplazó `SECRET_KEY` fija por `os.environ.get("SECRET_KEY") or secrets.token_hex(32)` y se migró la verificación de clave de MD5 a `werkzeug.security.generate_password_hash` / `check_password_hash` (PBKDF2) | `8f5f8ec` | #3                    |
| VULN-02 | Inyección SQL y exposición de mensajes internos de error (CWE-89, CWE-209)                         | Bandit (B608), OWASP ZAP y revisión manual | `app.py:202-209`        | Alto   | Se reemplazó la concatenación de strings (`f"... LIKE '%{nombre}%'"`) por consulta parametrizada (`WHERE nombre LIKE ?`) y se ocultó el detalle técnico de `sqlite3.Error`                                           | `2b8b9d5` | #3                    |
| VULN-03 | Cross-Site Scripting (XSS) reflejado y DOM (CWE-79)                                                  | OWASP ZAP y revisión manual                | `app.py:155-157`        | Alto   | Se sanitizó el parámetro `name` en `/hello` utilizando `markupsafe.escape()` antes de renderizar la respuesta HTML                                                                                                     | `28bd8e3` | #3                    |
| VULN-04 | Ejecución con `debug=True` activo y enlace a todas las interfaces `0.0.0.0` (CWE-489, CWE-605)  | Bandit (B201, B104)                         | `app.py:239`            | Alto   | Se estableció `debug=False` por defecto y se restringió el enlace por defecto a `127.0.0.1` configurable mediante variables de entorno `FLASK_HOST` y `FLASK_PORT`                                                   | `66ae0cb` | #3                    |
| VULN-05 | Ausencia de cabeceras HTTP de seguridad (CSP, Anti-Clickjacking y MIME Sniffing) (CWE-693, CWE-1021) | OWASP ZAP                                   | `app.py:130-139`        | Medio  | Se añadieron las cabeceras `Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff` y `Referrer-Policy` en `@app.after_request`                                                        | `35aa737` | #3                    |
| VULN-06 | Ausencia de tokens Anti-CSRF en formulario de autenticación (CWE-352)                               | OWASP ZAP y revisión manual                | `app.py:68-71, 214-228` | Medio  | Se implementó generación de token aleatorio por sesión (`secrets.token_hex(32)`), campo oculto `csrf_token` en portada y validación con `secrets.compare_digest` en `POST /login`                                  | `75c2b6b` | #3                    |

## 3. Resultados de OWASP ZAP

| Alerta de ZAP                                | Riesgo | URL de ejemplo                                    | VULN relacionada | Cómo se mitiga                                                                                               | ¿Mitigada? Build |
| -------------------------------------------- | ------ | ------------------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------- | ----------------- |
| Cross Site Scripting (Reflected) / DOM Based | High   | `http://jenkins:5001/hello?name=...`            | VULN-03          | Escape de salida HTML con `markupsafe.escape()` en `/hello`                                               | Sí (#3 y #5)     |
| SQL Injection - SQLite                       | High   | `http://jenkins:5001/alumnos/buscar?nombre=...` | VULN-02          | Consulta SQL parametrizada con `?` y manejo genérico de excepciones                                        | Sí (#3 y #5)     |
| Absence of Anti-CSRF Tokens                  | Medium | `http://jenkins:5001/` (`POST /login`)        | VULN-06          | Inclusión y validación en servidor de token anti-CSRF ligado a la sesión                                   | Sí (#3 y #5)     |
| Content Security Policy (CSP) Header Not Set | Medium | `http://jenkins:5001/`                          | VULN-05          | Configuración de `Content-Security-Policy: default-src 'self'; form-action 'self'; frame-ancestors 'none'` | Sí (#3 y #5)     |
| Missing Anti-clickjacking Header             | Medium | `http://jenkins:5001/`                          | VULN-05          | Inclusión de cabecera `X-Frame-Options: DENY` en todas las respuestas                                      | Sí (#3 y #5)     |
| X-Content-Type-Options Header Missing        | Low    | `http://jenkins:5001/`                          | VULN-05          | Inclusión de cabecera `X-Content-Type-Options: nosniff`                                                    | Sí (#3 y #5)     |

## 4. Monitoreo e incidentes

| Hora  | Panel / métrica de Grafana                                                | Qué se observó                                                                                                                                                                                                                                           | Entorno                               | Clasificación                                                       | Acción de respuesta                                                                                                                                                                           | Verificación                                                                                            |
| ----- | -------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------- | -------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| 18:32 | `Logins Fallidos (Fuerza Bruta)` (`login_fallidos_total`)              | Incremento súbito de 0 a 30 intentos fallidos consecutivos contra `POST /login` (`instance="jenkins:5000"`) y disparo en rojo (`1 firing`) de la regla `Alerta Fuerza Bruta Login` (`increase(login_fallidos_total{job="produccion"}[5m]) > 5`) | Producción (`http://jenkins:5000`) | Ataque de fuerza bruta simulado (`simular_ataque.py`)              | Validación de alertas activas en Grafana, revisión de logs de autenticación y rotación de credencial por defecto `admin` mediante variable de entorno `ADMIN_CLAVE_HASH` en despliegue | Confirmado en dashboard `Seguridad - Sistema Alumnos` y vista `Alerting -> Alert rules` (`Firing`) |
| 18:32 | `Peticiones Sospechosas (XSS / SQLi)` (`peticiones_sospechosas_total`) | Detección e incremento de contadores `tipo="sqli"` (`UNION SELECT ... sqlite_master`) y `tipo="xss"` (`<script>alert('xss')</script>`) en Producción                                                                                             | Producción (`http://jenkins:5000`) | Intento de explotación SQLi y XSS reflejado (`simular_ataque.py`) | Verificación de que la consulta parametrizada (`VULN-02`) no filtró esquema SQLite y el escape HTML (`VULN-03`) neutralizó la ejecución del script                                     | Salida de consola confirmó HTTP 200 sin fuga de datos internos ni ejecución de script                  |

## 5. Gestión de dependencias

| Paquete           | Versión inicial | Vulnerabilidades (IDs)                                                                                                                 | Versión final | Origen del cambio (PR de Dependabot / manual)                  | ¿Pruebas, pip-audit y ZAP siguen OK? Build                       |
| ----------------- | ---------------- | -------------------------------------------------------------------------------------------------------------------------------------- | -------------- | -------------------------------------------------------------- | ----------------------------------------------------------------- |
| Flask             | 2.2.2            | GHSA-m2qf-hwjv-5gpq, GHSA-4grg-w6v8-c28g (2 CVEs)                                                                                      | 3.1.3          | PR de Dependabot (`dependabot/pip/flask-3.1.3`)              | Sí (9/9 pruebas, 0 CVEs en `pip-audit`, ZAP 0H/0M) — Build #5 |
| Werkzeug          | 2.2.2            | GHSA-px8h-6qhx-m222, GHSA-f9vj-2wh5-fj8j, GHSA-2g68-c3qc-8985, GHSA-q34m-jh98-gwm2, GHSA-hgf8-39gv-g3f2, GHSA-87hc-h4r5-73f7 (12 CVEs) | 3.1.9          | PR de Dependabot (`dependabot/pip/werkzeug-3.1.9`)           | Sí (9/9 pruebas, 0 CVEs en `pip-audit`, ZAP 0H/0M) — Build #5 |
| Jinja2            | 3.1.2            | GHSA-h5c8-rqwp-cp95, GHSA-h75v-3vvj-5mfj, GHSA-q2x7-8rv6-6q7h, GHSA-gmj6-6f8f-6699, GHSA-cpwx-vrp4-4pq7 (5 CVEs)                       | 3.1.6          | PR de Dependabot (`dependabot/pip/jinja2-3.1.6`)             | Sí (9/9 pruebas, 0 CVEs en `pip-audit`, ZAP 0H/0M) — Build #5 |
| gunicorn          | 21.2.0           | Actualización preventiva de seguridad y compatibilidad                                                                                | 27.0.1         | PR #1 de Dependabot (`dependabot/pip/gunicorn-27.0.1`)       | Sí (9/9 pruebas, 0 CVEs en `pip-audit`, ZAP 0H/0M) — Build #5 |
| prometheus-client | 0.17.1           | Sin CVEs conocidos; actualizado por mantenimiento preventivo                                                                           | 0.26.0         | PR de Dependabot (`dependabot/pip/prometheus-client-0.26.0`) | Sí (9/9 pruebas, 0 CVEs en `pip-audit`, ZAP 0H/0M) — Build #5 |

## 6. Trazabilidad

 `TRAZABILIDAD.md` del último build y completa un recorrido por cada actividad del pipeline:

```markdown
# Trazabilidad del build 5

- **Build:** 5
- **Fecha (UTC):** 2026-11-23 20:58
- **Versión:** 1.0.5+1e50a49
- **Commit:** `1e50a49`
- **Mensaje del commit:** Merge pull request #1 from saruo-san/dependabot/pip/gunicorn-27.0.1
- **Pruebas:** 9/9 OK
- **Bandit H/M/L:** n/a
- **pip-audit:** n/a
- **ZAP H/M/L:** 0/0/1
- **Resultado:** SUCCESS
- **Producción:** 1.0.5+1e50a49

- **Autor del commit:** Jav1er
- **Enlace al build:** http://localhost:8080/job/SecureDev-Pipeline/5/
- **Reportes de este build:** junit.xml, bandit.txt, pip-audit.txt, zap-report.html
```
