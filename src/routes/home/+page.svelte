<script>
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';
    import { supabase } from '$lib/supabaseClient';

    let usuarioGuardia = null;

    onMount(async () => {
        const sesion = localStorage.getItem('usuarioActual');
        if (!sesion) {
            goto('/', { replaceState: true });
        } else {
            usuarioGuardia = JSON.parse(sesion);
            await cargarDatosSupabase(); 
        }
    });

    // --- 1. ESTADO ---
    let plazasEstacionamiento = new Array(98).fill(false); 
    let registrosVehiculos = []; 
    
    $: ocupados = plazasEstacionamiento.filter(Boolean).length;
    $: disponibles = plazasEstacionamiento.length - ocupados;
    
    let mostrarModalRegistro = false;
    let tablaRegistroHTML = ''; 

    // Coordenadas de las filas de plazas
    const filas = [
      { start: 1, count: 26, top: 4.4, marginLeft: 10.4, marginRight: 1.5, plazaHeight: 13, widthFactor: 0.8, innerOffset: 0, shiftPct: 0 },
      { start: 27, count: 23, top: 34.2, marginLeft: 10.4, marginRight: 11.7, plazaHeight: 13, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 },
      { start: 50, count: 23, top: 54, marginLeft: 10.4, marginRight: 11.7, plazaHeight: 13, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 },
      { start: 73, count: 26, top: 83, marginLeft: 10.4, marginRight: 1.5, plazaHeight: 13, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 }
    ];

    // --- 2. LÓGICA SUPABASE ---

    //Cargar el Mapa (Estado del Parking)
    async function cargarDatosSupabase() {
        try {
            // Buscamos autos que están dentro
            const { data, error } = await supabase
                .from('Vehiculos')
                .select('plaza_actual')
                .eq('en_estacionamiento', true);

            if (error) throw error;

            let nuevoEstado = new Array(98).fill(false);
            
            if (data) {
                data.forEach(auto => {
                    if (auto.plaza_actual && auto.plaza_actual >= 1 && auto.plaza_actual <= 98) {
                        nuevoEstado[auto.plaza_actual - 1] = true;
                    }
                });
            }
            plazasEstacionamiento = nuevoEstado;

        } catch (error) {
            console.error('Error cargando mapa:', error);
        }
    }

    //Generar Tabla de Historial
    async function generarTablaRegistro() {
        const { data: historial, error } = await supabase
            .from('Historial_Ingresos')
            .select('*')
            .order('created_at', { ascending: false })
            .limit(98);

        if (error) {
            console.error(error);
            return;
        }

        // 2. Construir la tabla HTML
        let tabla = `<table class="registro-table">
            <thead>
                <tr>
                    <th>Hora</th>
                    <th>Movimiento</th>
                    <th>Placa</th>
                    <th>Plaza</th>
                </tr>
            </thead>
            <tbody>`;
        
        historial.forEach((r) => {
            const fecha = new Date(r.created_at).toLocaleTimeString([], {hour: '2-digit', minute:'2-digit'});
            const color = r.tipo_movimiento === 'entrada' ? '#27ae60' : '#c0392b';
            const movimiento = r.tipo_movimiento.toUpperCase();

            tabla += `<tr>
                <td>${fecha}</td>
                <td style="color:${color}; font-weight:bold;">${movimiento}</td>
                <td>${r.placa}</td>
                <td>${r.plaza || '-'}</td>
            </tr>`;
        });
        
        tabla += `</tbody></table>`;
        tablaRegistroHTML = tabla;
    }
    
    async function mostrarRegistroVehiculos() {
        await generarTablaRegistro(); // Esperar a que traiga los datos
        mostrarModalRegistro = true;
    }
    
    function cerrarSesion() {
        localStorage.removeItem('usuarioActual');
        goto('/', { replaceState: true }); 
    }

    function irAIngreso() { goto('/ingreso'); }
    function irASalida() { goto('/salida'); }
</script>

<div class="top">
    <div class="logo-section">
        <img src="/Imagenes/LogoUleam.png" alt="Logo Uleam">
        <span>Parking ULEAM - Guardia</span>
    </div>
    <button class="btn-exit" on:click={cerrarSesion}>
      <i class="fas fa-sign-out-alt"></i> Cerrar Sesión
    </button>
</div>

<div class="fondo-app"></div>

    <main class="main-layout">
        <div class="left-panel">
            <div class="buttons-row">
                <button on:click={irAIngreso} class="btn ingreso">
                    <img src="/Imagenes/Ingreso.png" alt="Ingreso">
                    <span>INGRESO</span>
                </button>
                <button on:click={irASalida} class="btn salida">
                    <img src="/Imagenes/Salida.png" alt="Salida">
                    <span>SALIDA</span>
                </button>
            </div>

            <div class="info-panel">
                <div>Espacios Disponibles: <span id="espaciosDisponibles">{disponibles}</span></div>
                <div>Espacios Ocupados: <span id="espaciosOcupados">{ocupados}</span></div>
            </div>

            <div class="footer-buttons">
                <button on:click={mostrarRegistroVehiculos} class="btn small">Registro</button>
            </div>
        </div>

        <div class="map-container">
            <img id="mapImage" src="/Imagenes/EsquemaPark.png" alt="Estacionamiento" />
            <div id="estacionamiento" class="overlay">
                {#each filas as row}
                    {@const count = row.count}
                    {@const marginLeft = row.marginLeft || 2.5}
                    {@const marginRight = row.marginRight || 2.5}
                    {@const spacing = (100 - marginLeft - marginRight) / count}
                    {#each Array(count) as _, i}
                        {@const numero = row.start + i}
                        {@const leftPct = marginLeft + i * spacing + spacing * (row.innerOffset || 0.05) + (row.shiftPct || 0)}
                        <div 
                            class="plaza" 
                            class:ocupada={plazasEstacionamiento[numero - 1]}
                            style="left: {leftPct}%; top: {row.top}%; width: {spacing * (row.widthFactor || 0.9)}%; height: {row.plazaHeight || 6}%;"
                        >
                            {numero}
                        </div>
                    {/each}
                {/each}
            </div>
        </div>
    </main>

{#if mostrarModalRegistro}
    <div class="modal">
        <div class="modal-content">
            <span class="close" on:click={() => mostrarModalRegistro = false}>&times;</span>
            <h3>Registro de Vehículos</h3>
            <div class="table-wrapper">
                {@html tablaRegistroHTML}
            </div>
        </div>
    </div>
{/if}

<style>
    @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;600;700;800&display=swap');

    :global(body) {
        margin: 0;
        padding: 0;
        font-family: 'Poppins', sans-serif;
        background-color: #f4f4f4;
        overflow: hidden;
    }

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
    .logo-section { display: flex; align-items: center; gap: 15px; font-weight: 600; }
    .logo-section img { height: 40px; }
    .btn-exit { background: #e74c3c; color: white; border: none; padding: 8px 15px; border-radius: 6px; cursor: pointer; font-weight: 600; transition: 0.3s; }
    .btn-exit:hover { background: #c0392b; }
    
    .fondo-app {
        position: fixed; top: -200px; left: -20px; width: 102%; height: 150%;
        background-image: url("/Imagenes/FondoUleam.jpg");
        background-size: cover; background-position: center;
        filter: blur(8px) brightness(0.6);
        z-index: -1;
    }
    
    .main-layout {
        max-width: 1300px;
        margin: 200px auto; 
        height: calc(60vh - 60px);
        display: flex;
        align-items: center;
        justify-content: center;
    }

    .left-panel {
        width: 320px;
        height: 467px;
        background: rgba(255, 255, 255, 0.9);
        padding: 20px;
        border-radius: 15px;
        display: flex;
        flex-direction: column;
        justify-content: space-between;
        box-shadow: 0 10px 30px rgba(0,0,0,0.3);
    }

    .map-container {
        flex: 1;
        position: relative;
        background: rgba(255, 255, 255, 0.9);
        border-radius: 15px;
        padding: 10px;
        box-shadow: 0 10px 30px rgba(0,0,0,0.3);
        display: flex;
        align-items: center;
        justify-content: center;
    }

    #mapImage { max-width: 100%; max-height: 100%; object-fit: contain; }
    .overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: none; }
    .plaza {
        position: absolute;
        border: 2px solid #28a745;
        background: rgba(40, 167, 69, 0.6);
        color: white;
        font-weight: bold;
        font-size: 0.7rem;
        display: flex;
        align-items: center;
        justify-content: center;
    }
    .plaza.ocupada { border-color: #dc3545; background: rgba(220, 53, 69, 0.8); }

    /* BOTONES */
    .buttons-row { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
    .btn { 
        border: none; border-radius: 10px; padding: 15px 5px; cursor: pointer;
        display: flex; flex-direction: column; align-items: center; gap: 5px; font-weight: bold;
    }
    
    .ingreso { background: rgba(31, 149, 58, 0.8); }
    .salida { background: rgba(220, 53, 69, 0.8); }
    .btn img { height: 50px; }

    .info-panel { background: #ffffff; padding: 15px; border-radius: 10px; font-weight: bold; font-size: 0.9rem; }
    .footer-buttons { display: flex; flex-direction: column; gap: 10px; }
    .small {
        width: 100%; padding: 14px;
        background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
        color: white; border: none; border-radius: 10px;
        margin-top: 50px; font-weight: 700; cursor: pointer;
        box-shadow: 0 5px 15px rgba(0, 123, 255, 0.3);
        transition: 0.2s;
    }
    .small:hover { background: linear-gradient(135deg, #0056b3 0%, #003f7f 100%); }

    .modal {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: rgba(0,0,0,0.7);
        display: flex; justify-content: center; align-items: center;
        z-index: 2000;
    }
    
    .modal-content {
        background: white;
        padding: 40px;
        border-radius: 20px;
        width: 90%;
        max-width: 900px;
        max-height: 90vh;
        overflow-y: auto;
        box-shadow: 0 20px 50px rgba(0,0,0,0.3);
    }

    .modal-content h3 {
        margin-top: 0;
        margin-bottom: 25px;
        font-size: 1.8rem;
        color: #1a202c;
        text-align: center;
    }

    :global(.registro-table) {
        width: 100%;
        border-collapse: collapse;
        margin-top: 10px;
    }

    :global(.registro-table th) {
        background: #f1f5f9;
        padding: 16px;
        text-align: left;
        font-size: 1rem;
        color: #334155;
        font-weight: 800;
        border-bottom: 2px solid #e2e8f0;
    }

    :global(.registro-table td) {
        padding: 16px;
        border-bottom: 1px solid #e2e8f0;
        font-size: 1rem;
        color: #1e293b;
        font-weight: 500;
    }

    .close {
        float: right;
        cursor: pointer;
        font-size: 2rem;
        font-weight: bold;
        color: #ef4444;
        line-height: 20px;
    }
    .close:hover { color: #dc2626; }
</style>