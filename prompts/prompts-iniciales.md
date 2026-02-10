# Prompts iniciales para la generación del pipeline

Este documento recopila los prompts diseñados para generar paso a paso el archivo `.github/workflows/pipeline.yml`. Estos prompts están pensados para ser utilizados con un asistente de IA (como ChatGPT o Claude) para obtener una configuración robusta y alineada con las mejores prácticas.

## 1. Prompt para la creación de los tests de backend

Este prompt se encarga de configurar el entorno de ejecución, la base de datos de prueba y la ejecución de los tests automatizados.

```markdown
Actúa como un experto en DevOps y GitHub Actions. Necesito configurar la primera fase de un pipeline de CI/CD para una aplicación backend en Node.js (TypeScript) que utiliza Prisma y Jest.

Por favor, genera un job llamado build-and-test que cumpla con lo siguiente:
1.  **Trigger**: Debe ejecutarse en Pull Requests y pushes a main.
2.  **Entorno**: Runs-on `ubuntu-latest`.
3.  **Servicios**: Incluye un contenedor de servicio PostgreSQL (postgres:15) para los tests de integración. Configura puertos y healthchecks adecuados.
4.  **Pasos**:
    *   Checkout del código.
    *   Setup de Node.js (versión 20).
    *   Instalación de dependencias (usando `npm ci` para entornos limpios).
    *   Ejecución de tests (`npm test`).
5.  **Variables**: Asegura que la variable de entorno `DATABASE_URL` esté disponible y apunte al servicio de postgres creado.

El código debe ser limpio y usar las acciones oficiales `actions/checkout` y `actions/setup-node`.
```

## 2. Prompt para la generación del build del backend

Este prompt se centra en la compilación del código TypeScript y la preparación de los artefactos desplegables.

```markdown
Ahora necesito añadir una fase de construcción (job 'build') que se ejecute después de que los tests pasen exitosamente.

Instrucciones para este job:
1.  **Dependencia**: Debe esperar a que el job de tests finalice correctamente (`needs: test`).
2.  **Preparación**: Realiza checkout e instala dependencias nuevamente (para tener acceso a `tsc`).
3.  **Compilación**: Ejecuta el script de build (`npm run build`).
4.  **Artefactos**:
    *   Crea un paquete que incluya *solo* lo necesario para producción: la carpeta compilada (`dist`), `package.json`, `package-lock.json` y la carpeta `prisma` (para migraciones/generación de cliente).
    *   Sube este paquete como un artefacto de GitHub llamado `backend-build`.

El objetivo es que este artefacto sea descargado posteriormente por el job de despliegue.
```

## 3. Prompt para el despliegue del backend en EC2

Este prompt define el proceso de despliegue continuo en AWS EC2, gestionando la transferencia de archivos y el reinicio de servicios.

```markdown
Finalmente, genera el job de despliegue ('deploy') para AWS EC2.

Requisitos:
1.  **Dependencia**: Debe ejecutarse solo si el job 'build' tuvo éxito.
2.  **Entorno**: Ubuntu-latest.
3.  **Transferencia de archivos**:
    *   Descarga el artefacto `backend-build` generado anteriormente.
    *   Usa la acción `appleboy/scp-action` para copiar los archivos a la instancia EC2.
    *   Ruta destino en el servidor: `/home/ubuntu/app/`.
    *   Usa secrets de GitHub (`EC2_HOST`, `EC2_USER`, `EC2_KEY`) para la conexión.
4.  **Ejecución remota**:
    *   Usa la acción `appleboy/ssh-action` para conectarte por SSH y ejecutar los siguientes comandos en el servidor:
        1.  Navegar al directorio de la app.
        2.  Instalar dependencias de producción (`npm ci --omit=dev`).
        3.  Generar el cliente de Prisma (`npx prisma generate`).
        4.  Reiniciar la aplicación usando PM2 (`pm2 reload backend --update-env` o `pm2 start` si no existe).
5.  **Variables de Entorno**: Inyecta la variable `DATABASE_URL` desde los secrets de GitHub para que la aplicación en producción pueda conectarse a la base de datos real.

Asegúrate de manejar posibles errores y que el script sea idempotente (pueda ejecutarse múltiples veces sin fallar).
```
