graph TD
    %% Definición de estilos para simular el diseño
    classDef actor fill:#f3e8ff,stroke:#9333ea,stroke-width:1px,color:#000
    classDef sistema fill:#c084fc,stroke:#7e22ce,stroke-width:2px,color:#000
    classDef externo fill:#f3e8ff,stroke:#9333ea,stroke-width:1px,color:#000
    classDef textoRelacion fill:#fff,stroke:#ccc,stroke-width:1px,color:#000

    %% Actores
    Empleado["Empleado de la<br>organización"]:::actor
    Jefe["Jefe de personal"]:::actor
    Contador["Contador"]:::actor

    %% Sistema Central
    NominaU["NóminaU<br>Sistema de nómina y<br>asistencia"]:::sistema

    %% Sistemas Externos
    Correo["Servicio de correo<br>(sistema externo)"]:::externo
    BD["Sistema de base<br>de datos"]:::externo
    Banco["Sistema financiero /<br>banco (sistema externo)"]:::externo

    %% Relaciones desde Actores hacia el Sistema
    Empleado -- "consulta horas, entrada/<br>salida y desprendibles" --> NominaU
    Jefe -- "registra asistencia diaria<br>y gestiona personal" --> NominaU
    Contador -- "ejecuta liquidación de<br>nómina y consulta histórico" --> NominaU

    %% Relaciones desde el Sistema hacia Externos
    NominaU -- "envía desprendibles y<br>notificaciones" --> Correo
    NominaU -- "valida asistencia diaria<br>o datos de contrato" --> BD
    NominaU -- "envía reporte de<br>pagos o dispersión" --> Banco
