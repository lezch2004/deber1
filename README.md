Nombre: Luis Eduardo Zaldumbide 
Clase: Data Mining 
Proyecto: Deber 1
-----------
1. Descripción y diagrama de arquitectura
    a. Descripción 
    El proyecto busca implementar una serie de conocimientos en todo lo que es minería de datos y los procesos devops relacionados con ETL. Esto se logra con el uso de el API de QuickBooks Online (extract), MageAI (transform), y PostgreSQL (load), utilizando las mejores prácticas en ingeniería de datos como el manejo de chunking, paginación, políticas de reintentos, etc... El proyecto utiliza tres pipelines en MageAI para poder manejar la ingesta de datos para el backfill parametrizado de invoices, customers y items desde Quickbooks Online, a travez de su API, en PostgreSQl que actúa como warehouse. Cada uno de estos pipelines cuenta con un trigger de ejecución única, logs para garantizar la integridad e idempotencia, y el uso de Mage Secrets para almacenar los valores sensibles como los tokens o las claves. En este archivo se buscará describir a profundidad el proyecto, y proporcionar evidencia de su estructura y uso. 

    b. Diagrama de Arquitectura 
        Se utiliza: 
        - QuickBooks Online (QBO): fuente y API
        - Docker: para la infraestructura
        - RedDocker: para la red (comunicación entre contenedores)
        - MageAI: para la orquestración de pipelines y triggers 
        - PostgreSQL: como el warehouse
        - pgAdmin: como el UI para el manejo de PostgreSQL

        Diagrama: evidencias/diagrama_arquitectura.png

2. Levantar el proyecto 
    Considerando que el proyecto se corre con Docker, es necesario tener Docker para poder correrlo. Para obtener una fuente de datos se ha creado un app en el QBO Developer con un ambiente sandbox de una empresa para poder probar el uso de su API. A travez del OAuth Playground se ha obtenido todas las credenciales, al igual que el refresh token. 

    Al clonar este repositorio, dado a que las variables ya han sido configuradas en Mage Secrets, lo único que se necesita hacer es levantar los contenedores. Esto se lo realiza ubicandose en la carpeta donde ha sido replicado el repositorio, donde se encuentra el docker-compose.yaml, y en la terminal correr: docker compose up -d. Esto levanta todo lo necesario. Para la realización del proyecto se ha tenido que iniciar sesión en el pgAdmin y crear el source al igual que las tablas raw, e iniciar sesión en MageAI para poder trabajar los pipelines. 

    Accediendo al MageAI en http://localhost:6789, e iniciando sesión si es necesario (user: admin@admin.com) (pass: admin), se puede ver los pipelines y correr los triggers.

    Cada pipeline (qb_invoices_backfill, qb_items_backfill, qb_customers_backfill) tiene su propio trigger, el cual acepta la fecha_inicio, fecha_fin y page_size para la ejecución del pipeline de manera parametrizada. Al terminar el trigger estos se detienen automáticamente. Si lo desea puede revisar el funcionamiento en pgAdmin ejecutando simples consultas SQL, como: SELECT COUNT(*) FROM raw.qb_items. 

    Para parar los contenedores de manera segura se ejecuta en la terminal, en el directorio del repositorio: docker compose down. Estos al estar en un volumen aseguran la persistencia de sus datos.

3. Getión de Secretos 

    La gestión de información sensible se ha realizado dentro de Mage Secrets, generando un total de 10 variables encriptadas. 

    - QBO_CLIENT_ID: id de cliente registrado en Intuit Developer, utilizado para la conexión con el API.
    - QBO_CLIENT_SECRET: secret asociado al cliente.
    - QBO_REFRESH_TOKEN: token utilizado para obtener el acess token para poder realizar las llamadas. Si causa problemas los triggers, reemplazar este token y volver a generarlo.
    - QBO_REALM_ID: identificador de la compañia.
    - QBO_ENVIRONMENT: ambiente del app (sandbox)

    - POSTGRES_HOST: host de la base de datos
    - POSTGRES_PORT: puerto de conexión a la base de datos
    - POSTGRES_DB: nombre de la base de datos
    - POSTGRES_USER: usuario para ingresar a la base de datos
    - POSTGRES_PASSWORD: contrasena para la base de datos

4. Pipelines 
    Todos los pipelines siguen una estructura parecida, donde se garantiza todo lo especificado. Los tres son divididos en dos bloques: un data exporter y un data loader. Los pipelines son: qb_invoices_backfill, qb_items_backfill y qb_customers_backfill

    El data loader inicia con la inicialización de las variables dentro de Mage Secrets y la obtención del access token a partir del refresh token. Con este token se contruye el llamado al API posteriormente. Al ser parametrizado, el pipeline recibe las fechas y hace un chunking diario para procesar los datos. Estos parámetros son fecha_inicio, fecha_fin, y page_size que están en los trigger para correr los pipelines.Además por cada chunk o día, se realiza paginación, lo cual significa que mientras llega la información se va procesando por páginas de un tamaño definido, hasta que se agote la ingesta de ese chunk. En todo este proceso se realizan logs para detallar como va la ingesta, en que chunk/ventana se encuentra, y la paginación. De esta manera si es necesario reanudar una corrida, se sabe desde donde realizarla mediante los logs. Todos los elementos son cargados posteriormente en una lista para el siguiente bloque. 

    El data exporter en cambio, se encarga de cargar los jsons (paginados) a PostgreSQL. Esto lo hace al tomar la lista e ir generando las filas necesarias para cargar en las tablas raw. Aquí se agregan los metadatos necesarios. Con psycopg2 se abre una conexión y se utiliza Upsert para garantizar la idempotencia de los pipelines. En la carpeta de evidencias se encuentran más detalles.

    Dentro del pipeline tienen manejo de rate limits y de errores al identificarlos y usando sleep. Además tienen un sleep entre chunks, para evitar sobre saturar el API con llamadas.

    Sobre el runbook, se puede mencionar que dado el diseño de los pipelines, los tres permiten ejecutar los triggers desde tramos no finalizados, reintentar tramos con problemas de días específicos con la ayuda de los logs detallados sobre la ejecución de la carga y de la extracción de los datos. Estos resultados han sido verificados, son idempotentes, y cumplen con las exigencias del proyecto, evidencia en la carpeta de evidencias.

    5. Trigger
        Al igual que los pipelines los trigger siguen la misma estructura los tres. En este caso son: customers_backfill, items_backfill, e invoices_backfill. Los tres han sido configurados para correr una sola vez, con fecha 1 de febrero a las 14:00 UTC, es decir las 09:00 (America/Guayaquil). Dentro del trigger se han ingresado las variables de parametrización, con la fecha inicio siendo 2026-01-01 a las 0:00 UTC y la fecha fin siendo 2026-01-31 a las 23:59 UTC, siguiendo el formato de fecha y hora correcto. El page_size es de 10 para verificar la paginación. Para el manejo de la deshabilitación del trigger se garantiza que corra una vez, por lo que la evidencia muestra que está inactivo una vez de que termina la ejecución. Al tener un success, se deshabilita. Esto ha sido visto en los logs. Evidencia en la carpeta.

        El correcto uso de estos triggers permiten el uso del runbook descrito para volver a correr por tramos.

    6. Esquema raw
        Dentro de pgAdmin se han creado las 3 tablas raw para guardar los payload que entran desde Mage. Estas tablas han sido configuradas con una clave primaria y campos no nulos para dar seguridad. Los campos permiten ingresar los metadatos de cada payload en su formato indicado y al hacer upsert, se garantiza y se ha probado la idempotencia.

    7. Validaciones/volumetría
        Las validaciones de volumetría se realizan mediante consultas SQL en PostgreSQL y revisando los logs de ejecución en Mage. Se ejecutan conteos totales y se comparan con los registros leídos reportados en los logs del pipeline. La integridad se valida verificando unicidad de claves primarias, y la idempotencia se confirma al reejecutar un tramo sin que el conteo cambie. 
        
        Si los resultados son iguales se confirma esto. Evidencia en la carpeta.

    8. Troubleshooting
    



