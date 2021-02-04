<script>
    import { fade, fly } from 'svelte/transition';
    import { HUB } from '@web-ict/hub'
    import { ICT } from '@web-ict/ict'
    import { economicCluster } from '@web-ict/ec'
    import { trytes, trytesToTrits } from '@web-ict/converter'
    import { ISS } from '@web-ict/iss'
    import { ADDRESS_LENGTH } from '@web-ict/transaction'
    
    export let name;

    let hub
    let seed
    let seedTrits = new Int8Array(243)
    let transfers = []
    let balance = '0 i'
    let loggedIn = false
    let seedChecksum = ''
    let depositStep = 0
    let withdrawStep = 0
    let depositValue = 0
    let depositUnit
    let depositAddress = '9'.repeat(81) // TODO: add checksum
    let withdrawalAddress = ''
    let withdrawalValue = undefined
    let withdrawalError = undefined

    function init() {
        hub = import('@web-ict/curl').then(({ Curl729_27 }) => {
            const ict = ICT({
                autopeering: {
                    signalingServers: ['ws://116.203.136.136:8080', 'ws://116.203.136.136:8081', 'ws://116.203.136.136:8082'],
                    iceServers: [
                        {
                            urls: ['stun:stun3.l.google.com:19302'],
                        },
                    ],
                    cooldownDuration:  10, // s
                    tiebreakerIntervalDuration:  10, // s
                    tiebreakerValue: Number.POSITIVE_INFINITY, // New transactions / second
                },
                dissemination: {
                    A: 1, // ms
                    B: 100,
                },
                subtangle: {
                    capacity: 1_000_000, // In transactions
                    pruningScale: 0.1, // In proportion to capacity
                    artificialLatency: 100, // ms
                },
                Curl729_27,
            })

            const cluster = economicCluster({
                intervalDuration: 5 * 1000,
                ixi: ict.ixi,
                Curl729_27,
            })

            cluster.addEconomicActor({
                address: 'XTTLDRSNRPBPGESXAVBKKS9PMNCY9ZRTIZOZJUIYMGEBFRCQPOHCCONRX9JMBPCSYYKTOYIIEVYGGCTMR',
                depth: 12,
                security: 1,
                weight: 1,
            })

    
            trytesToTrits(seed, seedTrits, 0, 243)

            const iss = ISS(Curl729_27)
            const persistenceIdTrits = iss.addressFromDigests(iss.digests(iss.key(iss.subseed(seedTrits, 0), 2)))

            hub = HUB({
                seed: seedTrits,
                security: 2,
                persistencePath: './',
                persistenceId: trytes(persistenceIdTrits, 0, persistenceIdTrits.length),
                reattachIntervalDuration: 30 * 1000,
                acceptanceThreshold: 100,
                Curl729_27,
                ixi: ict.ixi,
            })

            ict.launch()
            cluster.launch()
            hub.launch()

            const interval = setInterval(async () => {
                transfers = []
                const { getTransfers, getBalance } = await hub
                getTransfers().forEach(transfer => {
                    if (transfer.input) {
                        transfers.push({
                            address: trytes(transfer.input.address, 0, ADDRESS_LENGTH),
                            value: transfer.input.balance,
                        })
                    }
                })
                transfers = transfers

                balance = formatValueWithHumanReadableUnit(getBalance().toString())
            }, 1000)

            loggedIn = true

            return hub
        })   
    }

    function updateSeedChecksum () {
        import('@web-ict/curl').then(({ Curl729_27 }) => {
            const hash = new Int8Array(243)
            trytesToTrits(seed, seedTrits, 0, 243)
            Curl729_27.get_digest(seedTrits, 0, 243, hash, 0)
            seedChecksum = trytes(hash, 0, 243).slice(-3)
        })
    }

    async function deposit() {
        depositStep = 1
        const { deposit } = await hub
        depositAddress = await deposit({
             value: calculateValueGivenUnit(depositValue, depositUnit)
        })
    }

    function resetDepositDialog() {
        depositDialogVisible = false
        depositStep = 0
        depositValue = 0
        depositAddress = '9'.repeat(81)
    }

    function copyDepositAddress() {
        navigator.clipboard.writeText(depositAddress)
    }

    async function getWithdrawalValue() {
        withdrawStep = 1
        const { getWithdrawalValue } = await hub
        withdrawalValue = await getWithdrawalValue(withdrawalAddress)
    }

    async function withdraw() {
        const { withdraw } = await hub
        try {
            await withdraw({ address: withdrawalAddress, value: withdrawalValue })
            resetWithdrawDialog()
        } catch (error) {
            console.log(error)
            withdrawalError = error.message
        }
    }

    function resetWithdrawDialog() {
        withdrawDialogVisible = false
        withdrawStep = 0
        withdrawalAddress = ''
        withdrawalValue = undefined
        withdrawalError = undefined
    }

    function calculateValueGivenUnit(value, unit) {
        const units = new Map()
        units.set('i', 1)
        units.set('Ki', 1_000)
        units.set('Mi', 1_000_000)
        units.set('Gi', 1_000_000_000)
        units.set('Ti', 1_000_000_000_0000)
        units.set('Pi', 1_000_000_000_0000_000)

        return value * units.get(unit)
    }

    function formatValueWithHumanReadableUnit(value) {
        const units = ['i', 'Ki', 'Mi', 'Gi', 'Ti', 'Pi']
        let i = 0
        while (value >= 1000) {
            value = value / 1000
            i++
        }
        return `${value} ${units[i]}`
    }

    let depositDialogVisible = false
    let withdrawDialogVisible = false
</script>

<header>
    <img id="logo" src="/logo.png">
    {#if loggedIn}
        <div id="balance">Balance: {balance}</div>
        <div id="actions">
            <button id="receive-button" on:click={() => (depositDialogVisible = true)}>Receive</button>
            <button id="send-button" on:click={() => (withdrawDialogVisible = true)}>Send</button>
        </div>
    {/if}
</header>
<main>
    {#if !loggedIn}
        <label for="seed">Seed</label>
        <input name="seed" type="password" bind:value={seed} on:input={updateSeedChecksum}>
        <span id="seed-checksum">{seedChecksum}</span>
        <button disabled='{seed === undefined}' on:click={init}>Login</button>
    {/if}
    {#each transfers as transfer}
		<div class="transfer">
			{transfer.address}, {transfer.value}
        </div>
	{/each}

    {#if depositDialogVisible}
        <div class="dialog" transition:fade on:click={resetDepositDialog}>
            <div class="card" transition:fly="{{ y: 100, duration: 300 }}" on:click={e => e.stopPropagation()}>
                <h2>Receive</h2>
                <div>
                    {#if depositStep === 0}
                        <label for="deposit-value">Value:</label>
                        <input name="deposit-value" bind:value={depositValue} type="number" min="0">
                        <select name="deposit-unit" bind:value={depositUnit} class="unit">
                            <option value="i">i</option>
                            <option value="Ki">Ki</option>
                            <option value="Mi">Mi</option>
                            <option value="Gi">Gi</option>
                            <option value="Ti">Ti</option>
                            <option value="Pi">Pi</option>
                        </select>
                        <button disabled='{depositValue === 0}' on:click={deposit}>Next ❯</button>
                    {:else}
                        <div transition:fly="{{ y: 30, duration: 300 }}">
                            <p>Address should be used only once.</p>
                            <p id="deposit-address">{depositAddress}</p>
                            <button on:click={resetDepositDialog}>Close</button>
                            <button on:click={copyDepositAddress}>Copy</button>
                        </div>
                    {/if}
                </div>
            </div>
        </div>
    {/if}
    {#if withdrawDialogVisible}
        <div class="dialog" transition:fade on:click={resetWithdrawDialog}>
            <div class="card" transition:fly="{{ y: 100, duration: 300 }}" on:click={e => e.stopPropagation()}>
                <h2>Send</h2>
                {#if withdrawStep === 0}
                    <div id="withdrawal-step-1">
                        <label for="withdrawal-address">Address:</label>
                        <input name="withdrawal-address" id="withdrawal-address" bind:value={withdrawalAddress} type="text">
                        <button disabled='{withdrawalAddress === ''}' on:click={getWithdrawalValue}>Next ❯</button>
                    </div>
                {:else}
                    <div id="withdrawal-step-2">
                        {#if withdrawalError !== undefined}
                            <p>{withdrawalError}</p>
                            <button on:click={resetWithdrawDialog}>Close</button>
                        {:else if withdrawalValue !== undefined}
                            <p>About to send {formatValueWithHumanReadableUnit(withdrawalValue)} to:</p>
                            <p>{withdrawalAddress}</p>
                            <button on:click={resetWithdrawDialog}>Cancel</button>
                            <button on:click={withdraw}>Send ❯</button>
                        {:else}
                            <p>Transfer has not been initiated.</p>
                            <button on:click={resetWithdrawDialog}>Close</button>
                        {/if}

                    </div>
                {/if}
            </div>
        </div>
    {/if}
</main>

<style>
    header {
        width: 100%;
        height: 2.5em;
        padding: 1em 0em;
        position: fixed;
        top: 0;
        left: 0;
        background: #000;
        display: flex;
        justify-content: space-between;
    }
    
    main {
        padding: 1em 2em;
        width: 100%;
        margin: 2em auto 0 auto;
        position: relative;
        top: 60px;
        font-family: monospace;
        box-sizing: border-box;
    }

    main p, label {
        font-size: 1.2em;
    }

    #deposit-address {
        padding: 20px;
        border-radius: 20px;
        background: rgba(0,0,0,0.3)
    }

    #logo {
        height: 2.5em;
        padding-left: 2em;
        filter: invert();
    }

    #balance {
        font-size: 1.5em;
        padding-top: 0.5em;
        color: #fff;
        font-family: monospace;   
    }

    #actions {
        padding-right: 2em;
    }

    #receive-button {
        margin-right: 0.5em;
    }

    button {
        padding: 0.6em 1em;
        font-size: 1.5em;
        font-family: monospace;
        color: #fff;
        border: 0;
        border-radius: 1.2em;
        background: transparent;
        cursor: pointer;
        transition: background-color 0.3s ease, color 0.3s ease;
    }


    button:hover, button:focus {
        color: #000;
        background: #fff;
    }

    .card {
        background: #222;
        box-shadow: 0px 0px 20px rgba(0,0,0,.5);
        border: 0px solid #efefef;
        border-radius: 10px;
        padding: 2em;
        box-sizing: border-box;
        max-width: 90%;
    }

    h2 {
        color: #fff;
        font-size: 2em;
        font-weight: 300;
        margin: 0;
        padding-bottom: 1em;
        font-family: monospace;
    }

    input, select {
        padding: 0.6em 0em;
        margin: 0.2em;
        font-family: monospace;
        font-size: 1.5em;
        border: 0;
        border-bottom: 2px solid rgba(255,255,255,0.5);
        background: transparent;
    }

    select:focus {
        outline: 0;
    }

    input:focus {
        outline: none;
    }

    #withdrawal-address {
        width: 800px;
    }

    .dialog {
        position: fixed;
        display: flex;
        justify-content: center;
        align-items: center;
        top: 0;
        left: 0;
        height: 100%;
        width: 100%;
        background: rgba(10,10,10,0.9)
    }

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>
