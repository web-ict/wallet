<script>
    import { fade, fly } from 'svelte/transition';
    import { HUB } from '@web-ict/hub'
    import { ICT } from '@web-ict/ict'
    import { economicCluster } from '@web-ict/ec'
    import { trytes, trytesToTrits } from '@web-ict/converter'
    import { ISS } from '@web-ict/iss'
    import { ADDRESS_LENGTH } from '@web-ict/transaction'
    
    export let name;

    let ict
    let cluster
    let hub
    let seed
    let seedTrits = new Int8Array(243)
    let interval
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
    let info = {}
    let numberOfPeers = 0
    let online

    import('@web-ict/curl').then(({ Curl729_27 }) => {
        ict = ICT({
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
        ict.launch()

        const step = () => {
            info = ict.info()
            numberOfPeers = info.peers.filter(peer => peer.uptime > 0).length
            online = numberOfPeers > 0
            requestAnimationFrame(step)
        }
        requestAnimationFrame(step)
    })

    function init() {
        import('@web-ict/curl').then(({ Curl729_27 }) => {
            cluster = economicCluster({
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

            const iss = ISS(Curl729_27)
            const persistenceId = trytes(iss.addressFromDigests(iss.digests(iss.key(iss.subseed(seedTrits, 0), 2))), 0, ADDRESS_LENGTH)

            hub = HUB({
                seed: seedTrits,
                security: 2,
                persistencePath: './',
                persistenceId,
                reattachIntervalDuration: 30 * 1000,
                acceptanceThreshold: 100,
                Curl729_27,
                ixi: ict.ixi,
            })

            cluster.launch()
            hub.launch()

            interval = setInterval(async () => {
                transfers = []
                const { getTransfers, getBalance } = await hub
                getTransfers().forEach(transfer => {
                    if (transfer.transactionObjects) {
                        transfers.push(transfer)
                    }
                })
                transfers = transfers

                balance = formatValueWithHumanReadableUnit(getBalance().toString())
            }, 1000)

            loggedIn = true
        })   
    }

    function logout () {
        seed = undefined
        seedTrits = new Int8Array(243)
        loggedIn = false
        clearInterval(interval)
        transfers = []
        transfers = transfers
        cluster.terminate()
        hub.terminate()
    }

    let seed0 = seed
    function updateSeedChecksum (event) {
        if (!new RegExp('^[9A-Z]{1,}$').test(seed)) {
            seed = seed0
            return
        }
        seed0 = seed
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
        units.set('Ti', 1_000_000_000_000)
        units.set('Pi', 1_000_000_000_000_000)

        return value * units.get(unit)
    }

    function formatValueWithHumanReadableUnit(value) {
        const units = ['i', 'Ki', 'Mi', 'Gi', 'Ti', 'Pi']
        let i = 0
        while (value >= 1000) {
            value = value / 1000
            i++
        }
        return `${value}${units[i]}`
    }

    let depositDialogVisible = false
    let withdrawDialogVisible = false
</script>

<header>
    <div>
    <img id="logo" src="/logo.png">
    </div>
    {#if loggedIn}
        <div id="actions">
            <button id="receive-button" class="button" on:click={() => (depositDialogVisible = true)}>Receive</button>
            <button id="send-button" class="button" on:click={() => (withdrawDialogVisible = true)}>Send</button>
            <div id="balance">
                <div>Balance:</div>
                <div id="balance-value">{balance}</div>
            </div>
            <div id="info">
                <div id="peers"><span id="online-{online}"></span> {numberOfPeers}/3 peers</div>
            </div>
            <div id="secondary-actions">
                <button class="button-2">Cluster settings</button>
                <button class="button-2" on:click={logout}>Logout</button>
            </div>
        </div>
    {:else}
        <div id="info">
            <div id="peers"><span id="online-{online}"></span> {numberOfPeers}/3 peers</div>
        </div>
    {/if}
</header>
<main>
    {#if !loggedIn}
        <form>
            <label for="seed">Seed</label>
            <input name="seed" type="password" size="81" autofocus bind:value={seed} on:input={updateSeedChecksum}>
            <span id="seed-checksum">{seedChecksum}</span>
            <input type="submit" class="button" disabled='{seed === undefined}' on:click={init} value="Login">
        </form>
    {/if}
    {#each transfers as transfer}
		<div class="transfer card">
            {#if transfer.attachments !== undefined}
                {transfer.attachments[transfer.attachments.length -1]}
            {/if}
            {#each transfer.transactionObjects.filter(({value}) => !value.equals(0)) as transaction}
                <div class="transaction">
                    <span class="transaction-address">
                        {transaction.address}
                    </span>
                    <span class="transaction-value">
                      {(transaction.value.greater(0) ? '+' : '-')}{formatValueWithHumanReadableUnit(transaction.value.abs())}
                    </span>
                </div>
            {/each}
        </div>
        <br>
	{/each}

    {#if depositDialogVisible}
        <div class="dialog" transition:fade on:click={resetDepositDialog}>
            <div class="card" transition:fly="{{ y: 100, duration: 300 }}" on:click={e => e.stopPropagation()}>
                <h2>Receive</h2>
                <div>
                    {#if depositStep === 0}
                        <form>
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
                            <input type="submit" class="button" disabled='{depositValue === 0}' on:click={deposit} value="Next ❯">
                        </form>
                    {:else}
                        <div transition:fly="{{ y: 30, duration: 300 }}">
                            <p>Address should be used only once.</p>
                            <p id="deposit-address">{depositAddress}</p>
                            <button class="button" on:click={resetDepositDialog}>Close</button>
                            <button class="button" on:click={copyDepositAddress}>Copy</button>
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
                        <form>
                            <label for="withdrawal-address">Address:</label>
                            <input name="withdrawal-address" size="81" autocomplete="off" bind:value={withdrawalAddress} type="text">
                            <button class="button" disabled='{withdrawalAddress === ''}' on:click={getWithdrawalValue}>Next ❯</button>
                        </form>
                    </div>
                {:else if withdrawalError !== undefined}
                    <p>{withdrawalError}</p>
                    <button class="button" on:click={resetWithdrawDialog}>Close</button>
                {:else if withdrawalValue !== undefined}
                    <p>About to send {formatValueWithHumanReadableUnit(withdrawalValue)} to:</p>
                    <p>{withdrawalAddress}</p>
                    <button class="button" on:click={resetWithdrawDialog}>Cancel</button>
                    <button class="button" on:click={withdraw}>Send ❯</button>
                {:else}
                    <p>Transfer has not been initiated.</p>
                    <button class="button" on:click={resetWithdrawDialog}>Close</button>
                {/if}
            </div>
        </div>
    {/if}
</main>

<style>
    header {
        width: 100%;
        padding: 0 2.5em;
        height: 5em;
        box-sizing: border-box;
        position: fixed;
        top: 0;
        left: 0;
        background: #000;
        display: flex;
        justify-content: space-between;
        align-items: center;
        z-index: 1000;
    }
    
    main {
        padding: 1em 2em;
        width: 100%;
        margin: 6em auto 0 auto;
        position: relative;
        top: 60px;
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
        height: 3em;
        filter: invert();
        align-self: center;
    }

    #balance {
        font-size: 1em;
        align-self: center;
        padding: 0 1em;
        flex-direction: column;
    }

    #balance-value {
        color: #fff;
        font-size: 1.5em;
    }

    #actions {
        display: flex;
        justify-content: space-between;
    }

    #secondary-actions {
        display: flex;
        align-items: center;
    }

    #info {
        font-size: 1em;
        align-self: center;
        flex-direction: column;
    }

    #peers {
        padding-right: 1em;
    }

    #online-false {
        display: inline-block;
        width: 10px;
        height: 10px;
        background: #555;
        border-radius: 10px;
        transition: background-color 1s;
    }

    #online-true {
        display: inline-block;
        width: 10px;
        height: 10px;
        background: rgb(21, 202, 27);
        border-radius: 10px;
        transition: background-color 1s;
    }

    #receive-button {
        margin-right: 0.5em;
    }

    .button {
        padding: 0.3em 0.8em;
        font-size: 1.5em;
        color: #fff;
        border: 0;
        border-radius: 1.2em;
        background: transparent;
        cursor: pointer;
        transition: background-color 0.3s ease, color 0.3s ease;
    }

    .button:hover {
        color: #000;
        background: #fff;
    }

    .button-2 {
        background: transparent;
        border: 0;
        color: rgba(255,255,255,.7)
    }

    .button-2:hover {
        text-decoration: underline;
    }

    .card {
        background: #222;
        box-shadow: 0px 0px 20px rgba(0,0,0,.5);
        border-radius: 10px;
        padding: 2em;
        margin: 0;
        box-sizing: border-box;
    }

    h2 {
        color: #fff;
        font-size: 2em;
        font-weight: 300;
        margin: 0;
        padding-bottom: 1em;
    }

    input, select {
        padding: 0.6em 0em;
        margin: 0.2em;
        font-size: 1.2em;
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

    .transfer {
        font-size: 1.2em;
    }

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}
</style>
