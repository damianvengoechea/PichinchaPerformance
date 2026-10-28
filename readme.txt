=====================================================================
 EJERCICIO 1 - PRUEBA DE CARGA DEL SERVICIO DE LOGIN (JMETER)
=====================================================================

1. OBJETIVO
-----------
Ejecutar una prueba de carga sobre el servicio de login de FakeStoreAPI
con credenciales parametrizadas desde un archivo CSV. Criterios:
  - Alcanzar al menos 20 TPS (transacciones por segundo).
  - Tiempo de respuesta máximo permitido: 1,5 segundos.
  - Tasa de error menor al 3 % del total de peticiones.

Servicio probado (equivalente al CURL entregado):
  curl -X POST https://fakestoreapi.com/auth/login \
       -H "Content-Type: application/json" \
       -d '{"username":"johnd","password":"m38rmF$"}'


2. TECNOLOGÍAS Y VERSIONES
--------------------------
  - Apache JMeter 5.6.3
      https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-5.6.3.zip
  - Java 21 LTS (OpenJDK Temurin 21.0.12.1)
      https://adoptium.net/temurin/releases/?version=21
      IMPORTANTE: JMeter 5.6.3 necesita Java 8 a 25. Con Java 26 o
      superior la interfaz gráfica falla con
      "ClassNotFoundException: java.applet.Applet", porque esa API fue
      eliminada del JDK.
  - Git 2.x (para clonar el repositorio)
  - Sistema operativo usado: Windows 11 Pro (también funciona en Linux/macOS)
  - Conexión a Internet (el servicio es público)


3. ESTRUCTURA DEL REPOSITORIO
-----------------------------
  Pichincha.jmx                Plan de pruebas de JMeter
  Data/usuarios.csv            Datos de entrada (user,passwd)
  reports/
     results.jtl               Resultados crudos de la ejecución incluida
     html/index.html           Dashboard HTML generado por JMeter
     InformeResultados.doc     Informe con gráficas, hallazgos y recomendaciones
  jmeter.log                   Log de la ejecución incluida
  readme.txt                   Este archivo
  conclusiones.txt             Hallazgos y conclusiones


4. INSTALACIÓN
--------------
  Paso 1. Instalar Java 21 y comprobarlo:
             java -version
  Paso 2. Descargar apache-jmeter-5.6.3.zip y descomprimirlo, por ejemplo
          en C:\tools\apache-jmeter-5.6.3  (en adelante <JMETER>).
  Paso 3. (Solo si hay varias versiones de Java instaladas) crear el archivo
          <JMETER>\bin\setenv.bat para que JMeter use Java 21:
             set JAVA_HOME=C:\ruta\a\jdk-21
             set PATH=%JAVA_HOME%\bin;%PATH%
  Paso 4. Comprobar JMeter:
             <JMETER>\bin\jmeter.bat --version


5. EJECUCIÓN PASO A PASO
------------------------
  Paso 1. Clonar el repositorio y entrar en la carpeta:
             git clone https://github.com/damianvengoechea/PichinchaPerformance.git
             cd PichinchaPerformance

  Paso 2. (Opcional) Abrir el plan en la interfaz gráfica para revisarlo
          o hacer una prueba de humo:
             <JMETER>\bin\jmeter.bat -t Pichincha.jmx
          Para la prueba de humo: en el Thread Group usar 1 hilo y Loop
          Count 5 (sin duración), habilitar "View Results Tree" y
          ejecutar con el botón Start. Deben salir 5 respuestas 201 con
          token. Luego restaurar la configuración del punto 6 y
          deshabilitar "View Results Tree" antes de guardar.

  Paso 3. Ejecutar la prueba de carga en modo no GUI, desde la carpeta
          del repositorio (la carpeta "reports" ya existe en el repo):

     Windows (PowerShell):
        & "<JMETER>\bin\jmeter.bat" -n -t Pichincha.jmx `
          -l reports\results.jtl -e -o reports\html -f

     Windows (CMD):
        <JMETER>\bin\jmeter.bat -n -t Pichincha.jmx -l reports\results.jtl -e -o reports\html -f

     Linux/macOS:
        <JMETER>/bin/jmeter.sh -n -t Pichincha.jmx \
          -l reports/results.jtl -e -o reports/html -f

     Parámetros:
        -n      modo no GUI (recomendado para pruebas de carga)
        -t      plan de pruebas
        -l      archivo de resultados
        -e -o   genera el dashboard HTML en la carpeta indicada
        -f      sobrescribe los resultados de una ejecución anterior

     Duración aproximada: 2 minutos. Durante la ejecución la consola
     muestra una línea "summary" con el throughput y los errores.

  Paso 4. Abrir reports\html\index.html en el navegador y revisar la
          tabla "Statistics":
             - Transactions/s            >= 20
             - 95th / 99th pct / Max     <= 1500 ms
             - Error %                   < 3 %


6. DISEÑO DEL PLAN DE PRUEBAS
-----------------------------
  Test Plan
  ├── HTTP Request Defaults      Protocolo https, servidor fakestoreapi.com
  ├── HTTP Header Manager        Content-Type: application/json
  ├── CSV Data Set Config        Data/usuarios.csv (ruta relativa al .jmx);
  │                              variables tomadas de la cabecera
  │                              (user, passwd); delimitador ",";
  │                              Recycle on EOF = True;
  │                              Stop thread on EOF = False;
  │                              compartido entre todos los hilos
  └── Thread Group               20 hilos, ramp-up 30 s, loop infinito,
      │                          duración 120 s, continuar ante errores
      ├── Constant Throughput Timer   1500 peticiones/min = 25 TPS
      │                               (all active threads, shared)
      ├── HTTP Request           POST /auth/login (keep-alive)
      │   Body Data: {"username":"${user}","password":"${passwd}"}
      │   ├── Response Assertion  código de respuesta coincide con 20[01]
      │   ├── JSON Assertion      existe $.token
      │   └── Duration Assertion  <= 1500 ms
      ├── View Results Tree      (deshabilitado; solo para depuración)
      ├── Summary Report
      └── Aggregate Report

  Notas de diseño:
  - El objetivo se fijó en 25 TPS para superar con margen los 20 TPS
    exigidos, incluso contando la rampa de subida en el promedio.
  - Con una latencia de ~0,4 s, 20 hilos pueden generar hasta ~50 TPS.
    El Constant Throughput Timer los regula a 25 TPS.
  - Una petición que falla cualquier aserción (código, token o duración)
    cuenta como error, así que el "Error %" del reporte resume las tres
    validaciones del ejercicio.


7. RESULTADO DE LA EJECUCIÓN INCLUIDA (2026-10-28 16:55 - 16:57 COT)
-------------------------------------------------------------------
  Peticiones: 2 838 | Throughput: 23,61 TPS | Error: 0,00 %
  Promedio: 390 ms | p95: 471 ms | p99: 560 ms | Máximo: 1 276 ms
  Veredicto: CUMPLE los tres criterios.
  Detalle en conclusiones.txt y reports/InformeResultados.doc.
