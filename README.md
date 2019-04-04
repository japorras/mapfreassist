# Aplicacion AssistEvent
## Descripcion
Aplicación SpringBoot que despliega bajo un contenedor web embebido para la publicación de 2 operaciones:
- /assit/event?id={id} : Operación GET que recibe como parámetro un identificador de evento para su búsqueda en base de datos, devolviendo como respuesta OK/KO en función de si es recogido el evento de bbdd

- /send/event?id={id} : Operación GET que recibe como parámetro un identificador de evento para una vez recogido de base de datos el evento asociado al identificador, este sea enviado al servicio correspondiente del APIGateway de Mapfre para delegar el evento sobre AWS.

- /async/notification/event : Operación POST que recibe en el cuerpo de la petición, la respuesta devuelta por la invocación del servicio de integración de cierto proveedor. Esta operación será utilizada para las comunicaciones asincronas, servício que deberá ser invocado por los servicios en la nube.  

## Documentación Orientada al API
Se ha echo uso de Swagger y Springfox como librerías para exponer la documentación orientada al API. La URL de acceso sobre la interfaz web de swagger es la siguiente:
```
http://<hostname>:<port>/swagger-ui.html
```
## Despliegue de la Aplicación SpringBoot
La aplicación SpringBoot es desplegada sobre un contenedor web embebido, a partir del cual expondrá las dos operaciones indicadas anteriormente. Para poder lanzar la ejecución de la aplicación, será necesario lanzar el siguiente comando:
```
java -jar -Dspring.profiles.active=dev integration-<version>.jar
```
La aplicación actualmente cuenta con un único profile (dev), que será necesario indicar en fase de arranca de la aplicación.
El artefacto de despliegue correspondiente a la aplicación, es totalmente agnóstico, no conteniendo ningún tipo de configuración dependiente del entorno de manera embebida. Luego, el artefacto de despliegue requiere de un directorio cuyo nombre debe ser "config" que se encuentre al mismo nivel donde será ejecutado el artefacto .jar. Este directorio, contendrá los ficheros de configuración dependientes del entorno, para cada uno de los perfiles manejados, es decir, application-<environment>.properties

## Configuración y Base de datos
- Base de datos H2
Actualmente la base de datos de la que se esta haciendo uso en la aplicación, es una solución de base de datos en memoría H2.
En tiempo de arranque de la aplicación es generado el script ddl y jutno con el schema y la tabla asociada a la definición de la Entity  'com.mapfre.spain.assist.entity.AssistEvent'.
Generada la tabla, será cargado el script dml con nombre data.sql ubicado en el directorio del classpath 'src/main/resources'. Este script insertará un registro con código identificador de mensaje 1 sobre la tabla creada.
Contenido del script:
```
INSERT INTO tbl_integration (message_Id, date_creation, types, payload, endpoint, language, type_authorization, endpoint_authorization, user_pass, syncronization) VALUES ('1', '2019-03-19', 'SOAP', '{"prueba":"test1"}', 'https://domain:432/services/operation', 'xml', 'Basic', 'https://hostdomain:443/service2/auth','asdfa3w5t34vcgbadatg4tjh', 'true');
```
- Configuración
La base de datos H2, es configurada para que pueda ser accesible vía consola web bajo el path '/console', desde este path podra ser accesible mediante una interfaz web la base de datos para contemplar la modificación, inserción o recuperación de los datos sobre la tabla creada.
```
http://<hostname>:<port>/console
```
La configuración asociada para este comportamiento de la base de datos se encuentra en el fichero application.properties.
```
spring.datasource.url=jdbc:h2:mem:mapfre-app;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.platform=h2
spring.datasource.username = sa
spring.datasource.password =
spring.datasource.driverClassName = org.h2.Driver
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect

spring.h2.console.enabled=true
spring.h2.console.path=/console
spring.h2.console.settings.trace=false
spring.h2.console.settings.web-allow-others=true
```
## Servicios Moqueados y Pruebas Unitarias
La aplicaciónn posee varios test unitarios, entre los que se encuentran: 
- La prueba unitaria para lanzar al ejecución del servicio '/send/event' y moquear la comunicación con el servicio del ApiGateway. (SendEventTest)

- Prueba Unitaria para simular la recepción de la llamada al servicio de notificación de recepción de evento. (notificationEventTest)

- Prueba Unitaria para validar la recogida del evento de base de datos. 

El modo de lanzar la ejecución de las pruebas unitarias sería mediante la herramienta maven, con el siguiente comando desde el directorio raiz del proyecto del código fuente: 
```
mvn clean test
```

## Configuración y sistema de trazas
- La aplicación contiene una configuración asociada para el sistema de trazas utilizado, haciendo uso de logback como solución para este cometido.
- Las trazas a nivel de consola y las trazas a nivel de info, debug y error de la aplicación serán almacenadas sobre un fichero log llamado 'assistevent.log', el cual será alojado sobre el directorio raiz en el que es ejecutada la aplicación, dentro del directorio 'logs'.
- Se ha configurado un sistema de rotado de logs cuando sea alcanzado el tamaño de 100 MB por parte del fichero de log, manteniendo un máximo de 10 ficheros historicos.
```
<appender name="SAVE-TO-FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">

		  <file>${LOG_PATH}/assistevent.log</file>

		  <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
		    <Pattern>
		      %d{dd-MM-yyyy HH:mm:ss.SSS} [%thread] %-5level %logger{36}.%M - %msg%n
		    </Pattern>
		  </encoder>

		  <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
		    <fileNamePattern>
		      ${LOG_PATH}/archived/log_%d{dd-MM-yyyy}.log
		    </fileNamePattern>
		    <maxHistory>10</maxHistory>
		    <totalSizeCap>100MB</totalSizeCap>
		  </rollingPolicy>

  </appender>
```  

El fichero de configuración de trazas se encuentra en el directorio 'src/main/resources' con nombre logback-spring.xml.

## Configuración Acceso endpoint Integracion AWS
La configuración del nombre de dns del endpoint al servicio de integración en AWS, se encuentra en el fichero de configuración dependiente del entorno application<environment>.properties en las siguientes variables
```
app.hostnameAPI=<hostname>
app.portAPI=443
```
EL hostname de manera momentanea debera ir cambiando por cada redespliegue sobre AWS. 
