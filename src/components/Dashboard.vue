<template>
  <div class="hello">
    <div class="row">
      <div class="col-12">
        <h1>
          <b class="float-right text-warning">{{ ledger }}</b>
          rippled-ws-client-pool
        </h1>
      </div>
      <div class="col-12 margin-bottom-10">
        <ul class="nav nav-tabs nav-fill">
          <li class="nav-item" v-for="(m, k) in menu" v-bind:key="k"><a :href="'#' + k" @click="view = k" class="nav-link" :class="{ active: view === k }">{{ m }}</a></li>
        </ul>
      </div>

      <!--
        /************************************
         **        CONNECTION DETAILS
         ************************************/
      -->

      <div class="col-3" v-if="view === 'c'">
        <ul class="list-group list-sm list-group-flush">
          <li class="list-group-item text-center" v-if="servers.length < 1">
            <span class="text-danger"><b>No servers</b></span>
          </li>
          <li class="list-group-item" v-for="server in servers" v-bind:key="server">
            <a @click="removeServer(server)" class="text-danger float-right rm">×</a>
            <b><code class="text-secondary"><span v-html="serverStateIcon(server)"></span> {{ server }}</code></b>
          </li>
        </ul>
        <br />
        <input ref="addserver" type="text" placeholder="Add server (hostname)" class="form-control" v-on:keydown.enter="addServer" v-model="addHostname" />
      </div>
      <div v-for="(c) in connections" v-bind:key="c.hostname" class="col-3" v-if="view === 'c'">
        <b>{{ c.hostname }}</b>
        <div class="alert" :class="{ 'alert-warning' : !c.state.online, 'alert-primary' : c.state.online }"><pre>{{ c.state }}</pre></div>
      </div>

      <!--
        /************************************
         **        HEALTH RANKING TABLE
         ************************************/
      -->

      <div class="col-12" v-if="view === 'h'">
        <p class="alert alert-primary">
          Server health ranking: the value in cells are the points by ranking. The smaller red value is the metric value.
          A green background marks the #1 position (might be shared).
        </p>
        <table class="table table-hover table-sm">
          <thead>
            <tr>
              <th></th>
              <th></th>
              <th></th>
              <th v-for="(x, t) in ranking.data.source" v-bind:key="t" class="text-center"><small><b>{{ t }}</b></small></th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(s, i) in ranking.data.ranking" v-bind:key="s">
              <th width="30">#{{ i + 1 }}</th>
              <th width="30" v-html="serverStateIcon(s)"></th>
              <th width="300">{{ s }}</th>
              <td width="90" class="text-center" v-for="(x, t) in ranking.data.source" v-bind:key="t" :class="{ 'alert-success': ranking.data.source[t].order ? ranking.data.source[t].values[s] === ranking.data.source[t].values[ranking.data.source[t].order[0]] : false }">
                <b>{{ ranking.data.source[t].points[s] }}</b>
                <br />
                <small><code>{{ ranking.data.source[t].values[s] }}</code></small>
              </td>
              <th><h2 class="health-points alert-primary">{{ ranking.data.points[s] }}</h2></th>
            </tr>
          </tbody>
          <tbody>
            <tr v-for="s in offlineServers" v-bind:key="s">
              <th width="30"></th>
              <th width="30" v-html="serverStateIcon(s)"></th>
              <th width="250">{{ s }}</th>
              <td width="90" class="text-center" v-for="(x, t) in ranking.data.source" v-bind:key="t"></td>
              <th></th>
            </tr>
          </tbody>
        </table>
      </div>

      <!--
        /************************************
         **           RAW COMMAND
         ************************************/
      -->

      <div class="col-4" v-if="view === 'r'">
        <codemirror v-model="rawCommand.json" :options="rawCommand.options"></codemirror>
        <br />
        <h5><b>Samples</b></h5>
        <button @click="rawCommand.json=rawCommand.samples[key]" class="btn btn-sm btn-block" :class="{ 'btn-outline-secondary': rawCommand.samples[key] !== rawCommand.json, 'btn-primary': rawCommand.samples[key] === rawCommand.json }" v-for="key in Object.keys(rawCommand.samples)" v-bind:key="key">{{ key }}</button>
      </div>
      <div class="col-2 text-center" v-if="view === 'r'">
        <button @click="sendRawCommand" class="btn btn-md btn-success">
          <b>⇨ Execute</b>
        </button>
      </div>
      <div class="col-6" v-if="view === 'r'">
        <div v-if="rawCommand.response.state === 'response'">
          <table class="table-sm table table-bordered">
            <tbody>
              <tr>
                <td width="140">Response from</td>
                <td colspan="3" class="text-primary"><b>{{ rawCommand.response.data.server }}</b></td>
              </tr>
              <tr>
                <td>Requests sent</td>
                <td colspan="3" class="text-primary">
                  <b>{{ rawCommand.response.data.requestsSent }}</b>
                  (req. #{{ rawCommand.response.data.waterfallSeq + 1 }} succeeded)
                </td>
              </tr>
              <tr v-for="(failure, key) in rawCommand.response.data.failures" v-bind:key="failure.server">
                <td v-if="key === 0" :rowspan="rawCommand.response.data.failures.length">Failures</td>
                <td><code>{{ failure.server }}</code></td>
                <td class="text-primary">{{ failure.error }}</td>
                <td class="text-primary">{{ failure.type }}</td>
              </tr>
            </tbody>
          </table>
        </div>
        <div class="jsonResponse alert" :class="{ 'alert-secondary': rawCommand.response.state === null, 'alert-warning': rawCommand.response.state === 'await', 'alert-danger': rawCommand.response.state === 'error', 'success': rawCommand.response.state === 'response' }">
          <vue-json-pretty :data="rawCommand.response.body"></vue-json-pretty>
        </div>
      </div>

      <!--
        /************************************
         **        WATCH TRANSACTIONS
         ************************************/
      -->

      <div class="col-3" v-if="view === 't'">
        <ul class="list-group list-sm list-group-flush">
          <li class="list-group-item text-center" v-if="accounts.length < 1">
            <span class="text-danger"><b>No accounts watched</b></span>
          </li>
          <li class="list-group-item" v-for="account in accounts" v-bind:key="account">
            <a @click="removeAccount(account)" class="text-danger float-right rm">×</a>
            <b><code class="text-secondary">{{ account }}</code></b>
          </li>
        </ul>
        <br />
        <input ref="addaccount" type="text" placeholder="Add account (rXXX...)" class="form-control" v-on:keydown.enter="addAccount" v-model="newAccount" />
        <br />
        <a :class="{ 'd-block': true }" v-for="a in interestingAccounts" v-bind:key="a" v-if="accounts.indexOf(a) < 0" @click="newAccount=a;addAccount()"><small><code>{{ a }}</code></small></a>
        <hr />
        <div class="alert alert-warning text-center">
          <div class="form-check">
            <input type="checkbox" class="form-check-input" v-model="showOnlyPayments" id="showOnlyPayments" />
            <label for="showOnlyPayments" class="form-check-label">Show only "Payments"</label>
          </div>
        </div>
      </div>
      <div class="col-9" v-if="view === 't'">
        <div v-if="filteredTransactions.length < 1" class="alert alert-primary text-center">
          No transactions (yet)
          <span v-if="showOnlyPayments">
             - but only <b>payments</b> will be displayed
            <a @click="showOnlyPayments = !showOnlyPayments"><b><u>disable filter</u></b></a>
          </span>
        </div>
        <table class="table table-sm table-tx table-hover table-bordered" v-if="filteredTransactions.length > 0">
          <thead>
            <tr>
              <th width="50"></th>
              <th width="100">Tx Type</th>
              <th width="100">Tx Status</th>
              <th>Tx Hash @ Ledger</th>
              <th width="300">Account (first responder)</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="tx in filteredTransactions" v-bind:key="tx.Hash">
              <td valign="top" class="text-right">{{ tx.__increment }}</td>
              <td valign="top">
                <b>{{ tx.Data.transaction.TransactionType }}</b><br />
              </td>
              <td valign="top" :class="{ complete: (Math.min(serverCount, tx.SeenBy.length) / serverCount) === 1 }">
                <code>{{ tx.Data.engine_result }}</code>
                <div class="seenByPct" :style="'width: ' + (tx.SeenBy.length / serverCount * 100) + '%;'"></div>
              </td>
              <td valign="top">
                <code class="text-secondary">{{ tx.Hash }}</code> @ <b>{{ tx.Data.ledger_index }}</b><br />
                <b class="text-right amount text-info" v-if="tx.Data.transaction.Amount && typeof tx.Data.transaction.Amount === 'string'">
                  {{ parseFloat(tx.Data.transaction.Amount) / 1000000 }} XRP
                </b>
                <span v-if="tx.Data.transaction.Destination">⇨ <code class="text-info">{{ tx.Data.transaction.Destination }}</code></span>
              </td>
              <td valign="top">
                <code class="text-primary"><b>{{ tx.Data.transaction.Account }}</b></code><br />
                <code class="text-secondary">{{ tx.FirstResponder }}</code>
              </td>
              <!-- <td><pre v-if="tx.Data.transaction.TransactionType === 'Payment'">{{ tx.Data.transaction }}</pre></td> -->
            </tr>
          </tbody>
        </table>
      </div>

      <!--
        /************************************
         **        RETRIEVE TRANSACTION
         ************************************/
      -->

      <div class="col-4" v-if="view === 'x'">
        <codemirror v-model="accountTransactions.json" :options="rawCommand.options"></codemirror>
      </div>
      <div class="col-2 text-center" v-if="view === 'x'">
        <button @click="getTxs(false)" class="btn btn-md btn-success">
          <b>⇨ Execute</b>
        </button>
      </div>
      <div class="col-6" v-if="view === 'x'">
        <p class="alert alert-danger text-center" v-if="accountTransactions.data.error !== ''">
          {{ accountTransactions.data.error }}
        </p>
        <p class="alert alert-warning text-center" v-if="accountTransactions.data.error === '' && accountTransactions.data.account === ''">
          Awaiting execution...
        </p>
        <div v-if="accountTransactions.data.error === '' && accountTransactions.data.account !== ''">
          <h5><b>Transactions:</b> {{ accountTransactions.data.account }}</h5>

          <div v-if="typeof accountTransactions.data.lastResponse === 'object'">
            Data from: <code><b class="text-primary">{{ accountTransactions.data.lastResponse.server.host }}</b></code> in <b>{{ accountTransactions.data.lastResponse.replyMs }}ms</b> <small class="text-muted">(preferred order: {{ accountTransactions.data.lastResponse.server.preferenceIndex + 1 }})</small>
          </div>
          <hr />
          <table class="table table-sm table-hover">
            <thead>
              <tr>
                <th>Ledger</th>
                <th width="100">Type</th>
                <th>Counterparty</th>
                <th>Amount</th>
                <th>Date</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="tx in accountTransactions.data.records" v-bind:key="tx.tx.hash">
                <td>{{ tx.tx.inLedger }}</td>
                <td>{{ tx.tx.TransactionType }}</td>
                <td><code><small>{{ tx.tx.Account === accountTransactions.data.account ? tx.tx.Destination : tx.tx.Account }}</small></code></td>
                <td class="text-right">{{ isNaN(parseFloat(tx.tx.Amount)) ? '' : (parseFloat(tx.tx.Amount) / 1000000).toFixed(6) }}</td>
                <td>{{ new Date((tx.tx.date + 946684800) * 1000).toISOString().split('.')[0].replace('T', ' ') }}</td>
              </tr>
            </tbody>
          </table>
          <button v-if="typeof accountTransactions.data.lastResponse.more === 'function'" @click="getTxs(true)" class="btn btn-primary btn-block">+ Load more</button>
          <br />&nbsp;
        </div>
      </div>

      <!--
        /************************************
         **        SUBMIT TRANSACTION
         ************************************/
      -->

      <div class="col-4" v-if="view === 's'">
        <div class="input-group">
          <input ref="secret" v-on:focus="txSecretFocussed=true" v-on:blur="txSecretFocussed=false" :type="txSecretFocussed ? 'text' : 'password'" placeholder="Secret (for local signing), eg. sXXX...XXX" class="form-control" v-model="sendTransaction.secret" />
          <div class="input-group-append">
            <button @click="sendTransaction.extraSecrets.push({ focussed: false, secret: '' })" class="btn btn-outline-primary" id="addSecret" type="button">
              + <span>Multisign</span>
            </button>
          </div>
        </div>
        <div style="margin-top: 4px;" class="input-group" v-for="(s, i) in sendTransaction.extraSecrets" v-bind:key="i">
          <input v-on:focus="s.focussed=true" v-on:blur="s.focussed=false" :type="s.focussed ? 'text' : 'password'" placeholder="Secret (for local signing), eg. sXXX...XXX" class="form-control" v-model="s.secret" />
          <div class="input-group-append">
            <button @click="sendTransaction.extraSecrets.splice(i, 1)" class="btn btn-outline-danger" type="button">
              &times;
            </button>
          </div>
        </div>
        <div class="text-center" style="color: rgb(209,22,61); line-height: .9em; padding: 8px 6px 12px 6px;"><small><b>Secrets will <u>NOT</u> be sent over the internet, since transaction signing will happen locally (client side) 🎉</b></small></div>
        <codemirror v-model="sendTransaction.json" :options="rawCommand.options"></codemirror>
        <br />
        <ul class="text-muted">
          <li><small>Enter the sending wallet address at <code>Account</code>. Of course the Secret should be allowed to sign transactions for this account.</small></li>
          <li><small>If you don't enter a <code>Sequence</code>, <a href="https://github.com/WietseWind/rippled-ws-client-sign" target="_blank">rippled-ws-client-sign</a> will auto-lookup using an account_info request.</small></li>
          <li><small>Enter the <code>Amount</code> and <code>Fee</code> in drops (<code>1.000.000</code> drops = <code>1</code> XRP) </small></li>
        </ul>
      </div>
      <div class="col-2 text-center" v-if="view === 's'">
        <button :disabled="sendTransaction.busy" @click="sendTx" class="btn btn-md btn-success">
          <b>⇨ Execute</b>
        </button>
      </div>
      <div class="col-6" v-if="view === 's'">
        <p class="alert alert-warning text-center" v-if="sendTransaction.response === null && !sendTransaction.busy && sendTransaction.error === ''">
          Awaiting execution...
        </p>
        <div v-if="sendTransaction.error !== ''">
          <div class="alert alert-danger text-center">
            <b>Error: </b> {{ sendTransaction.error }}
          </div>
          <div class="error" v-if="sendTransaction.errorDetails !== {}" >
            <h5><b>Error details:</b></h5>
            <vue-json-pretty :data="sendTransaction.errorDetails"></vue-json-pretty>
          </div>
        </div>
        <p class="alert alert-success text-center" v-if="sendTransaction.response !== null && sendTransaction.response.hash">
          Transaction succeeded! <b><a :href="'https://xrpcharts.ripple.com/#/transactions/' + sendTransaction.response.hash" target="_blank">View transaction on xrpcharts.ripple.com</a></b>
        </p>
        <p class="alert alert-primary text-center" v-if="sendTransaction.response === null && sendTransaction.busy">
          Processing transaction...
        </p>
        <vue-json-pretty v-if="sendTransaction.response !== null && !sendTransaction.busy" :data="sendTransaction.response"></vue-json-pretty>
        <br />&nbsp;
      </div>
    </div>
  </div>
</template>

<script>
// import RippledWsClientPool from './npm-module/rippled-ws-client-pool.js'
// import RippledWsClientSign from './npm-module/rippled-ws-client-sign.js'
import RippledWsClientPool from 'rippled-ws-client-pool'
import RippledWsClientSign from 'rippled-ws-client-sign'
import { codemirror } from 'vue-codemirror'
import VueJsonPretty from 'vue-json-pretty'
import 'codemirror/lib/codemirror.css'
import 'codemirror/theme/monokai.css'
import 'codemirror/mode/javascript/javascript.js'
import 'codemirror/keymap/sublime.js'

export default {
  name: 'Dashboard',
  components: {
    codemirror,
    VueJsonPretty
  },
  data () {
    return {
      ledger: null,
      connectionPool: null,
      view: 'c',
      menu: {
        c: 'Connections',
        h: 'Health Ranking',
        t: 'TxWatch',
        r: 'Raw Commands',
        x: 'Get Transactions',
        s: 'Submit Tx'
      },
      servers: [],
      accounts: [],
      connections: [],
      addHostname: 'rippled.xrptipbot.com',
      newAccount: '',
      txSecretFocussed: false,
      ranking: {
        data: {},
        interval: null
      },
      interestingAccounts: [
        'rhub8VRN55s94qWKDv6jmDy1pUykJzF3wq',
        'rETx8GBiH6fxhTcfHM9fGeyShqxozyD3xe',
        'rH3uSRUJYoJhK4kL9x1mzUhDimKE2n3oT6',
        'rLEsXccBGNR3UPuPu2hUXPjziKC3qKSBun',
        'rENDnFwR3CPvrsPjD9XXeqVoXeVt2CpPWX',
        'rhPGyJfM8bGjfNCiXLL7WRU8VZ5Cw4kHVe',
        'rhKgFxe7Mp38yeJzvwoNLm46RxMdXotTn6',
        'rDsbeomae4FXwgQTJp9Rs64Qg9vDiTCdBv',
        'rPdvC6ccq8hCdPKSPJkPmyZ4Mi1oG2FFkT',
        'rEb8TK3gBgk5auZkwc6sHnwrGVJH8DuaLh',
        'rPVMhWBsfF9iMXYj3aAzJVkPDTFNSyWdKy',
        'rCoinaUERUrXb1aA7dJu8qRcmvPNiKS3d',
        'r9KG7Du7aFmABzMvDnwuvPaEoMu4Eurwok',
        'rQHYSEyxX3GKK3F6sXRvdd2NHhUqaxtC6F'
      ],
      storeTxAmount: 5000,
      showOnlyPayments: true,
      txCount: 0,
      transactions: [],
      accountTransactions: {
        json: `{
  "command": "account_tx",
  "account": "rPEPPER7kfTD9w2To4CQk6UCfuHM9c6GDY",
  "ledger_index_min": -1,
  "ledger_index_max": -1,
  "limit": 5,
  "forward": true
}`,
        data: {
          account: '',
          records: {},
          lastResponse: null,
          error: ''
        }
      },
      sendTransaction: {
        busy: false,
        json: `{
  "TransactionType": "Payment",
  "Account": "rLmCYXJbMd8HgGj843PjWbsGARD8Q5mRSc",
  "Fee": 10,
  "Destination": "rPdvC6ccq8hCdPKSPJkPmyZ4Mi1oG2FFkT",
  "DestinationTag": 495,
  "Amount": 1234
}`,
        response: null,
        secret: 'ss5Vzfhxwaom4K9qyeezCHPSCG5D7',
        extraSecrets: [],
        error: '',
        errorDetails: {},
        signed: {
          id: '',
          blob: ''
        }
      },
      rawCommand: {
        commandId: 0,
        options: {
          // codemirror options
          tabSize: 2,
          mode: 'application/json',
          theme: 'monokai',
          lineNumbers: true,
          line: true,
          keymap: 'sublime'
        },
        response: {
          body: 'Response over here... (Execute command first)',
          state: null,
          data: null
        },
        json: `{
  "command": "server_info"
}`,
        samples: {
          'Server info': `{
  "command": "server_info"
}`,
          'Random': `{
  "command": "random"
}`,
          'Account transactions': `{
  "command": "account_tx",
  "account": "rPEPPER7kfTD9w2To4CQk6UCfuHM9c6GDY",
  "ledger_index_min": -1,
  "ledger_index_max": -1,
  "limit": 2
}`,
          'Ledger info (recent)': `{
  "command": "ledger",
  "ledger_index": "closed",
  "transactions": true
}`,
          'Ledger info (full history)': `{
  "command": "ledger",
  "ledger_index": 32570,
  "transactions": true
}`,
          'Account (wallet) info': `{
  "command": "account_info",
  "account": "rPEPPER7kfTD9w2To4CQk6UCfuHM9c6GDY",
  "ledger_index": "current"
}`
        }
      }
    }
  },
  watch: {
    view (v) {
      if (v === 'h') {
        this.ranking.data = this.connectionPool.getRanking()
        this.ranking.interval = setInterval(() => {
          this.ranking.data = this.connectionPool.getRanking()
        }, 500)
      } else {
        clearInterval(this.ranking.interval)
      }
    }
  },
  computed: {
    filteredTransactions () {
      return this.transactions.filter(tx => {
        if (this.showOnlyPayments && tx.Data.transaction.TransactionType !== 'Payment') {
          return false
        }
        return true
      }).filter((tx, key) => {
        if (key > 20) {
          return false
        }
        return true
      })
    },
    serverCount () {
      return this.connections.filter(c => {
        return c.state.online
      }).length
    },
    offlineServers () {
      let offline = []
      this.servers.forEach((s) => {
        if (this.connections.filter((c) => {
          return c.hostname === s && c.state.online && this.ranking.data.ranking.indexOf(s) > -1
        }).length < 1) {
          offline.push(s)
        }
      })
      return offline
    }
  },
  beforeMount () {
    /**
     * Create the Pool
     */
    this.connectionPool = new RippledWsClientPool({
      // Config (rating weight) over here
    })

    // On ledger closed
    this.connectionPool.on('ledger', (ledger) => {
      this.ledger = ledger
    })

    // On added server
    this.connectionPool.on('added', (endpoint) => {
      this.servers.push(endpoint)
    })

    // On removed server
    this.connectionPool.on('removed', (endpoint) => {
      let serverIndex = this.servers.indexOf(endpoint)
      if (serverIndex > -1) {
        this.servers.splice(serverIndex, 1)
      }
    })

    // On pool server state update
    this.connectionPool.on('hostinfo', (state) => {
      let existing = this.connections.filter((c) => {
        return c.hostname === state.hostname
      })
      if (existing.length === 1) {
        this.connections[this.connections.indexOf(existing[0])] = state
      } else {
        if (this.servers.indexOf(state.hostname) > -1) {
          this.connections.push(state)
        }
      }
    })

    // On Tx on watched account
    this.connectionPool.on('transaction', (transaction) => {
      this.txCount++
      transaction.__increment = this.txCount
      this.transactions.unshift(transaction)
      this.transactions = this.transactions.slice(0, this.storeTxAmount)
    })
  },
  mounted () {
    if (document.location.hash.length === 2) {
      let char = document.location.hash.substring(1, 2).toLowerCase()
      if (Object.keys(this.menu).indexOf(char > -1)) {
        this.view = char
      }
    }

    let startWithServers = [
      'rippled-dev.xrpayments.co',
      's1.ripple.com',
      's2.ripple.com'
    ]
    startWithServers.forEach((s) => {
      this.connectionPool.addServer(s)
    })

    this.interestingAccounts.forEach((s) => {
      this.accounts.push(s)
      this.connectionPool.subscribeAccount(s)
    })
  },
  methods: {
    sendTx () {
      this.sendTransaction.error = ''
      this.sendTransaction.errorDetails = {}
      this.sendTransaction.busy = true
      this.sendTransaction.signed.id = ''
      this.sendTransaction.signed.blob = ''
      this.sendTransaction.response = null
      let Transaction
      try {
        Transaction = JSON.parse(this.sendTransaction.json)
      } catch (e) {
        this.sendTransaction.busy = false
        this.sendTransaction.error = `${e.message}`
      }
      if (Transaction) {
        let Connection = this.connectionPool.getConnection()
        console.log('Use RippledWsClient connection: ', Connection)
        console.log('Send TX @ connection:', Connection.getState().server.uri)
        let secretOrSecrets = this.sendTransaction.secret
        if (this.sendTransaction.extraSecrets.filter(s => {
          return s.secret.trim().match(/^s/)
        }).length > 0) {
          secretOrSecrets = [ secretOrSecrets ]
          this.sendTransaction.extraSecrets.forEach(s => {
            if (s.secret.trim().match(/^s/)) {
              secretOrSecrets.push(s.secret.trim())
            }
          })
        }
        new RippledWsClientSign(Transaction, secretOrSecrets, Connection).then(TransactionSuccess => {
          this.sendTransaction.busy = false
          this.sendTransaction.response = TransactionSuccess
        }).catch((SignSubmitError) => {
          console.log('SignSubmitError:', SignSubmitError)
          this.sendTransaction.busy = false
          this.sendTransaction.error = `${SignSubmitError.details.type} - ${SignSubmitError.details.message}`
          if (typeof SignSubmitError.details !== 'undefined' && typeof SignSubmitError.details.error !== 'undefined') {
            this.sendTransaction.errorDetails = SignSubmitError.details.error
          }
        })
      }
    },
    getTxs (loadMore) {
      this.accountTransactions.data.error = ''
      let promise
      if (typeof loadMore !== 'undefined' && loadMore) {
        promise = this.accountTransactions.data.lastResponse.more()
      } else {
        promise = this.connectionPool.getTransactions('', this.accountTransactions.json)
      }
      promise.then(r => {
        // console.log('getTxs', r)
        this.accountTransactions.data.lastResponse = r
        if (r.account !== this.accountTransactions.data.account || (typeof loadMore !== 'undefined' && !loadMore)) {
          // New dataset
          this.accountTransactions.data.account = r.account
          this.accountTransactions.data.records = {}
        }
        if (r.txCount > 0) {
          r.transactions.forEach(tx => {
            this.accountTransactions.data.records[tx.tx.hash] = tx
          })
        }
      }).catch(e => {
        console.log('getTxs ERROR', e)
        this.accountTransactions.data.error = e.message
      })
    },
    serverStateIcon (server) {
      let textClass = 'muted'
      let connection = this.connections.filter((c) => { return c.hostname === server })
      if (connection.length === 1 && typeof connection[0] === 'object' && typeof connection[0].state !== 'undefined' && typeof connection[0].state === 'object' && connection[0].state !== null && typeof connection[0].state.online !== 'undefined') {
        textClass = connection[0].state.online ? 'success' : 'danger'
      }
      return '<span class="text-' + textClass + '">⬤</span>'
    },
    addServer () {
      this.connectionPool.addServer(this.addHostname)
      this.addHostname = ''
      this.$refs.addserver.focus()
    },
    removeServer (server) {
      this.connectionPool.removeServer(server)
      let existing = this.connections.filter((c) => { return c.hostname === server })
      if (existing.length === 1) {
        this.connections.splice(this.connections.indexOf(existing[0]), 1)
      }
    },
    addAccount () {
      let account = this.newAccount.trim()
      // this.connectionPool.addAccount(this.addHostname)
      let existing = this.accounts.indexOf(account)
      if (existing < 0) {
        this.accounts.push(account)
        this.connectionPool.subscribeAccount(account)
      }
      this.newAccount = ''
      this.$refs.addaccount.focus()
    },
    removeAccount (account) {
      // this.connectionPool.removeServer(server)
      let existing = this.accounts.indexOf(account)
      if (existing > -1) {
        this.accounts.splice(existing, 1)
        this.connectionPool.unsubscribeAccount(account)
      }
    },
    sendRawCommand () {
      let commandId = this.rawCommand.commandId
      this.rawCommand.commandId++

      this.connectionPool.send(this.rawCommand.json, {
        idempotency: commandId,
        serverTimeout: 1500,
        overallTimeout: 10000
      }).then(response => {
        if (response.idempotency === this.rawCommand.commandId - 1) {
          this.rawCommand.response.state = 'response'
          this.rawCommand.response.body = response.response
          this.rawCommand.response.data = response
        }
      }).catch(error => {
        this.rawCommand.response.state = 'error'
        this.rawCommand.response.body = 'Error: ' + error.message
      })
      this.rawCommand.response.state = 'await'
      this.rawCommand.response.body = 'Waiting for results...'
    }
  }
}
</script>

<style lang="scss" scoped>
  #addSecret {
    &:hover { span { opacity: 1; width: 73px; } }
    span {
      display: block; text-align: right; float: right;
      width: 0px; opacity: 0; overflow: hidden;
      -webkit-transition: 0.5s; transition: 0.5s;
    }
  }
  .table-tx {
    font-size: .75em;
    td {
      position: relative;
      &.complete {
        code {
          color: darken(#45BE60, 15%);
          font-weight: bold;
        }
        .seenByPct {
          background-color: lighten(#4DB653, 40%);
        }
      }
    }
    .seenByPct {
      position: absolute;
      transition: width 0.5s;
      height: 100%;
      max-width: 100% !important;
      background-color: #FFFBE7;
      z-index: -1;
      top: 0px;
      left: 0px;
    }
  }

  a {
    cursor: pointer;
    &:hover {
      code {
        text-decoration: underline;
      }
    }
  }

  .margin-bottom-10 { margin-bottom: 10px; }

  li.list-group-item {
    a.rm {
      margin-right: -15px;
    }
  }

  li.nav-item {
    a {
      &:hover {
        text-decoration: underline;
        &:not(.active) {
          border-color: transparent;
        }
      }
    }
  }

  .health-points {
    margin: 0;
    line-height: 1em;
    border-radius: 48px;
    display: inline-block;
    width: 48px;
    line-height: 1.9em;
    font-size: 1.6em;
    height: 48px;
    text-align: center;
  }

  .jsonResponse.success { padding: 0; }

  .txblob { white-space: normal; word-wrap: break-word; border: 1px solid #ccc; border-radius: 4px; padding: 4px 8px; background-color: #f0f0f0; }
</style>

<style lang="scss">
  div.cm-s-monokai.CodeMirror { border-radius: 4px; }
  .vjs__tree .vjs__value__string { color: #35B64C; font-weight: bold; }
  .error .vjs__tree .vjs__value__string { color: #ca0000; font-weight: bold; }
  .vjs__value__string { word-break: break-all; }
</style>
