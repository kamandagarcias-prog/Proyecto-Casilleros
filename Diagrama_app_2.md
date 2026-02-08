graph TD
    %% Definición de Estilos (Clases)
    classDef database fill:#f96,stroke:#333,stroke-width:2px;
    classDef logic fill:#bbf,stroke:#333,stroke-width:1px;
    classDef activity fill:#dfd,stroke:#333,stroke-width:2px;
    classDef error fill:#fbb,stroke:#333,stroke-width:1px;
    classDef sensor fill:#ff9,stroke:#333,stroke-width:2px;
    classDef timer fill:#fdf,stroke:#333,stroke-width:2px;

    %% --- MÓDULO DE ACCESO (LOGIN & REGISTRO) ---
    subgraph RegistroActivity_Logic [Lógica de Registro]
        R1(Input: Datos Usuario) --> R2{Validar Formato}
        R2 -- Pass != Confirm --> R2_Err[Toast: No coinciden]
        R2 -- Pass < 6 chars --> R2_Err2[Toast: Muy corta]
        
        R2 -- OK --> R3[(DB: /matriculas_autorizadas/matricula)]
        
        R3 --> R4{¿Habilitado && !Registrado?}
        R4 -- No --> R4_Err[Toast: No autorizada / Ya registrada]
        
        R4 -- Sí --> R5[mAuth.createUserWithEmailAndPassword]
        R5 -- Éxito --> R6[(DB: /usuarios/matricula)]
        R5 -- Error --> R5_Err[Toast: Correo en uso / Error Auth]
        
        R6 --> R7[Map: nombre, correo, matricula]
        R7 --> R8[setValue: userData]
        
        R8 -- Éxito --> R9[(DB: Actualizar registrado = true)]
        R9 --> R10[Handler Delay 1.5s]
        R10 --> R11[Intent -> LoginActivity]
    end

    subgraph LoginActivity_Logic [Lógica de Login]
        L1(Input: Correo/Pass) --> L2{Campos Vacíos?}
        L2 -- Sí --> L2_Err[Toast: Llenar campos]
        L2 -- No --> L3[mAuth.signInWithEmailAndPassword]
        
        L3 -- Éxito --> L4[(DB: /usuarios)]
        L3 -- Error --> L3_Err[Toast: Auth Fallida]
        
        L4 --> L5{¿Existe Correo?}
        L5 -- No --> L5_Err[mAuth.signOut + Toast: Perfil no encontrado]
        L5 -- Sí --> L6[Extraer Matricula Key]
        
        L6 --> L7[(DB: /matriculas_autorizadas/matricula)]
        L7 --> L8{¿habilitado == true?}
        L8 -- No --> L8_Err[mAuth.signOut + Toast: Deshabilitado]
        L8 -- Sí --> L9[Intent -> CasillerosActivity]
        L9 --> L9_Data[PutExtra: matricula_usuario]
    end

    %% --- MÓDULO DE GESTIÓN (LISTA DE CASILLEROS) ---
    subgraph CasillerosActivity_Logic [Gestión de Lista y Tiempo Real]
        C1[addValueEventListener] --> C2[(DB: /Casilleros)]
        C2 --> C3{¿Ocupado && Tiempo Vencido?}
        C3 -- Sí --> C3_Action[liberarCasillero: RESET DB]
        C3 -- No --> C4[Actualizar Estado Visual]
        
        C4 --> C4_Logic{¿Es mi Casillero?}
        C4_Logic -- Sí --> C4_UI[Color: Azul / Amarillo]
        C4_Logic -- No --> C4_UI2[Color: Rojo]

        C5[Click Botón Casillero] --> C6{BiometricManager}
        C6 -- Success --> C7[abrirCasillero Method]
        C6 -- Fail/No Hardware --> C6_Err[Toast: Error Biométrico]
        
        C7 --> C8{¿Está Libre?}
        C8 -- Sí --> C9[ocuparCasillero: Set HoraInicio + 3 min]
        C8 -- No && Es Mío --> C10[Intent -> CasilleroActivity]
    end

    %% --- MÓDULO DE CONTROL (DETALLE DEL CASILLERO) ---
    subgraph CasilleroActivity_Logic [Control Individual y Automatización]
        CA1[addValueEventListener] --> CA2{¿Sigue siendo mío?}
        CA2 -- No --> CA2_Err[finish Activity]
        CA2 -- Sí --> CA3[Cálculo Tiempo Restante]
        
        CA3 --> CA4[iniciarCronometro: CountDownTimer]
        
        CA4 --> CA5{onTick: T < 1 min?}
        CA5 -- Sí (una vez) --> CA6[Enviar Notificación Push]
        CA5 -- Sí --> CA7[Update DB: estado = POR_VENCER]
        
        CA4 --> CA8{onFinish: Tiempo Agotado}
        CA8 --> CA9[actualizarEstadoPuerta: TRUE]
        CA9 --> CA10[Handler: postDelayed 5s]
        CA10 --> CA11[actualizarEstadoPuerta: FALSE]
        CA11 --> CA12[liberarCasillero: RESET DB]
    end

    %% --- BOTONES Y ACCIONES MANUALES ---
    btn_A[Botón Abrir/Cerrar Manual] --> CA_manual[Update DB: abierto = true/false]

    %% --- CONEXIONES GLOBALES ---
    LoginActivity_Logic -- Botón Registro --> RegistroActivity_Logic
    RegistroActivity_Logic -- Registro Exitoso --> LoginActivity_Logic
    L9 -- Start --> CasillerosActivity_Logic
    C10 -- Start --> CasilleroActivity_Logic
    C9 -- Tras Ocupar --> C10
    CA12 -- Al Finalizar --> CasillerosActivity_Logic

    %% Aplicación de Clases
    class L4,L7,R3,R6,R9,C2 database;
    class L2,L5,L8,R2,R4,C3,C8,CA2,CA5 logic;
    class L9,R11,C10,C_Act,CA_Act activity;
    class L2_Err,L3_Err,L5_Err,L8_Err,R2_Err,R2_Err2,R4_Err,R5_Err,C6_Err,CA2_Err error;
    class C6,CA9,CA11 sensor;
    class CA4,CA8,CA10 timer;
