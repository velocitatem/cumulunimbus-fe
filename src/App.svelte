<script>
    // Placeholder for the server URL
    let HOST = "https://cumulunimbus-web.azurewebsites.net"
    //HOST="http://localhost:3000"

    let flipInterval = 15;
    let timeToFlip = flipInterval;

    let valves = [];
    let valveStates = {};

    setInterval(() => {
        if (timeToFlip > 0) timeToFlip--;
        else{
            //alert("Flipping all valves");
            timeToFlip = flipInterval;
            for(let i = 0; i < valves.length; i++){
                toggleValve(i);
            }
        }
    }, 1000);

 const getDevices = async () => {
        try {
            const response = await fetch(`${HOST}/devices`);
            if (response.ok) {
                const data = await response.json();
                return data;
            }
        } catch (error) {
            console.error('Error getting devices:', error);
        }
    }

 document.addEventListener('DOMContentLoaded', async () => {
        const devices = await getDevices();
        valves = devices.map(device => device);
        valveStates = {};
        valves.forEach(valve => {
            valveStates[valve] = false;
        });
    });

    async function toggleValve(valveId) {
        try {
            const response = await fetch(`${HOST}/open/${valveId}`, {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json'
                },
                body: JSON.stringify({ state: !valveStates[valveId] })
            });

            if (response.ok) {
                const data = await response.json();
                if (data.actionId) {
                    waitForAcknowledgement(data.actionId, valveId);
                }
            }
        } catch (error) {
            console.error('Error toggling valve:', error);
        }
    }

    async function waitForAcknowledgement(actionId, valveId,count=0) {
        try {
            const response = await fetch(`${HOST}/acknowledged/${actionId}`);
            if (response.ok) {
                const data = await response.json();
                if (count > 10) {
                    console.error('No acknowledgement received');
                    alert('No acknowledgement received');
                    return;
                }
                if (data.acknowledged) {
                    valveStates[valveId] = !valveStates[valveId];
                } else {
                    setTimeout(() => {
                        waitForAcknowledgement(actionId, valveId,count+1);
                    }, 1000);
                }
            }
        } catch (error) {
            console.error('Error waiting for acknowledgement:', error);
        }
    }

 const addValve = () => {
        const valveId = document.getElementById('valveId').value;
     // send to /devices/:id
        fetch(`${HOST}/devices/${valveId}`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ state: false })
        })
            .then(response => response.json())
            .then(data => {
                valves.push(valveId);
                valveStates[valveId] = false;
            })
            .catch(error => console.error('Error adding valve:', error));
    }
</script>

<style>
    /* Add your CSS styles here */
</style>

<main>
    <h1>Welcome to the IoT Valve Controller</h1>
    {#each valves as valve}
        <div class="valve-control">
            <h2>{valve} - Status: {valveStates[valve] ? 'OPEN' : 'CLOSED'}</h2>
            <button on:click={() => toggleValve(valve)}>
                Toggle {valve}
            </button>
        </div>
    {/each}
    <div class="valve-control">
        <h2>Add a new valve</h2>
        <input type="text" id="valveId" placeholder="Valve ID" />
        <button on:click={addValve}>
            Add Valve
        </button>
    </div>
    <div>
        <p><br><br><br><br></p>
        <p>Valves will toggle in {timeToFlip} seconds.</p>
    </div>
</main>
