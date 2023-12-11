<script>
    // Placeholder for the server URL
    let HOST = "https://cumulunimbus-web.azurewebsites.net"
 /* HOST="http://localhost:3000" */

    let flipInterval = 15;
    let timeToFlip = flipInterval;

    let valves = [];
    let valveStates = {};
    let interestValve = null;
    let usingInterval = false;



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
                    await waitForAcknowledgement(data.actionId, valveId);
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

    const triggerFlip = async () => {
        console.log('triggering flip', usingInterval, timeToFlip);
        if (usingInterval) {
            if (timeToFlip <= 0) {
                await toggleValve(interestValve);
                timeToFlip = flipInterval;
            } else {
                timeToFlip--;
            }
            setTimeout(triggerFlip, 1000);
        }
    }
</script>

<style>
    main {
        display: flex;
        flex-direction: column;
        align-items: center;
    }

    .valve-control {
        display: flex;
        flex-direction: column;
        align-items: center;
        margin: 1rem;
    }

    .valve-control button {
        margin-top: 1rem;
    }

    .valve-control input {
        margin-bottom: 1rem;
    }

    .valve-control h2 {
        margin-bottom: 0;
    }

    .valve-control h2::after {
        content: '';
        display: block;
        width: 100%;
        height: 1px;
        background-color: #000;
        margin-top: 0.5rem;
    }

    .valve-control h2:last-child::after {
        display: none;
    }

    .valve-control button {
        padding: 0.5rem 1rem;
        border-radius: 0.5rem;
        border: 1px solid #000;
        background-color: #fff;
        cursor: pointer;
    }

    .valve-control button:hover {
        background-color: #000;
        color: #fff;
    }

    .valve-control button:active {
        background-color: #000;
        color: #fff;
    }

    .valve-control button:focus {
        outline: none;
    }

    .valve-control button:disabled {
        background-color: #ccc;
        color: #000;
        cursor: not-allowed;
    }

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
        <h2>Set Flip Interval</h2>
        <input type="text" id="flipInterval" placeholder="Flip Interval" />
        <input type="text" id="interestValve" placeholder="Valve ID" />
        <button on:click={() => {
            flipInterval = document.getElementById('flipInterval').value;
            interestValve = document.getElementById('interestValve').value;
            timeToFlip = flipInterval;
            usingInterval = ! usingInterval;
            triggerFlip();
        }}>
            {usingInterval ? 'Stop' : 'Start'} Interval
        </button>

        <p>{"Flipping every " + flipInterval + " seconds"}</p>

    </div>
</main>
