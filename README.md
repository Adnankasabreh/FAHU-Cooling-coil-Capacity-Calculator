<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>FAHU Cooling Capacity Calculator</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f0d0f0;
            margin: 0;
            padding: 20px;
        }
        .calculator {
            background-color: #e6b3e6;
            padding: 20px;
            border-radius: 10px;
            max-width: 600px;
            margin: 0 auto;
        }
        h1 {
            color: #4a004a;
            text-align: center;
        }
        .input-group {
            margin-bottom: 10px;
        }
        label {
            display: inline-block;
            width: 200px;
        }
        input[type="number"], select {
            width: 100px;
        }
        .buttons {
            text-align: center;
            margin: 20px 0;
        }
        button {
            padding: 10px 20px;
            margin: 0 10px;
        }
        .results {
            display: grid;
            grid-template-columns: 1fr 1fr;
            gap: 10px;
        }
    </style>
</head>
<body>
    <div class="calculator">
        <h1>FAHU Cooling Capacity Calculator</h1>
        
        <div class="input-group">
            <label for="airFlow">Air Flow Rate:</label>
            <input type="number" id="airFlow" step="0.01"> L/s
        </div>
        
        <div class="input-group">
            <label for="onCoilDB">On Coil DB Temp:</label>
            <input type="number" id="onCoilDB" step="0.01"> °C
        </div>
        
        <div class="input-group">
            <label for="onCoilWB">On Coil WB Temp:</label>
            <input type="number" id="onCoilWB" step="0.01"> °C
        </div>
        
        <div class="input-group">
            <label for="offCoilDB">Off Coil DB Temp:</label>
            <input type="number" id="offCoilDB" step="0.01"> °C
        </div>
        
        <div class="input-group">
            <label for="offCoilParam">Off Coil Parameter:</label>
            <select id="offCoilParam" onchange="toggleOffCoilInput()">
                <option value="RH">RH (%)</option>
                <option value="WB">WB (°C)</option>
            </select>
        </div>
        
        <div class="input-group" id="offCoilRHGroup">
            <label for="offCoilRH">Off Coil RH:</label>
            <input type="number" id="offCoilRH" step="0.01"> %
        </div>
        
        <div class="input-group" id="offCoilWBGroup" style="display: none;">
            <label for="offCoilWB">Off Coil WB Temp:</label>
            <input type="number" id="offCoilWB" step="0.01"> °C
        </div>
        
        <div class="buttons">
            <button onclick="calculateCoolingCapacity()">Calculate</button>
            <button onclick="reset()">Reset</button>
        </div>
        
        <h2>Results:</h2>
        <div class="results">
            <div class="input-group">
                <label for="sensibleLoad">Sensible Load:</label>
                <input type="text" id="sensibleLoad" readonly> kW
            </div>
            <div class="input-group">
                <label for="totalLoad">Total Load:</label>
                <input type="text" id="totalLoad" readonly> kW
            </div>
            <div class="input-group">
                <label for="shr">SHR:</label>
                <input type="text" id="shr" readonly>
            </div>
            <div class="input-group">
                <label for="airDensity">Air Density:</label>
                <input type="text" id="airDensity" readonly> kg/m³
            </div>
        </div>
    </div>

    <script>
        function toggleOffCoilInput() {
            const param = document.getElementById('offCoilParam').value;
            if (param === 'RH') {
                document.getElementById('offCoilRHGroup').style.display = '';
                document.getElementById('offCoilWBGroup').style.display = 'none';
            } else {
                document.getElementById('offCoilRHGroup').style.display = 'none';
                document.getElementById('offCoilWBGroup').style.display = '';
            }
        }

        function saturationVaporPressure(T) {
            return 610.78 * Math.exp((17.2694 * T) / (T + 238.3));
        }

        function calculateProperties(Tdb, Twb = null, RH = null) {
            const altitude = 10; // Default altitude
            const P0 = 101325; // Standard atmospheric pressure at sea level in Pa
            const L = 0.0065; // Temperature lapse rate in K/m
            const T0 = 288.15; // Standard temperature at sea level in K
            const g = 9.80665; // Gravitational acceleration in m/s^2
            const R = 8.314462618; // Universal gas constant, J/(mol·K)
            const M_air = 0.0289652; // Molar mass of dry air, kg/mol

            // Calculate atmospheric pressure
            const P = P0 * Math.pow((1 - (L * altitude) / T0), (g * M_air) / (R * L));

            // Saturation Vapor Pressure
            const Pws_db = saturationVaporPressure(Tdb);

            let Pw, W;

            if (Twb !== null) {
                const Pws_wb = saturationVaporPressure(Twb);
                Pw = Pws_wb - 0.000666 * P * (Tdb - Twb);
                W = 0.622 * Pw / (P - Pw);
            } else if (RH !== null) {
                Pw = RH * Pws_db;
                W = 0.622 * Pw / (P - Pw);
            }

            // Air density
            const Rd = 287.058; // Specific gas constant for dry air, J/(kg·K)
            const Rv = 461.495; // Specific gas constant for water vapor, J/(kg·K)
            const density = (P / (Rd * (Tdb + 273.15))) * (1 - W * (1 - Rd / Rv));

            // Specific Enthalpy
            const Cpa = 1006; // Specific heat of dry air, J/(kg·K)
            const Cpv = 1860; // Specific heat of water vapor, J/(kg·K)
            const hfg = 2501000; // Latent heat of vaporization at 0°C, J/kg

            const h = Cpa * Tdb + W * (hfg + Cpv * Tdb);

            return {
                h: h / 1000, // Convert J/kg to kJ/kg
                density: density
            };
        }

        function calculateCoolingCapacity() {
            const airFlow = parseFloat(document.getElementById('airFlow').value) / 1000;  // Convert L/s to m³/s
            const onCoilDB = parseFloat(document.getElementById('onCoilDB').value);
            const onCoilWB = parseFloat(document.getElementById('onCoilWB').value);
            const offCoilDB = parseFloat(document.getElementById('offCoilDB').value);

            const param = document.getElementById('offCoilParam').value;
            const offCoilRH = parseFloat(document.getElementById('offCoilRH').value);
            const offCoilWB = parseFloat(document.getElementById('offCoilWB').value);

            if (isNaN(airFlow) || isNaN(onCoilDB) || isNaN(onCoilWB) || isNaN(offCoilDB)) {
                alert("All input fields are required!");
                return;
            }

            const onCoilResults = calculateProperties(onCoilDB, onCoilWB);
            const offCoilResults = param === 'RH' ? calculateProperties(offCoilDB, null, offCoilRH / 100) : calculateProperties(offCoilDB, offCoilWB);

            const h_on_coil = onCoilResults.h;
            const h_off_coil = offCoilResults.h;

            const air_density_on_coil = onCoilResults.density;
            const air_density_off_coil = offCoilResults.density;
            const air_density = (air_density_on_coil + air_density_off_coil) / 2; // Average air density

            const total_load = airFlow * air_density * (h_on_coil - h_off_coil);
            const sensible_load = airFlow * air_density * 1.006 * (onCoilDB - offCoilDB);
            const shr = sensible_load / total_load;

            document.getElementById('sensibleLoad').value = sensible_load.toFixed(2);
            document.getElementById('totalLoad').value = total_load.toFixed(2);
            document.getElementById('shr').value = shr.toFixed(2);
            document.getElementById('airDensity').value = air_density.toFixed(3);
        }

        function reset() {
            const inputs = ['airFlow', 'onCoilDB', 'onCoilWB', 'offCoilDB', 'offCoilRH', 'offCoilWB'];
            inputs.forEach(id => document.getElementById(id).value = '');

            const results = ['sensibleLoad', 'totalLoad', 'shr', 'airDensity'];
            results.forEach(id => document.getElementById(id).value = '');
        }
    </script>
</body>
</html>

