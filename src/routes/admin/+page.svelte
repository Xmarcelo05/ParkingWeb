<script>
  import { onMount } from 'svelte';
  import { supabase } from '$lib/supabaseClient';
  import { goto } from '$app/navigation';

  let pestanaActual = 'inventario'; // 'inventario' | 'solicitudes'
  let vehiculos = [];
  let solicitudes = [];
  let cargando = false;
  let usuarioAdmin = null;

  onMount(async () => {
    await verificarAdmin();
    await cargarDatos();
  });

  async function verificarAdmin() {
    const { data: { session } } = await supabase.auth.getSession();
    if (!session) goto('/');
    
    // Verificar rol
    const { data: usuario } = await supabase
      .from('Usuarios')
      .select('rol, nombres')
      .eq('id', session.user.id)
      .single();

    if (usuario?.rol !== 'admin') goto('/home');
    usuarioAdmin = usuario;
  }

  async function cargarDatos() {
    cargando = true;

    // 1. Inventario
    const { data: datosVehiculos } = await supabase
      .from('Vehiculos')
      .select(`*, Usuarios ( nombres, apellidos, cedula )`)
      .order('created_at', { ascending: false });
    
    if (datosVehiculos) vehiculos = datosVehiculos;

    // 2. Solicitudes
    const { data: datosSolicitudes } = await supabase
      .from('Solicitudes')
      .select(`*, Usuarios ( nombres, apellidos, cedula ), Vehiculos ( placa )`)
      .eq('estado', 'pendiente');

    if (datosSolicitudes) solicitudes = datosSolicitudes;
    cargando = false;
  }

  async function procesarSolicitud(solicitud, accion) {
    if(!confirm(`¿Deseas ${accion.toUpperCase()} esta solicitud?`)) return;

    try {
      if (accion === 'aprobar') {
        const cambios = solicitud.datos_nuevos; 
        const { error: errUpdate } = await supabase.from('Vehiculos').update({
            marca: cambios.marca, modelo: cambios.modelo, color: cambios.color, placa: cambios.placa
        }).eq('id', solicitud.vehiculo_id);
        if (errUpdate) throw errUpdate;
      }

      const { error: errEstado } = await supabase.from('Solicitudes')
        .update({ estado: accion === 'aprobar' ? 'aprobada' : 'rechazada' })
        .eq('id', solicitud.id);

      if (errEstado) throw errEstado;

      alert(`Solicitud ${accion === 'aprobar' ? 'APROBADA' : 'RECHAZADA'} correctamente.`);
      cargarDatos();

    } catch (error) {
      alert("Error: " + error.message);
    }
  }

  // --- CORRECCIÓN AQUÍ: Renombrada a cerrarSesion ---
  async function cerrarSesion() {
    await supabase.auth.signOut(); // Cierra sesión en Supabase
    localStorage.removeItem('usuarioActual'); // Limpia caché local
    goto('/', { replaceState: true }); // Redirige al login
  }
</script>

<div class="top">
  <div class="logo-container">
    <img src="/Imagenes/LogoUleam.png" alt="Logo Uleam">
    {#if usuarioAdmin}
        <span class="admin-badge">Admin: {usuarioAdmin.nombres}</span>
    {/if}
  </div>
  <button class="btn-logout" on:click={cerrarSesion}>
      <i class="fas fa-sign-out-alt"></i> Cerrar Sesión
    </button>
</div>

<div class="main-container">
  
  <div class="header-section">
    <div class="titulo-pagina">Panel de Control</div>
    <div class="tabs">
        <button 
            class:activo={pestanaActual === 'inventario'} 
            on:click={() => pestanaActual = 'inventario'}>
            🚙 Inventario
        </button>
        <button 
            class:activo={pestanaActual === 'solicitudes'} 
            on:click={() => pestanaActual = 'solicitudes'}>
            📋 Solicitudes 
            {#if solicitudes.length > 0}
                <span class="contador">{solicitudes.length}</span>
            {/if}
        </button>
    </div>
  </div>

  <div class="contenido">
    {#if cargando}
      <div class="loading">Cargando datos...</div>
    {:else}

      {#if pestanaActual === 'inventario'}
        <div class="card-table">
          <table>
            <thead>
              <tr>
                <th>Propietario</th>
                <th>Cédula</th>
                <th>Vehículo</th>
                <th>Placa</th>
                <th>Estado</th>
              </tr>
            </thead>
            <tbody>
              {#each vehiculos as v}
                <tr>
                  <td class="resaltado">{v.Usuarios?.nombres} {v.Usuarios?.apellidos}</td>
                  <td>{v.Usuarios?.cedula}</td>
                  <td>{v.marca} {v.modelo} <span class="color-dot" style="background-color: {v.color}"></span></td>
                  <td><span class="placa-badge">{v.placa}</span></td>
                  <td>
                    {#if v.en_estacionamiento}
                      <span class="status dentro">En Parking (P{v.plaza_actual})</span>
                    {:else}
                      <span class="status fuera">Fuera</span>
                    {/if}
                  </td>
                </tr>
              {:else}
                <tr><td colspan="5" class="vacio">No hay vehículos registrados</td></tr>
              {/each}
            </tbody>
          </table>
        </div>
      {/if}

      {#if pestanaActual === 'solicitudes'}
        <div class="grid-cards">
          {#each solicitudes as s}
            <div class="card-solicitud">
              <div class="card-header">
                <span class="fecha">{new Date(s.created_at).toLocaleDateString()}</span>
                <strong>{s.Usuarios?.nombres} {s.Usuarios?.apellidos}</strong>
              </div>
              <div class="card-body">
                <div class="cambio-row">
                    <div class="col">
                        <small>Actual</small>
                        <div class="dato viejo">{s.Vehiculos?.placa || 'Sin Datos'}</div>
                    </div>
                    <div class="arrow">➜</div>
                    <div class="col">
                        <small>Solicitado</small>
                        <ul class="lista-cambios">
                            <li><strong>Placa:</strong> {s.datos_nuevos?.placa}</li>
                            <li><strong>Auto:</strong> {s.datos_nuevos?.marca} {s.datos_nuevos?.modelo}</li>
                            <li><strong>Color:</strong> {s.datos_nuevos?.color}</li>
                        </ul>
                    </div>
                </div>
              </div>
              <div class="card-actions">
                <button class="boton-accion rechazar" on:click={() => procesarSolicitud(s, 'rechazar')}>Rechazar</button>
                <button class="boton-accion aprobar" on:click={() => procesarSolicitud(s, 'aprobar')}>Aprobar</button>
              </div>
            </div>
          {:else}
            <div class="empty-state">
                <p>✨ No hay solicitudes pendientes</p>
            </div>
          {/each}
        </div>
      {/if}

    {/if}
  </div>
</div>

<style>
  @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;600;700;800&display=swap');

  :global(body) { margin: 0; font-family: 'Poppins', sans-serif; background: #f0f2f5; }

  /* BARRA SUPERIOR (Estilo Original) */
  .top {
    background: rgba(45, 45, 45, 0.95);
    backdrop-filter: blur(10px);
    position: fixed;
    top: 0; left: 0;
    width: 100%; height: 65px;
    display: flex; justify-content: space-between; align-items: center;
    z-index: 100;
    padding: 0 20px;
    box-sizing: border-box;
    border-bottom: 1px solid rgba(255, 255, 255, 0.1);
  }

  .logo-container { height: 100%; display: flex; align-items: center; gap: 15px; }
  .top img { height: 40px; filter: drop-shadow(0 2px 4px rgba(0,0,0,0.5)); }
  
  .admin-badge {
    color: #edf2f7;
    font-weight: 600;
  }

    .btn-logout { background: #e74c3c; color: white; border: none; padding: 8px 15px; border-radius: 6px; cursor: pointer; font-weight: 600; transition: 0.3s; }
    .btn-logout:hover { background: #c0392b; }

  /* CONTENEDOR PRINCIPAL */
  .main-container {
    margin-top: 65px;
    padding: 40px 20px;
    max-width: 1200px;
    margin-left: auto; margin-right: auto;
  }

  .header-section {
    display: flex; justify-content: space-between; align-items: center;
    margin-bottom: 30px; flex-wrap: wrap; gap: 20px;
  }

  .titulo-pagina {
    font-size: 1.8rem; font-weight: 800; color: #1a202c;
  }

  /* PESTAÑAS ESTILO ULEAM */
  .tabs {
    display: flex; background: white; padding: 5px; border-radius: 12px;
    box-shadow: 0 4px 6px -1px rgba(0,0,0,0.05);
  }

  .tabs button {
    background: transparent; border: none; padding: 10px 20px;
    font-family: 'Poppins', sans-serif; font-weight: 600; color: #718096;
    cursor: pointer; border-radius: 8px; transition: all 0.3s;
    display: flex; align-items: center; gap: 8px;
  }

  .tabs button.activo {
    background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
    color: white; shadow: 0 4px 12px rgba(0,123,255,0.3);
  }

  .contador {
    background: #e53e3e; color: white; font-size: 0.7rem; padding: 2px 6px;
    border-radius: 10px;
  }

  /* TABLA DE INVENTARIO */
  .card-table {
    background: white; border-radius: 20px; padding: 25px;
    box-shadow: 0 10px 25px -5px rgba(0,0,0,0.05); overflow-x: auto;
  }

  table { width: 100%; border-collapse: collapse; }
  th { text-align: left; color: #a0aec0; font-size: 0.85rem; text-transform: uppercase; padding: 15px; border-bottom: 2px solid #edf2f7; }
  td { padding: 15px; border-bottom: 1px solid #edf2f7; color: #4a5568; font-size: 0.95rem; }
  
  .resaltado { font-weight: 700; color: #2d3748; }
  .placa-badge {
    background: #ffeb3b; color: #000; font-weight: 800;
    padding: 5px 10px; border-radius: 6px; border: 2px solid #fdd835;
  }
  
  .status { padding: 5px 10px; border-radius: 20px; font-size: 0.8rem; font-weight: 600; }
  .status.dentro { background: #c6f6d5; color: #22543d; }
  .status.fuera { background: #edf2f7; color: #718096; }

  /* CARDS SOLICITUDES */
  .grid-cards { display: grid; grid-template-columns: repeat(auto-fill, minmax(320px, 1fr)); gap: 25px; }
  
  .card-solicitud {
    background: white; border-radius: 20px; padding: 25px;
    box-shadow: 0 10px 20px -5px rgba(0,0,0,0.05);
    border: 1px solid rgba(0,0,0,0.02);
    display: flex; flex-direction: column; justify-content: space-between;
  }

  .card-header { border-bottom: 1px solid #edf2f7; padding-bottom: 15px; margin-bottom: 15px; }
  .card-header strong { display: block; font-size: 1.1rem; color: #2d3748; }
  .fecha { font-size: 0.8rem; color: #a0aec0; }

  .cambio-row { display: flex; align-items: center; gap: 10px; margin-bottom: 20px; }
  .col small { display: block; text-transform: uppercase; font-size: 0.7rem; color: #a0aec0; margin-bottom: 5px; font-weight: 700; }
  .dato.viejo { background: #edf2f7; padding: 8px; border-radius: 8px; color: #718096; text-decoration: line-through; }
  .arrow { color: #007bff; font-weight: 800; }
  
  .lista-cambios { list-style: none; padding: 0; margin: 0; font-size: 0.9rem; }
  .lista-cambios li { margin-bottom: 4px; color: #2d3748; }

  .card-actions { display: flex; gap: 10px; }
  
  .boton-accion {
    flex: 1; padding: 12px; border: none; border-radius: 12px;
    font-family: 'Poppins', sans-serif; font-weight: 700; cursor: pointer;
    transition: transform 0.2s;
  }
  .boton-accion:hover { transform: translateY(-2px); }
  
  .boton-accion.aprobar {
    background: linear-gradient(135deg, #007bff 0%, #0056b3 100%); color: white;
    box-shadow: 0 4px 10px rgba(0,123,255,0.3);
  }
  .boton-accion.rechazar {
    background: #fff; border: 2px solid #fed7d7; color: #e53e3e;
  }
  .boton-accion.rechazar:hover { background: #fff5f5; border-color: #e53e3e; }

  .empty-state { width: 100%; text-align: center; padding: 50px; color: #a0aec0; font-size: 1.2rem; }
</style>