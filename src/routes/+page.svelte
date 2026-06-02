<script>
  import { goto } from '$app/navigation';
  import { supabase } from '$lib/supabaseClient'; 

  let vistaActual = 'login'; 
  let verPassword = false; 

  // Variables para Login
  let loginCedula = '';
  let loginPassword = '';

  // Variables para Registro
  let regNombres = '';
  let regApellidos = '';
  let regCedula = '';
  let regPassword = '';
  let regCorreo = '';

  // Variables para Olvido Contraseña
  let olvidoCorreo = '';
  let codigoVerificacion = '';
  let codigoGenerado = '';
  let nuevaPassword = '';
  let pasoOlvido = 1;

  function cambiarVista(nuevaVista) {
    loginCedula = ''; loginPassword = '';
    regNombres = ''; regApellidos = ''; regCedula = ''; regPassword = ''; regCorreo = '';
    olvidoCorreo = ''; codigoVerificacion = ''; nuevaPassword = ''; pasoOlvido = 1;
    verPassword = false;
    vistaActual = nuevaVista;
  }

// --- 1. LÓGICA DE LOGIN ---
  async function manejarLogin() {
    if (!loginCedula || !loginPassword) {
      alert("Por favor complete todos los campos");
      return;
    }

    try {
      // Pedimos correo y rol a la base de datos
      const { data: usuarioDB, error: errorDB } = await supabase
        .from('Usuarios')
        .select('correo, rol') 
        .eq('cedula', loginCedula.trim())
        .single();

      if (errorDB || !usuarioDB) {
        alert("Cédula no encontrada.");
        return;
      }

      // Revisamos si hay una contraseña recuperada localmente para este correo
      const clavesRecuperadas = JSON.parse(localStorage.getItem('clavesRecuperadas') || '{}');
      const passwordAUsar = clavesRecuperadas[usuarioDB.correo] || loginPassword;

      // Iniciamos sesión en Supabase Auth
      const { data, error } = await supabase.auth.signInWithPassword({
        email: usuarioDB.correo,
        password: passwordAUsar
      });

      if (error) {
        alert("Contraseña incorrecta o acceso denegado.");
        return;
      }

      // Guardamos la sesión
      localStorage.setItem('usuarioActual', JSON.stringify(data.user));

      const rol = usuarioDB.rol;
      if (rol === 'guardia') {
          goto('/home');
      } else if (rol === 'admin') {
          goto('/admin');
      } else {
          goto('/usuario'); 
      }

    } catch (err) {
      console.error(err);
      alert("Error de conexión con el servidor.");
    }
  }
  
  // --- 2. LÓGICA DE REGISTRO ---
  async function manejarRegistro() {
    if (!regNombres || !regApellidos || !regCedula || !regPassword || !regCorreo) {
        alert("Por favor, complete todos los campos obligatorios.");
        return;
    }

    const correo = regCorreo.toLowerCase().trim();
    const cedula = regCedula.trim();

    if (cedula.length < 10 || !/^\d+$/.test(cedula)) {
        alert("La cédula debe tener al menos 10 dígitos numéricos.");
        return;  
    }

    try {
        const { data: existente } = await supabase
            .from('Usuarios')
            .select('cedula, correo')
            .or(`cedula.eq.${cedula},correo.eq.${correo}`);

        if (existente && existente.length > 0) {
            alert("Error: La cédula o el correo ya están registrados.");
            return;
        }
    } catch (err) { console.error(err); }

    let rolAsignado = '';
    if (correo.endsWith('@live.uleam.edu.ec')) rolAsignado = 'alumno';
    else if (correo.endsWith('@uleam.edu.ec')) rolAsignado = 'maestro';
    else if (correo === '2pq2pugj10@gmail.com') rolAsignado = 'guardia';
    else if (correo === '2pq2pugj9@gmail.com') rolAsignado = 'admin';
    else {  
        alert("Error: Usa un correo institucional (@live.uleam.edu.ec o @uleam.edu.ec)");
        return;
    }

    if (regPassword.length < 6 || !/[A-Z]/.test(regPassword) || !/[0-9]/.test(regPassword)) {
        alert("La contraseña debe tener mín. 6 caracteres, una mayúscula y un número.");
        return;
    }

    const { error } = await supabase.auth.signUp({
      email: correo,
      password: regPassword,
      options: {
        data: { cedula, nombres: regNombres, apellidos: regApellidos, rol: rolAsignado },
        emailRedirectTo: `${window.location.origin}/home`
      },
      emailRedirectTo: 'https://apcw-4to-marcelo-bravo.vercel.app/'
      
    });

    if (error) alert("Error al registrar: " + error.message);
    else {
      alert("¡Revisa tu correo! Se ha enviado un enlace de confirmación.");
      cambiarVista('login');
    }
  }

  // --- 3. LÓGICA DE OLVIDO DE CONTRASEÑA ---
  async function enviarCodigo() {
    if (!olvidoCorreo) { alert("Ingrese su correo"); return; }
    
    // Verificar si el correo existe
    const { data, error } = await supabase.from('Usuarios').select('correo').eq('correo', olvidoCorreo.trim()).single();
    if (error || !data) { alert("Correo no registrado."); return; }

    codigoGenerado = Math.floor(100000 + Math.random() * 900000).toString();
    alert(`🔐 CÓDIGO DE RECUPERACIÓN: ${codigoGenerado}`);
    pasoOlvido = 2; 
  }

  async function verificarYCambiar() {
    if (codigoVerificacion !== codigoGenerado) { alert("Código incorrecto."); return; }
    if (nuevaPassword.length < 6) { alert("Mínimo 6 caracteres."); return; }

    // Guardamos la nueva clave localmente para simular el cambio sin afectar Auth
    const clavesRecuperadas = JSON.parse(localStorage.getItem('clavesRecuperadas') || '{}');
    clavesRecuperadas[olvidoCorreo] = nuevaPassword;
    localStorage.setItem('clavesRecuperadas', JSON.stringify(clavesRecuperadas));

    alert("¡Contraseña actualizada localmente! Ya puedes iniciar sesión.");
    cambiarVista('login');
  }

</script>

<div class="login-wrapper">
  <div class="top">
    <button on:click={() => cambiarVista('login')}>
      <img src="/Imagenes/LogoUleam.png" alt="Logo Uleam">
    </button>
  </div>
  
  <div class="fondo"></div>
  <div class="base-blanca"></div>

  {#if vistaActual === 'login'}
    <div class="card">
      <div class="titulo">Parking ULEAM</div>
      <div class="form-group">
        <label for="cedula">Cédula</label>
        <input id="cedula" type="text" bind:value={loginCedula} placeholder="Ingresa tu cédula">
      </div>
      <div class="form-group">
        <label for="pass">Contraseña</label>
        <div class="input-password-container">
          <input id="pass" type={verPassword ? "text" : "password"} bind:value={loginPassword} placeholder="Contraseña">
          <button type="button" class="toggle-password" on:click={() => verPassword = !verPassword}>
            {verPassword ? "😮" : "😴"}
          </button>
        </div>
      </div>
      <button class="boton1" on:click={manejarLogin}>Iniciar Sesión</button>
      <div class="links">
        <button class="link-btn" on:click={() => cambiarVista('olvido')}>Olvidé mi contraseña</button>
        <p class="centrado">¿No tienes cuenta? <button class="link-btn" on:click={() => cambiarVista('registro')}>Regístrate</button></p>
      </div>
    </div>
  {/if}

  {#if vistaActual === 'registro'}
    <div class="card">
      <div class="card-registro">
        <div class="titulo">Registro</div>
        <div class="grid-form">
          <div class="form-group"><label>Nombres</label><input type="text" bind:value={regNombres}></div>
          <div class="form-group"><label>Apellidos</label><input type="text" bind:value={regApellidos}></div>
          <div class="form-group"><label>Cédula</label><input type="text" bind:value={regCedula}></div>
          <div class="form-group"><label>Correo Institucional</label><input type="email" bind:value={regCorreo}></div>
          <div class="form-group">
            <label>Contraseña</label>
            <div class="input-password-container">
              <input type={verPassword ? "text" : "password"} bind:value={regPassword} placeholder="Mín. 6 caracteres">
              <button type="button" class="toggle-password" on:click={() => verPassword = !verPassword}>
                {verPassword ? "😮" : "😴"}
              </button>
            </div>
          </div>
        </div>
      </div>
      <button class="boton1" on:click={manejarRegistro}>Enviar Confirmación</button>
      <p class="centrado">¿Ya tienes cuenta? <button class="link-btn" on:click={() => cambiarVista('login')}>Inicia Sesión</button></p>
    </div>
  {/if}

  {#if vistaActual === 'olvido'}
    <div class="card">
      <div class="titulo">Recuperar Cuenta</div>
      {#if pasoOlvido === 1}
        <div class="form-group">
          <label>Correo Electrónico</label>
          <input type="email" bind:value={olvidoCorreo} placeholder="Correo institucional">
        </div>
        <button class="boton1" on:click={enviarCodigo}>Enviar Código</button>
      {:else}
        <p class="centrado">Código enviado a: <strong>{olvidoCorreo}</strong></p>
        <div class="form-group">
            <label>Código</label>
            <input type="text" bind:value={codigoVerificacion} placeholder="Ingrese el código">
        </div>
        <div class="form-group">
            <label>Nueva Contraseña</label>
            <input type="password" bind:value={nuevaPassword} placeholder="Nueva clave">
        </div>
        <button class="boton1" on:click={verificarYCambiar}>Cambiar Contraseña</button>
      {/if}
      <button class="link-btn cancelar" on:click={() => cambiarVista('login')}>Cancelar</button>
    </div>
  {/if}
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600;700;800&display=swap');

  /* Reset global para evitar desbordes */
  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    margin: 0;
    padding: 0;
    overflow-x: hidden;
    background-color: #ffffff;
    font-family: 'Poppins', sans-serif;
  }

  .login-wrapper {
    position: relative;
    width: 100%;
    /* Ajuste para móviles: permite scroll si la pantalla es muy pequeña */
    min-height: calc(100vh - 64px);
    margin-top: 64px;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 20px; /* Padding extra para que la card no toque los bordes */
  }

  .top {
    background: rgba(45, 45, 45, 0.95);
    backdrop-filter: blur(10px);
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 65px;
    display: flex;
    align-items: center;
    z-index: 100;
    padding-left: 20px;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  }

  .top button {
    background: none;
    border: none;
    height: 64px;
    max-width: 166px; /* Evita que el logo sea gigante */
    cursor: pointer;
    transition: transform 0.2s;
    box-shadow: none;
    padding: 10px 0;
  }
  .top button:hover { transform: scale(1.02); }
  .top img { height: 100%; width: auto; filter: drop-shadow(0 2px 4px rgba(45, 45, 45, 0.95)); }

  .fondo {
    position: absolute;
    top: 0; 
    left: 0; 
    width: 100%; 
    height: 55vh;
    background-image: url("/Imagenes/FondoUleam.jpg");
    background-size: cover;
    background-position: center;
    filter: blur(10px) brightness(0.9);
    z-index: 0;
  }

  .base-blanca {
    position: absolute;
    top: 50vh; left: 0; width: 100%; height: 50vh;
    background: #ffffff;
    box-shadow: 0 -15px 35px rgba(0, 0, 0, 0.1);
    z-index: 1;
  }

  /* Tarjeta principal con ancho controlado */
  .card {
    position: relative;
    z-index: 10;
    background: #ffffff;
    padding: 45px;
    border-radius: 20px;
    box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
    width: 100%;
    max-width: 460px; /* Límite máximo para que no se deforme en PC */
    text-align: center;
    border: 1px solid rgba(0, 0, 0, 0.05);
    margin: 0 auto;
  }

  .titulo {
    font-size: 2rem;
    font-weight: 800;
    margin-bottom: 25px;
    color: #1a202c;
    letter-spacing: -0.5px;
    line-height: 1.2;
  }

  .form-group { margin-bottom: 15px; text-align: left; }
  
  .grid-form {
    max-height: 55vh; /* Ajustado para pantallas pequeñas */
    overflow-y: auto;
    padding-right: 5px; 
  }

  .input-password-container {
    position: relative;
    display: flex;
    align-items: center;
  }

  .toggle-password {
    position: absolute;
    right: 12px;
    background: none;
    border: none;
    cursor: pointer;
    font-size: 1.2rem;
    padding: 0;
    box-shadow: none; 
    height: 100%;
    display: flex;
    align-items: center;
  }

  .toggle-password:hover {
    transform: scale(1.1);
  }

  .input-password-container input {
    padding-right: 45px;
  }

  label {
    display: block;
    font-size: 0.85rem;
    margin-bottom: 6px;
    color: #4a5568;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.5px;
  }

  input {
    width: 100%;
    padding: 14px 16px;
    background-color: #f7fafc;
    border: 2px solid #edf2f7;
    border-radius: 10px;
    box-sizing: border-box;
    font-size: 1rem;
    color: #2d3748;
    transition: all 0.3s ease;
  }

  input:focus {
    outline: none;
    background-color: #ffffff;
    border-color: #007bff;
    box-shadow: 0 0 0 4px rgba(0, 123, 255, 0.15);
  }

  .boton1 {
    width: 100%;
    padding: 16px;
    background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
    color: white;
    border: none;
    border-radius: 12px;
    cursor: pointer;
    font-size: 1.1rem;
    font-weight: 700;
    box-shadow: 0 10px 15px -3px rgba(0, 123, 255, 0.3);
    transition: all 0.3s ease;
    margin-top: 15px;
    margin-bottom: 15px;
  }

  .boton1:hover {
    transform: translateY(-2px);
    box-shadow: 0 15px 20px -5px rgba(0, 123, 255, 0.4);
    background: linear-gradient(135deg, #0088ff 0%, #0066cc 100%);
  }

  .links {
    display: flex;
    flex-direction: column;
    gap: 10px;
    text-align: center;
  }

  .link-btn {
    background: none;
    border: none;
    color: #007bff;
    text-decoration: none;
    font-size: 0.95rem;
    font-weight: 600;
    transition: color 0.2s;
    box-shadow: none;
    padding: 0;
    cursor: pointer; 
    width: fit-content;
    margin: 0 auto;   
  }

  .link-btn:hover {
    color: #0056b3;
    text-decoration: underline;
  }

  .centrado { margin-top: 10px; font-size: 0.9rem; color: #718096; }
  .cancelar { color: #e53e3e; margin-top: 12px; display: block; width: 100%; text-align: center; }
  .cancelar:hover { color: #c53030; }

  /* --- MEDIA QUERIES PARA CELULARES --- */
  @media (max-width: 600px) {
    .card {
      padding: 25px 20px; /* Menos padding en celular para ganar espacio */
      width: 95%; /* Ocupa casi todo el ancho */
    }

    .titulo {
      font-size: 1.7rem; /* Título un poco más pequeño */
      margin-bottom: 20px;
    }

    /* Ajuste para que iOS no haga zoom al escribir */
    input {
      font-size: 16px; 
      padding: 12px 14px;
    }

    .boton1 {
      padding: 14px;
      font-size: 1rem;
    }
  }
</style>