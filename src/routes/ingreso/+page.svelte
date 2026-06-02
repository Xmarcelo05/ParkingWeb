<script>
    import { onMount } from 'svelte';
    import { goto } from '$app/navigation';
    import { supabase } from '$lib/supabaseClient';

    onMount(() => {
        const usuario = localStorage.getItem('usuarioActual');
        if (!usuario) {
            goto('/', { replaceState: true });
        }
        cargarDatosSupabase();
    });

    // --- ESTADO GLOBAL ---
    let plazasEstacionamiento = new Array(98).fill(false);
    let registrosCargados = []; 
    
    // Contadores reactivos
    $: ocupados = plazasEstacionamiento.filter(Boolean).length;
    $: disponibles = plazasEstacionamiento.length - ocupados;
    
    // --- MODALES ---
    let mostrarModalPlaza = false;
    let plazaSeleccionada = null; 
    let inputNombre = ''; 
    let inputPlaca = '';
    
    let mostrarModalRegistro = false;
    let tablaRegistroHTML = ''; 
    
    // Configuración de las plazas
    const filas = [
      { start: 1, count: 26, top: 4.4, marginLeft: 10.4, marginRight: 1.5, plazaHeight: 13, widthFactor: 0.8, innerOffset: 0, shiftPct: 0 },
      { start: 27, count: 23, top: 34.2, marginLeft: 10.4, marginRight: 11.7, plazaHeight: 13, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 },
      { start: 50, count: 23, top: 54, marginLeft: 10.4, marginRight: 11.7, plazaHeight: 13, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 },
      { start: 73, count: 26, top: 83, marginLeft: 10.4, marginRight: 1.5, plazaHeight: 13, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 }
    ];

    // --- 3. LÓGICA DE DATOS (SUPABASE) ---
    async function cargarDatosSupabase() {
        try {
            const { data, error } = await supabase
                .from('Vehiculos')
                .select('*')
                .eq('en_estacionamiento', true);

            if (error) throw error;

            registrosCargados = data;
            
            let nuevoEstadoPlazas = new Array(98).fill(false);
            data.forEach(v => {
                if (v.plaza_actual && v.plaza_actual >= 1 && v.plaza_actual <= 98) {
                    nuevoEstadoPlazas[v.plaza_actual - 1] = true;
                }
            });
            plazasEstacionamiento = nuevoEstadoPlazas; 

        } catch (error) {
            console.error('Error al cargar datos:', error.message);
        }
    }
    
    function openRegistroPlazaModal(numero) {
        if (plazasEstacionamiento[numero - 1]) {
            alert('Esta plaza ya está ocupada');
            return;
        }
        inputNombre = '';
        inputPlaca = '';
        plazaSeleccionada = numero;
        mostrarModalPlaza = true;
    }

    function cerrarModalPlaza() { mostrarModalPlaza = false; }
    
    // LÓGICA PRINCIPAL: REGISTRAR EL INGRESO
    async function manejarRegistroPlaza(e) {
        e.preventDefault(); 
        
        if (!inputPlaca) {
            alert("Por favor ingrese la placa del vehículo.");
            return;
        }

        const placaBuscar = inputPlaca.trim().toUpperCase();

        try {
            // 1. Verificar si el vehículo existe
            const { data: vehiculo, error } = await supabase
                .from('Vehiculos')
                .select('*')
                .eq('placa', placaBuscar)
                .maybeSingle();

            if (error) throw error;

            if (!vehiculo) {
                alert("🚫 ERROR: Vehículo no registrado en el sistema.");
                return;
            }

            // 2. Verificar si ya está adentro
            if (vehiculo.en_estacionamiento) {
                alert(`⚠️ Este vehículo ya figura dentro del estacionamiento (Plaza ${vehiculo.plaza_actual}).`);
                return;
            }

            // 3. Registrar Ingreso
            const { error: updateError } = await supabase
                .from('Vehiculos')
                .update({ 
                    en_estacionamiento: true, 
                    plaza_actual: plazaSeleccionada 
                })
                .eq('id', vehiculo.id);

            if (updateError) throw updateError;

            // 4. GUARDAR EN HISTORIAL
            await supabase.from('Historial_Ingresos').insert({
                placa: placaBuscar,
                plaza: plazaSeleccionada,
                tipo_movimiento: 'entrada'
            });

            alert(`✅ Acceso concedido.\nVehículo: ${vehiculo.marca} ${vehiculo.modelo}\nPlaza: ${plazaSeleccionada}`);
            
            cargarDatosSupabase();
            cerrarModalPlaza();

        } catch (err) {
            alert("Error de conexión: " + err.message);
        }
    }
    
    function generarTablaRegistro() {
      let tabla = `<table class="registro-table"><thead><tr><th>#</th><th>Vehículo</th><th>Placa</th><th>Plaza</th><th>Estado</th></tr></thead><tbody>`;
      
      registrosCargados.forEach((r, i) => {
        tabla += `<tr>
            <td>${i + 1}</td>
            <td>${r.marca || ''} ${r.modelo || ''}</td>
            <td><strong>${r.placa}</strong></td>
            <td>${r.plaza_actual}</td>
            <td style="color:green; font-weight:bold">En Sitio</td>
        </tr>`;
      });
      tabla += `</tbody></table>`;
      tablaRegistroHTML = tabla;
    }

    function mostrarRegistroVehiculos() {
      generarTablaRegistro();
      mostrarModalRegistro = true; 
    }

    function cerrarSesion() {
        localStorage.removeItem('usuarioActual');
        goto('/', { replaceState: true });
    }
    
    function volverHome() { goto('/home'); }
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
        <div class="info-box-ingreso">
            <h3>Registrar Vehículo</h3>
            <p>Seleccione una plaza en el mapa</p>
        </div>

        <div class="info-panel">
            <div>Espacios Disponibles: <span id="espaciosDisponibles">{disponibles}</span></div>
            <div>Espacios Ocupados: <span id="espaciosOcupados">{ocupados}</span></div>
        </div>

        <div class="footer-buttons">
            <button on:click={mostrarRegistroVehiculos} class="btn small">Registro</button>
            <button on:click={volverHome} class="btn small" style="background: #e74c3c;">Volver</button>
        </div>
    </div>

    <div class="map-container">
        <img id="mapImage" src="/Imagenes/EsquemaPark.png" alt="Estacionamiento" />
        
        <div id="estacionamiento" class="overlay">
            {#each filas as row}
                {@const count = row.count}
                {@const marginLeft = row.marginLeft || 2.5}
                {@const marginRight = row.marginRight || 2.5}
                {@const plazaHeightPct = row.plazaHeight || 6}
                {@const widthFactor = row.widthFactor || 0.9}
                {@const spacing = (100 - marginLeft - marginRight) / count}
                
                {#each Array(count) as _, i}
                    {@const numero = row.start + i}
                    {@const leftPct = marginLeft + i * spacing + spacing * (row.innerOffset || 0.05) + (row.shiftPct || 0)}
                    {@const widthPct = spacing * widthFactor} 
                    {@const topPct = row.top}
                    
                    <div 
                        class="plaza" 
                        class:ocupada={plazasEstacionamiento[numero - 1]}
                        style="left: {leftPct}%; top: {topPct}%; width: {widthPct}%; height: {plazaHeightPct}%;"
                        data-numero={numero}
                        on:click={() => openRegistroPlazaModal(numero)}
                    >
                        {numero}
                    </div>
                {/each}
            {/each}
        </div>
    </div>
</main>

{#if mostrarModalPlaza}
    <div class="modal">
        <div class="modal-content">
            <span class="close" on:click={cerrarModalPlaza}>&times;</span>
            <h3>Registro en Plaza {plazaSeleccionada}</h3>
            
            <form on:submit|preventDefault={manejarRegistroPlaza} class="form-registro">
                <div style="margin-bottom: 15px;">
                    <label style="display:block; font-weight:600; margin-bottom:5px;">Placa Vehículo</label>
                    <input bind:value={inputPlaca} type="text" required placeholder="Ingrese Placa" style="width:100%; padding:12px; border:2px solid #eee; border-radius:8px; font-size: 1.1rem; text-transform: uppercase;" />
                </div>
                <button type="submit" class="btn small" style="margin-top: 10px;">Validar e Ingresar</button>
            </form>
        </div>
    </div>
{/if}

{#if mostrarModalRegistro}
    <div class="modal">
        <div class="modal-content">
            <span class="close" on:click={() => mostrarModalRegistro = false}>&times;</span>
            <h3>Vehículos Dentro</h3>
            <div class="table-wrapper">
                {@html tablaRegistroHTML}
            </div>
        </div>
    </div>
{/if}

<style>
    @import url('https://fonts.googleapis.com/css2?family=Poppins:wght@400;600;700;800&display=swap');

    :global(body) {
        margin: 0; padding: 0;
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
        position: relative; 
    }
    .logo-section { display: flex; align-items: center; gap: 15px; font-weight: 600; }
    .logo-section img { height: 40px; }
    .btn-exit { 
        background: #e74c3c; 
        color: white; 
        border: none; 
        padding: 8px 15px; 
        border-radius: 6px; cursor: 
        pointer; font-weight: 600; 
        transition: 0.3s; 
    }
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
        gap: 20px; 
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
        height: 467PX; 
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

    .info-box-ingreso { margin-bottom: 20px; text-align: center; }
    .info-box-ingreso h3 { margin: 0; color: #333; font-weight: 800; font-size: 1.5rem; }
    .info-box-ingreso p { color: #666; font-size: 0.9rem; margin: 5px 0 0; }

    .info-panel { background: #ffffff; padding: 15px; border-radius: 10px; font-weight: bold; font-size: 0.9rem; }
    
    .footer-buttons { display: flex; flex-direction: column; gap: 10px; }
    
    .btn { width: 100%; border: none; border-radius: 10px; padding: 15px 5px; cursor: pointer; font-weight: bold; display: block; text-align: center; }
    
    .small {
        width: 100%; padding: 14px;
        background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
        color: white; 
        box-shadow: 0 5px 15px rgba(0, 123, 255, 0.3);
        transition: 0.2s;
    }
    .small:hover { background: linear-gradient(135deg, #0056b3 0%, #003f7f 100%) }

    /* ESTILOS DEL MODAL IGUALADOS AL HOME */
    .modal {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: rgba(0,0,0,0.7);
        display: flex; justify-content: center; align-items: center;
        z-index: 2000;
    }
    
    .modal-content {
        background: white;
        padding: 40px; /* Igual al Home */
        border-radius: 20px; /* Igual al Home */
        width: 90%; /* Igual al Home */
        max-width: 900px; /* Igual al Home */
        max-height: 90vh; /* Igual al Home */
        overflow-y: auto;
        box-shadow: 0 20px 50px rgba(0,0,0,0.3); /* Igual al Home */
    }

    .modal-content h3 {
        margin-top: 0;
        margin-bottom: 25px;
        font-size: 1.8rem;
        color: #1a202c;
        text-align: center;
    }

    .table-wrapper { overflow-x: auto; }
    
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