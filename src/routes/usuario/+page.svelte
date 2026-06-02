<script>
  import { onMount } from 'svelte';
  import { goto } from '$app/navigation';
  import { supabase } from '$lib/supabaseClient';

  let usuario = null;
  let cargando = true;
  let tieneVehiculo = false; // Estado clave para bloquear/desbloquear paneles
  let vehiculoActual = null;

  // --- DATOS FORMULARIO IZQUIERDO (Registro Nuevo) ---
  let regPlaca = '';
  let regMarca = '';
  let regModelo = '';
  let regColor = '';

  // --- DATOS FORMULARIO DERECHO (Solicitud Cambio) ---
  let solPlaca = '';
  let solMarca = '';
  let solModelo = '';
  let solColor = '';
  let motivoCambio = '';

  onMount(async () => {
    // Verificar sesión
    const sesion = localStorage.getItem('usuarioActual');
    if (!sesion) return goto('/', { replaceState: true });
    usuario = JSON.parse(sesion);

    // Consultar si ya tiene vehículo registrado
    const { data, error } = await supabase
      .from('Vehiculos')
      .select('*')
      .eq('propietario_id', usuario.id)
      .maybeSingle(); 

    if (data) {
      tieneVehiculo = true;
      vehiculoActual = data;
      regPlaca = data.placa;
      regMarca = data.marca;
      regModelo = data.modelo;
      regColor = data.color;
    }

    cargando = false;
  });

  // --- FUNCIÓN PANEL IZQUIERDO: REGISTRAR VEHÍCULO ---
  async function registrarVehiculo() {
    if (!regPlaca || !regMarca || !regModelo || !regColor) {
      return alert("Por favor completa todos los datos del vehículo.");
    }

    const { error } = await supabase.from('Vehiculos').insert([{
      propietario_id: usuario.id,
      placa: regPlaca.toUpperCase(),
      marca: regMarca,
      modelo: regModelo,
      color: regColor,
      estado: 'registrado'
    }]);

    if (error) {
      alert("Error al registrar: " + error.message);
    } else {
      alert("¡Vehículo registrado con éxito!");
      window.location.reload(); // Recargamos para actualizar el estado de los paneles
    }
  }

  // --- FUNCIÓN PANEL DERECHO: SOLICITAR CAMBIO ---
  async function enviarSolicitud() {
    if (!motivoCambio) return alert("Por favor indica un motivo o qué deseas cambiar.");

    // Preparamos el JSON con los datos nuevos que el usuario escribió
    const datosNuevos = {
      placa: solPlaca || vehiculoActual.placa,
      marca: solMarca || vehiculoActual.marca,
      modelo: solModelo || vehiculoActual.modelo,
      color: solColor || vehiculoActual.color,
      motivo: motivoCambio
    };

    const { error } = await supabase.from('Solicitudes').insert([{
      usuario_id: usuario.id,
      vehiculo_id: vehiculoActual.id,
      tipo_solicitud: 'actualizacion',
      estado: 'pendiente',
      datos_nuevos: datosNuevos
    }]);

    if (error) {
      alert("Error al enviar solicitud: " + error.message);
    } else {
      alert("Solicitud enviada al Administrador. Te notificaremos cuando se apruebe.");
      // Limpiar formulario derecho
      solPlaca = ''; solMarca = ''; solModelo = ''; solColor = ''; motivoCambio = '';
    }
  }

  function cerrarSesion() {
    localStorage.removeItem('usuarioActual');
    goto('/', { replaceState: true });
  }
</script>

<div class="main-container">
  <div class="top">
    <div class="logo-area">
      <img src="/Imagenes/LogoUleam.png" alt="Logo Uleam">
      <span>Panel de Usuario ({usuario?.user_metadata?.rol || 'Estudiante'})</span>
    </div>
    <button class="btn-logout" on:click={cerrarSesion}>
      <i class="fas fa-sign-out-alt"></i> Cerrar Sesión
    </button>
  </div>

  <div class="fondo"></div>
  
  <div class="content-wrapper">
    {#if cargando}
      <div class="loading">Cargando datos...</div>
    {:else}
      <div class="grid-paneles">
        
        <div class="card panel {tieneVehiculo ? 'bloqueado' : ''}">
          <div class="header-panel">
            <i class="fas fa-car"></i>
            <h3>{tieneVehiculo ? 'Vehículo Registrado' : 'Registrar Vehículo'}</h3>
          </div>
          
          <div class="form-body">
            <label>Placa</label>
            <input type="text" bind:value={regPlaca} disabled={tieneVehiculo} placeholder="Ej: MBC-1234">
            
            <label>Marca</label>
            <input type="text" bind:value={regMarca} disabled={tieneVehiculo} placeholder="Ej: Chevrolet">
            
            <label>Modelo</label>
            <input type="text" bind:value={regModelo} disabled={tieneVehiculo} placeholder="Ej: Aveo">
            
            <label>Color</label>
            <input type="text" bind:value={regColor} disabled={tieneVehiculo} placeholder="Ej: Rojo">

            {#if !tieneVehiculo}
              <button class="boton1" on:click={registrarVehiculo}>Confirmar Registro</button>
            {:else}
              <div class="status-badge">
                <i class="fas fa-check-circle"></i> Vehículo Activo
              </div>
            {/if}
          </div>
        </div>

        <div class="card panel {!tieneVehiculo ? 'desactivado' : ''}">
          <div class="header-panel">
            <i class="fas fa-edit"></i>
            <h3>Solicitud de Actualización</h3>
          </div>

          <div class="form-body">
            {#if !tieneVehiculo}
              <div class="mensaje-espera">
                <p>Registra un vehículo primero para habilitar este panel.</p>
              </div>
            {:else}
              <p class="instruccion">Llena solo los campos que deseas cambiar:</p>

              <div class="fila-doble">
                <div>
                  <label>Nueva Placa</label>
                  <input type="text" bind:value={solPlaca} placeholder={vehiculoActual.placa}>
                </div>
                <div>
                  <label>Nuevo Color</label>
                  <input type="text" bind:value={solColor} placeholder={vehiculoActual.color}>
                </div>
              </div>

              <div class="fila-doble">
                <div>
                  <label>Nueva Marca</label>
                  <input type="text" bind:value={solMarca} placeholder={vehiculoActual.marca}>
                </div>
                <div>
                  <label>Nuevo Modelo</label>
                  <input type="text" bind:value={solModelo} placeholder={vehiculoActual.modelo}>
                </div>
              </div>

              <label>Motivo del cambio (Obligatorio)</label>
              <textarea rows="2" bind:value={motivoCambio} placeholder="Ej: Vendí el auto anterior..."></textarea>

              <button class="boton1 warning" on:click={enviarSolicitud}>Enviar Solicitud</button>
            {/if}
          </div>
        </div>

      </div>
    {/if}
  </div>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600;700&display=swap');

  :global(body) { margin: 0; font-family: 'Poppins', sans-serif; background: #f0f2f5; }

  .main-container { min-height: 100vh; display: flex; flex-direction: column; }
  
  .top {
    background: rgba(45, 45, 45, 0.95);
    backdrop-filter: blur(10px);
    height: 64px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 0 20px;
    color: white;
    z-index: 100;
    box-shadow: 0 4px 6px rgba(0,0,0,0.1);
  }
  .logo-area { display: flex; align-items: center; gap: 15px; font-weight: 600; }
  .logo-area img { height: 40px; }
  .btn-logout { background: #e74c3c; color: white; border: none; padding: 8px 15px; border-radius: 6px; cursor: pointer; font-weight: 600; transition: 0.3s; }
  .btn-logout:hover { background: #c0392b; }

  .fondo {
    position: fixed; top: -200px; left: -20px; width: 102%; height: 150%;
    background-image: url("/Imagenes/FondoUleam.jpg");
    background-size: cover; background-position: center;
    filter: blur(8px) brightness(0.6);
    z-index: -1;
  }

  .content-wrapper {
    flex: 1;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 20px;
  }

  .grid-paneles {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 100px;
    width: 100%;
    max-width: 900px;
  }

  .card {
    background: rgba(255, 255, 255, 0.95);
    border-radius: 20px;
    padding: 30px;
    box-shadow: 0 15px 35px rgba(0,0,0,0.2);
    transition: transform 0.3s ease, opacity 0.3s ease;
    display: flex;
    flex-direction: column;
  }

  .header-panel { display: flex; align-items: center; gap: 10px; margin-bottom: 20px; border-bottom: 2px solid #eee; padding-bottom: 10px; }
  .header-panel i { font-size: 1.5rem; color: #007bff; }
  .header-panel h3 { margin: 0; color: #333; font-weight: 700; }

  .bloqueo input {
    background-color: #e9ecef;
    color: #6c757d;
    border-color: #ced4da;
    cursor: not-allowed;
  }
  
  .status-badge {
    margin-top: 20px;
    background: #d4edda; color: #155724;
    padding: 15px; border-radius: 10px;
    text-align: center; font-weight: bold;
    display: flex; align-items: center; justify-content: center; gap: 10px;
  }

  .desactivado {
    opacity: 0.6;
    pointer-events: none;
    filter: grayscale(0.8);
  }

  .mensaje-espera {
    height: 100%; display: flex; align-items: center; justify-content: center;
    text-align: center; color: #777; font-style: italic;
  }

  /* --- FORMULARIOS --- */
  .form-body { display: flex; flex-direction: column; flex: 1; }
  label { font-size: 0.85rem; color: #555; font-weight: 600; margin-bottom: 5px; margin-top: 10px; }
  input, textarea {
    width: 100%; padding: 12px;
    border: 2px solid #eee; border-radius: 8px;
    background: #f9f9f9; font-family: inherit;
    box-sizing: border-box; 
  }
  input:focus, textarea:focus { outline: none; border-color: #007bff; background: white; }

  .fila-doble { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; }

  .boton1 {
    width: 100%; padding: 14px;
    background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
    color: white; border: none; border-radius: 10px;
    margin-top: 50px; font-weight: 700; cursor: pointer;
    box-shadow: 0 5px 15px rgba(0, 123, 255, 0.3);
    transition: 0.2s;
  }
  .boton1:hover { transform: translateY(-2px); }
  
  .warning {
    background: linear-gradient(135deg, #f39c12 0%, #d35400 100%);
    box-shadow: 0 5px 15px rgba(243, 156, 18, 0.3);
  }

  .instruccion { font-size: 0.9rem; color: #666; margin-bottom: 15px; }
  .loading { text-align: center; font-size: 1.2rem; color: white; font-weight: bold; margin-top: 50px; }

  /* Responsive para móviles */
  @media (max-width: 768px) {
    .grid-paneles { grid-template-columns: 1fr; }
  }
</style>