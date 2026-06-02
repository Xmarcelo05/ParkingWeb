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

    // --- 1. ESTADO GLOBAL ---
    let plazasEstacionamiento = new Array(98).fill(false);
    
    // Contadores reactivos
    $: ocupados = plazasEstacionamiento.filter(Boolean).length;
    $: disponibles = plazasEstacionamiento.length - ocupados;
    
    // --- 2. MODALES ---
    let mostrarModalConfirmacion = false;
    let mostrarPanelResultado = false;
    
    let plazaSeleccionada = null;
    let inputPlacaConfirmacion = '';
    let registroSeleccionado = null; 
    let mostrarModalRegistro = false;
    let tablaRegistroHTML = ''; 
    let registrosCargados = [];

    // Configuración de las plazas
    const filas = [
      { start: 1, count: 26, top: 2.2, marginLeft: 9.5, marginRight: 0.4, plazaHeight: 13.8, widthFactor: 0.8, innerOffset: 0, shiftPct: 0 },
      { start: 27, count: 23, top: 33.3, marginLeft: 9.5, marginRight: 10.9, plazaHeight: 13.7, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 },
      { start: 50, count: 23, top: 54.1, marginLeft: 9.5, marginRight: 10.9, plazaHeight: 13.6, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 },
      { start: 73, count: 26, top: 84.3, marginLeft: 9.5, marginRight: 0.4, plazaHeight: 13.7, widthFactor: 0.8, innerOffset: 0.05, shiftPct: 0 }
    ];

    // --- 3. LÓGICA DE DATOS ---
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
    
    // Al hacer clic en una plaza del mapa
    function manejarClickPlaza(numero) {
        if (plazasEstacionamiento[numero - 1]) {
            plazaSeleccionada = numero;
            inputPlacaConfirmacion = ''; 
            mostrarPanelResultado = false;
            mostrarModalConfirmacion = true; 
        } else {
            alert('Esta plaza está libre. Solo puede registrar salida de plazas ocupadas.');
        }
    }
    
    function cerrarModalConfirmacion() {
        mostrarModalConfirmacion = false;
        plazaSeleccionada = null;
    }
    
    // VERIFICACIÓN DE PLACA
    async function manejarVerificacionPlaca(e) {
        e.preventDefault(); 
        
        const placaIngresada = inputPlacaConfirmacion.trim().toUpperCase();
        
        if (!placaIngresada) {
            alert('Por favor ingrese la placa del vehículo.');
            return;
        }

        try {
            // Se busca el vehículo en la plaza seleccionada
            const { data: vehiculoEnPlaza, error } = await supabase
                .from('Vehiculos')
                .select('*')
                .eq('plaza_actual', plazaSeleccionada)
                .eq('en_estacionamiento', true)
                .maybeSingle();

            if (error || !vehiculoEnPlaza) {
                alert("Error: No se encontró registro activo en esta plaza en la base de datos.");
                return;
            }

            // Validamos coincidencia
            if (vehiculoEnPlaza.placa === placaIngresada) {

                registroSeleccionado = vehiculoEnPlaza;
                mostrarModalConfirmacion = false; 
                mostrarPanelResultado = true;
            } else {
                alert(`La placa ingresada (${placaIngresada}) no coincide con el vehículo en la plaza ${plazaSeleccionada}. (El sistema indica: ${vehiculoEnPlaza.placa})`);
            }
        } catch (err) {
            alert("Error de conexión: " + err.message);
        }
    }
    
    // CONFIRMAR SALIDA FINAL
    async function confirmarSalida() {
        if (!registroSeleccionado) return;
        
        try {
            // Liberar el Vehículo (Actualizar BD)
            const { error: updateError } = await supabase
                .from('Vehiculos')
                .update({ 
                    en_estacionamiento: false, 
                    plaza_actual: null 
                })
                .eq('id', registroSeleccionado.id);

            if (updateError) throw updateError;

            //  Guardar en HISTORIAL
            await supabase.from('Historial_Ingresos').insert({
                placa: registroSeleccionado.placa,
                plaza: plazaSeleccionada,
                tipo_movimiento: 'salida'
            });

            alert(`Salida registrada exitosamente.`);
            
            cargarDatosSupabase(); // Actualizar mapa
            mostrarPanelResultado = false;
            registroSeleccionado = null;

        } catch (err) {
            alert('Error al procesar la salida: ' + err.message);
        }
    }

    function generarTablaRegistro() {
      let tabla = `<table class="registro-table"><thead><tr><th>#</th><th>Placa</th><th>Marca/Modelo</th><th>Plaza</th></tr></thead><tbody>`;
      registrosCargados.forEach((r, i) => {
        tabla += `<tr>
            <td>${i + 1}</td>
            <td><strong>${r.placa}</strong></td>
            <td>${r.marca || ''} ${r.modelo || ''}</td>
            <td>${r.plaza_actual}</td>
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
            <h3>Registrar Salida</h3>
            <p>Seleccione una plaza ocupada (roja) para liberar</p>
        </div>

        <div class="info-panel">
            <div>Espacios Disponibles: <span id="espaciosDisponibles">{disponibles}</span></div>
            <div>Espacios Ocupados: <span id="espaciosOcupados">{ocupados}</span></div>
        </div>
        
        {#if mostrarPanelResultado && registroSeleccionado}
            <div id="resultadoSalida" class="info-panel resultado-salida">
                <div style="margin-bottom:5px; font-size:1.1em; color:#d9534f;"><strong>Vehículo a liberar:</strong></div>
                <div>Vehículo: {registroSeleccionado.marca} {registroSeleccionado.modelo}</div> 
                <div>Placa: <strong>{registroSeleccionado.placa}</strong></div>
                <div>Plaza: {registroSeleccionado.plaza_actual}</div>
                
                <div style="margin-top:15px">
                    <button on:click={confirmarSalida} class="btn small" style="background: #dc3545; box-shadow: none;">Confirmar Liberación</button>
                </div>
            </div>
        {/if}

        <div class="footer-buttons">
            <button on:click={mostrarRegistroVehiculos} class="btn small">Registro</button>
            <button on:click={volverHome} class="btn small" style="background: #e74c3c;">Volver</button>
        </div>
    </div>

    <div class="map-container">
        <div class="map-wrapper">
            
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
                        {@const leftPct = (row.marginLeft || 2.5) + i * spacing + spacing * (row.innerOffset || 0.05) + (row.shiftPct || 0)}
                        {@const widthPct = spacing * widthFactor} 
                        {@const topPct = row.top}
                        
                        <div 
                            class="plaza" 
                            class:ocupada={plazasEstacionamiento[numero - 1]}
                            style="left: {leftPct}%; top: {topPct}%; width: {widthPct}%; height: {plazaHeightPct}%;"
                            data-numero={numero}
                            on:click={() => manejarClickPlaza(numero)}
                        >
                            {numero}
                        </div>
                    {/each}
                {/each}
            </div>
        </div>
    </div>
</main>

{#if mostrarModalConfirmacion}
    <div id="modalConfirmacion" class="modal">
        <div class="modal-content">
            <span class="close" on:click={cerrarModalConfirmacion}>&times;</span>
            <h3>Confirmar Salida - Plaza {plazaSeleccionada}</h3>
            
            <form on:submit|preventDefault={manejarVerificacionPlaca} class="form-registro">
                <p style="margin-bottom: 15px; color: #555;">Ingrese la placa para verificar que corresponde al vehículo correcto:</p>
                <div style="margin-bottom: 15px;">
                    <label style="display:block; font-weight:600; margin-bottom:5px;">Placa del vehículo</label>
                    <input bind:value={inputPlacaConfirmacion} type="text" placeholder="Ej: ABC-1234" required style="width:100%; padding:8px; border:1px solid #ccc; border-radius:5px; font-size:1.1rem; text-transform:uppercase;" />
                </div>

                <div class="form-actions">
                    <button type="button" on:click={cerrarModalConfirmacion} class="btn-cancel">Cancelar</button>
                    <button type="submit" class="btn-confirm">Verificar</button>
                </div>
            </form>
        </div>
    </div>
{/if}

{#if mostrarModalRegistro}
    <div class="modal">
        <div class="modal-content table-content">
            <span class="close" on:click={() => mostrarModalRegistro = false}>&times;</span>
            <h3>Vehículos en Estacionamiento</h3>
            <div class="table-wrapper">
                {@html tablaRegistroHTML}
            </div>
        </div>
    </div>
{/if}

<style>
    /* CSS ORIGINAL DE TU CÓDIGO (INTACTO) */
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
        background: rgba(255, 255, 255, 0.9);
        border-radius: 15px;
        padding: 10px;
        box-shadow: 0 10px 30px rgba(0,0,0,0.3);
        
        /* Centrado y Grid Fix */
        display: flex;
        align-items: center;
        justify-content: center;
        height: 467px; 
        overflow: hidden; 
    }

    /* Apilar imagen y botones perfectamente */
    .map-wrapper {
        display: grid;
        grid-template-areas: "stack";
        position: relative;
        width: fit-content;
        height: fit-content;
        max-width: 100%;
        max-height: 100%;
    }

    #mapImage {
        grid-area: stack; /* Ocupa el mismo espacio que el overlay */
        width: 100%;
        height: auto;
        max-height: 100%;
        object-fit: contain;
        display: block;
    }

    .overlay {
        grid-area: stack; 
        position: relative; 
        width: 100%;
        height: 100%;
        z-index: 10;
        top: 0; left: 0;
    }
    
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
        cursor: pointer;
        z-index: 20; 
    }
    .plaza.ocupada { border-color: #dc3545; background: rgba(220, 53, 69, 0.8); }


    .info-box-ingreso { margin-bottom: 20px; text-align: center; }
    .info-box-ingreso h3 { margin: 0; color: #333; font-weight: 800; font-size: 1.5rem; }
    .info-box-ingreso p { color: #666; font-size: 0.9rem; margin: 5px 0 0; }

    .info-panel { background: #ffffff; padding: 15px; border-radius: 10px; font-weight: bold; font-size: 0.9rem; }
    .resultado-salida { border-left: 5px solid #dc3545; background: #fff5f5; margin-bottom: 10px; }
    
    .footer-buttons { display: flex; flex-direction: column; gap: 10px; }
    
    .btn { width: 100%; border: none; border-radius: 10px; padding: 15px 5px; cursor: pointer; font-weight: bold; display: block; text-align: center; color: white; transition: 0.2s; }
    .small {
        width: 100%; padding: 14px;
        background: linear-gradient(135deg, #007bff 0%, #0056b3 100%);
        color: white; 
        box-shadow: 0 5px 15px rgba(0, 123, 255, 0.3);
        transition: 0.2s;
    }
    .small:hover { background: linear-gradient(135deg, #0056b3 0%, #003f7f 100%) }

    /* MODAL */
    .modal { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.7); display: flex; justify-content: center; align-items: center; z-index: 2000; }
    .modal-content { background: white; padding: 30px; border-radius: 15px; width: 90%; max-width: 400px; box-shadow: 0 25px 50px rgba(0,0,0,0.5); }
    .table-content { max-width: 900px; max-height: 80vh; overflow-y: auto; } /* Ajuste para tabla grande */
    
    .close { float: right; cursor: pointer; font-size: 1.5rem; color: #999; }
    
    .form-actions { display: flex; gap: 10px; margin-top: 20px; }
    .btn-cancel { flex: 1; padding: 10px; border: none; background: #dc3545; border-radius: 8px; cursor: pointer; font-weight: 600; }
    .btn-confirm { flex: 1; padding: 10px; border: none; background: #0056b3; color: white; border-radius: 8px; cursor: pointer; font-weight: 600; }

    .table-wrapper { overflow-x: auto; }
    :global(.registro-table) { width: 100%; border-collapse: collapse; margin-top: 15px; }
    :global(.registro-table th) { background: #eee; padding: 10px; text-align: left; }
    :global(.registro-table td) { padding: 10px; border-bottom: 1px solid #eee; }
</style>